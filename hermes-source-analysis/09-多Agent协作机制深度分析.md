# Hermes Agent 多 Agent 协作机制深度分析

> 基于源码 `D:\CSBR\opensource\hermes-agent-main` 的深度分析
> 分析日期：2026-04-14

## 一、总体结论

**Hermes Agent 不是传统意义上的多 Agent 编排框架**（如 CrewAI、AutoGen、LangGraph）。它是一个**单 Agent 架构，通过三种补充机制实现有限的多 Agent 协作能力**：

| 机制 | 类型 | 通信方式 | 并发支持 |
|------|------|----------|----------|
| Delegate Tool | 层级式父子 Agent | 同步阻塞，结果返回 | 批量并行（ThreadPoolExecutor） |
| Mixture-of-Agents (MoA) | 多模型协作投票 | 无，纯 API 调用 | 并行（asyncio.gather） |
| ACP 协议 | Agent-as-a-Service | 外部编排器调度 | 每会话独立 Agent 实例 |

**不支持**的场景：
- Agent 间对等通信（无消息总线、无 Pub/Sub）
- Agent 集群/蜂群（Swarm）拓扑
- 工作流编排（Pipeline/DAG 调度）
- 共享状态/共享对话上下文

---

## 二、Delegate Tool — 层级式父子 Agent 委托

### 2.1 架构设计

`tools/delegate_tool.py` 是 Hermes 的核心多 Agent 机制。它实现了一种**严格的层级委托模式**：父 Agent 生成子 Agent 实例，子 Agent 完成任务后返回摘要结果。

```
父 Agent (AIAgent)
  ├── 调用 delegate_task 工具
  │   ├── 构建 child AIAgent（隔离上下文）
  │   ├── child 在线程中运行
  │   └── 返回 JSON 结果摘要
  └── 继续处理其他任务
```

### 2.2 子 Agent 的隔离机制

每个子 Agent 获得**完全隔离的执行环境**：

| 隔离维度 | 实现方式 | 源码位置 |
|----------|----------|----------|
| 对话上下文 | `skip_context_files=True, skip_memory=True`，无父级历史 | `delegate_tool.py:365-366` |
| 终端会话 | 独立的 `task_id`，拥有自己的 shell 工作目录 | `delegate_tool.py:10-11` |
| 工具集 | 受限的 toolset，剥离了危险工具 | `delegate_tool.py:32-38` |
| 系统提示 | 基于委托目标动态构建的专注 prompt | `delegate_tool.py:90-122` |
| 迭代预算 | 独立的 `max_iterations`（默认 50） | `delegate_tool.py:80` |

### 2.3 被禁止的工具（安全边界）

子 Agent **绝不能**访问以下工具（`DELEGATE_BLOCKED_TOOLS`）：

```
delegate_task  — 禁止递归委托
clarify        — 禁止与用户交互
memory         — 禁止写入共享 MEMORY.md
send_message   — 禁止跨平台消息副作用
execute_code   — 禁止执行代码脚本
```

此外，以下工具集被整体排除（`_EXCLUDED_TOOLSET_NAMES`）：

```
debugging, safe, delegation, moa, rl
```

以及所有 `hermes-*` 前缀的平台组合工具集。

### 2.4 深度限制与递归防护

```python
MAX_DEPTH = 2  # parent(0) → child(1) → grandchild 被拒绝
```

- 父 Agent 深度为 0
- 子 Agent 深度为 1（`child._delegate_depth = parent._delegate_depth + 1`）
- 孙 Agent 会被硬性拒绝：`depth >= MAX_DEPTH` 时返回错误

### 2.5 并发子 Agent 管理

```python
_DEFAULT_MAX_CONCURRENT_CHILDREN = 3  # 默认最多 3 个并发子 Agent
```

配置优先级：`config.yaml delegation.max_concurrent_children` > `DELEGATION_MAX_CONCURRENT_CHILDREN` 环境变量 > 默认值 3。

**单任务模式**：直接在当前线程运行，无线程池开销。

**批量模式**：使用 `ThreadPoolExecutor(max_workers=max_children)` 并行执行：

```python
with ThreadPoolExecutor(max_workers=max_children) as executor:
    futures = {}
    for i, t, child in children:
        future = executor.submit(_run_single_child, ...)
        futures[future] = i
    for future in as_completed(futures):
        entry = future.result()
        results.append(entry)
```

### 2.6 子 Agent 的凭证管理

子 Agent 的模型凭证解析链：

```
delegation.base_url + delegation.api_key  →  直接指定端点
delegation.provider                        →  通过 runtime provider 系统解析
无配置                                    →  继承父 Agent 凭证
```

关键设计：**子 Agent 可运行在完全不同的 provider:model 上**。例如父 Agent 在 Nous Portal 运行，子 Agent 可路由到 OpenRouter 的便宜模型。

凭证池（`CredentialPool`）支持：
- 同 provider：子 Agent 共享父 Agent 的凭证池（同步冷却和轮转）
- 不同 provider：尝试加载该 provider 自己的凭证池
- 无池可用：回退到固定凭证

### 2.7 中断传播

当父 Agent 被用户中断时，通过 `_active_children` 列表向所有子 Agent 传播中断：

```python
# run_agent.py 中的中断传播（约第 2928 行）
# 使用 _active_children_lock 保护并发访问
with self._active_children_lock:
    for child in self._active_children:
        child.interrupt()
```

### 2.8 心跳机制

子 Agent 运行期间，独立的守护线程每 30 秒向父 Agent 发送心跳，防止 Gateway 的不活跃超时触发：

```python
_HEARTBEAT_INTERVAL = 30  # 秒

def _heartbeat_loop():
    while not _heartbeat_stop.wait(_HEARTBEAT_INTERVAL):
        touch = getattr(parent_agent, '_touch_activity', None)
        if touch:
            touch(f"delegate_task: subagent {task_index} working")
```

### 2.9 进度回调中继

子 Agent 的工具调用进度通过回调中继到父 Agent 的显示层：

```
子 Agent 工具调用 → child_progress_cb → 父 Agent spinner (CLI)
                                       → parent_cb "subagent_progress" (Gateway)
```

Gateway 模式下批量处理（每 5 个工具调用批量发送一次，减少消息噪音）。

### 2.10 返回结果结构

```json
{
  "results": [
    {
      "task_index": 0,
      "status": "completed",
      "summary": "子 Agent 的摘要输出",
      "api_calls": 12,
      "duration_seconds": 45.3,
      "model": "anthropic/claude-sonnet-4.6",
      "exit_reason": "completed",
      "tokens": { "input": 15000, "output": 3200 },
      "tool_trace": [
        { "tool": "terminal", "args_bytes": 85, "result_bytes": 1024, "status": "ok" }
      ]
    }
  ],
  "total_duration_seconds": 45.3
}
```

父 Agent **只能看到最终摘要**，永远看不到子 Agent 的中间工具调用和推理过程。

---

## 三、Mixture-of-Agents (MoA) — 多模型协作投票

### 3.1 设计理念

基于论文 *"Mixture-of-Agents Enhances Large Language Model Capabilities"* (Wang et al., arXiv:2406.04692)，MoA 是一种**2 层多模型协作架构**，通过聚合多个前沿模型的输出来提升推理质量。

**注意：这不是多 Agent 协作，而是多模型协作**。没有独立的 Agent 实例，只有对多个 LLM API 的并行调用。

### 3.2 架构图

```
用户问题 (user_prompt)
    │
    ├── Layer 1: Reference Models（并行，4 个模型）
    │   ├── claude-opus-4.6      ──→ response_1
    │   ├── gemini-3-pro-preview  ──→ response_2
    │   ├── gpt-5.4-pro           ──→ response_3
    │   └── deepseek-v3.2         ──→ response_4
    │
    └── Layer 2: Aggregator（综合）
        └── claude-opus-4.6 + 所有 response ──→ 最终输出
```

### 3.3 源码实现细节

**Reference 模型配置**（`mixture_of_agents_tool.py:63-68`）：

```python
REFERENCE_MODELS = [
    "anthropic/claude-opus-4.6",
    "google/gemini-3-pro-preview",
    "openai/gpt-5.4-pro",
    "deepseek/deepseek-v3.2",
]
AGGREGATOR_MODEL = "anthropic/claude-opus-4.6"
```

**温度设置**：
- Reference：`0.6`（平衡创造力，确保多样性）
- Aggregator：`0.4`（聚焦综合，确保一致性）

**推理配置**：所有模型启用 reasoning，effort 设为 `xhigh`（最高推理强度）。

**失败容错**：
- 最少需要 1 个成功响应（`MIN_SUCCESSFUL_REFERENCES = 1`）
- 每个模型最多 6 次重试（`max_retries = 6`）
- 指数退避：2s → 4s → 8s → 16s → 32s → 60s

**聚合提示词**（来自论文）：

```
You have been provided with a set of responses from various open-source models
to the latest user query. Your task is to synthesize these responses into a
single, high-quality response. It is crucial to critically evaluate the
information provided in these responses, recognizing that some of it may be
biased or incorrect.
```

### 3.4 与 Delegate Tool 的关系

MoA 工具被注册在 `moa` toolset 中，而子 Agent 的排除工具集包含 `"moa"`（`delegate_tool.py:43`）。这意味着：

- **父 Agent 可以使用 MoA**
- **子 Agent 不能使用 MoA**
- **Delegate Tool 和 MoA 是互补的**：Delegate 处理工具密集型子任务，MoA 处理推理密集型问题

### 3.5 适用场景

MoA 工具描述明确指出适用场景：
- 复杂数学证明和计算
- 高级编码问题和算法设计
- 多步骤分析推理任务
- 需要多领域专业知识的问题
- 单一模型表现不足的任务

**代价**：5 次 API 调用（4 个 reference + 1 个 aggregator），高延迟、高 token 消耗。

---

## 四、ACP 协议 — Agent-as-a-Service

### 4.1 定位

ACP (Agent Client Protocol) 是 Hermes 作为**被编排方**的集成协议。通过 ACP，外部编排器（IDE 插件、其他 Agent 系统）可以调用 Hermes 的能力。

**ACP ≠ Agent 间通信协议**。它是 Hermes 暴露给外部系统的服务接口。

### 4.2 服务端架构

`acp_adapter/server.py` 中的 `HermesACPAgent` 类实现了 ACP Agent 规范：

```python
class HermesACPAgent(acp.Agent):
    # 核心方法：
    async def initialize(...)      → 握手和能力声明
    async def new_session(...)     → 创建新会话（= 新 AIAgent 实例）
    async def prompt(...)          → 发送消息（= agent.run_conversation）
    async def fork_session(...)    → 深拷贝会话历史到新会话
    async def resume_session(...)  → 恢复已存在会话
    async def list_sessions(...)   → 列出所有会话
```

### 4.3 会话管理

`SessionManager` 维护会话状态：

```python
@dataclass
class SessionState:
    session_id: str
    agent: Any        # 独立的 AIAgent 实例
    cwd: str
    model: str
    history: List[Dict]
    cancel_event: threading.Event
```

关键特性：
- **每个 ACP 会话 = 一个独立的 AIAgent 实例**
- 会话持久化到 SQLite（`~/.hermes/state.db`），跨进程重启恢复
- Fork 深拷贝历史记录到新会话（可并行探索不同方向）
- 线程池（4 worker）处理并发请求

### 4.4 事件转换

Hermes 内部事件被转换为 ACP 标准事件：

| Hermes 内部事件 | ACP 事件 |
|----------------|----------|
| LLM 响应文本 | `MessageEvent` |
| 工具调用 | `StepEvent` |
| 思考/推理 | `ThinkingEvent` |
| 工具进度 | `ToolProgressEvent` |

### 4.5 ACP 与 Delegate Tool 的交集

Delegate Tool 支持 `acp_command` 参数，允许子 Agent 通过 ACP 协议调用外部 Agent：

```python
# delegate_tool.py 的 schema 中
"acp_command": {
    "type": "string",
    "description": "Override ACP command for child agents (e.g. 'claude', 'copilot')."
}
```

这意味着 Hermes 可以**通过 Delegate Tool 委托任务给其他 ACP 兼容 Agent**（如 Claude Code、Copilot），实现跨 Agent 系统的协作。

### 4.6 MCP vs ACP 定位对比

| 维度 | ACP | MCP |
|------|-----|-----|
| 角色 | Hermes 作为**被调用方** | Hermes 作为**调用方** |
| 方向 | 外部系统 → Hermes | Hermes → 外部工具 |
| 用途 | IDE 集成、编排系统 | 工具发现和调用 |
| 协议 | Agent Client Protocol | Model Context Protocol |

---

## 五、Gateway 多平台 Agent

### 5.1 单 Agent 多平台架构

Gateway 不是多 Agent 架构，而是**一个 Agent 实例同时服务多个消息平台**：

```
GatewayRunner
  └── 单个 Agent 进程
      ├── Telegram 适配器 ←→ 用户 A
      ├── Discord 适配器  ←→ 用户 B
      ├── Slack 适配器    ←→ 用户 C
      ├── 飞书适配器      ←→ 用户 D
      └── 企业微信适配器  ←→ 用户 E
```

每个用户会话有独立的 `AIAgent` 实例（缓存在 `_agent_cache` 中），但它们之间**没有通信**。

### 5.2 会话隔离

- 每个平台的每个用户/频道 = 独立的会话
- 会话间无共享状态
- 会话缓存用于 Prompt 缓存优化，不是 Agent 间通信

---

## 六、Batch Runner — 并行批量执行

### 6.1 用途

`batch_runner.py` 用于**RL 训练数据生成**和**大规模评测**，不是运行时多 Agent 协作。

### 6.2 架构

```
BatchRunner
  └── multiprocessing.Pool(num_workers=4)
      ├── Worker 1 → AIAgent(prompt_1)
      ├── Worker 2 → AIAgent(prompt_2)
      ├── Worker 3 → AIAgent(prompt_3)
      └── Worker 4 → AIAgent(prompt_4)
```

每个 Worker 进程创建独立的 `AIAgent` 实例，完全隔离。

### 6.3 关键特性

- **多进程**（非多线程），利用多核 CPU
- **断点续传**：基于 checkpoint 文件和内容匹配恢复
- **轨迹保存**：输出 JSONL 格式的对话轨迹
- **工具使用统计**：聚合所有批次的工具调用成功率
- **推理覆盖统计**：追踪有多少轮次包含推理过程

---

## 七、外部 Agent 集成（Skills）

### 7.1 Blackbox AI CLI

`optional-skills/autonomous-ai-agents/blackbox/SKILL.md` 定义了与 Blackbox AI 的集成。

本质上是**通过终端工具调用外部 Agent CLI**：

```
Hermes Agent
  └── terminal(command="blackbox --prompt 'Add JWT auth'", pty=true)
      └── Blackbox AI CLI（独立进程）
          ├── 多模型调度
          └── Judge 评分机制
```

这不是 Agent 间通信，而是 Hermes 通过 shell 命令驱动的子进程调用。

### 7.2 Honcho 记忆系统

Honcho 提供跨会话用户建模，支持多 Profile 场景下的共享记忆：

```
Profile A (coder) ←── 共享 workspace ──→ Profile B (writer)
      │                                         │
   AI Peer "coder"                          AI Peer "writer"
      │                                         │
      └── 观察用户行为 → 构建用户表示 ←──────────┘
```

每个 Profile 有独立的 AI Peer 身份，但共享同一用户表示。这是**记忆共享**而非 **Agent 间通信**。

---

## 八、与其他多 Agent 框架的对比

### 8.1 与 CrewAI 对比

| 维度 | Hermes Agent | CrewAI |
|------|-------------|--------|
| Agent 角色 | 无显式角色系统 | Role/Goal/Backstory |
| 编排模式 | 层级委托（深度 2） | Sequential/Hierarchical/Process |
| Agent 间通信 | 无 | Agent 间传递上下文 |
| 任务分解 | 由 LLM 自行决定 | 显式 Task 对象 |
| 共享状态 | 无 | 共享 Memory |

### 8.2 与 AutoGen 对比

| 维度 | Hermes Agent | AutoGen |
|------|-------------|---------|
| 通信模式 | 单向（子 → 父返回结果） | 双向对话 |
| Agent 拓扑 | 树形（2 层） | 任意拓扑 |
| 人类参与 | 子 Agent 禁止交互 | 支持 Human-in-the-loop |
| 代码执行 | 子 Agent 禁止 execute_code | 原生支持 |

### 8.3 与 LangGraph 对比

| 维度 | Hermes Agent | LangGraph |
|------|-------------|-----------|
| 工作流定义 | 无（LLM 自行决策） | 显式 DAG/状态机 |
| 状态管理 | 无跨 Agent 状态 | 共享 State 对象 |
| 条件路由 | 无 | 条件边 |
| 持久化 | 会话级 SQLite | Checkpointer |

---

## 九、关键发现与设计权衡

### 9.1 刻意的设计选择

Hermes 的多 Agent 能力有限是**刻意的设计选择**，而非实现不足：

1. **安全优先**：子 Agent 禁止递归委托、禁止用户交互、禁止写入共享记忆——每一项都是为了防止失控的 Agent 链
2. **上下文保护**：子 Agent 的中间结果不进入父 Agent 上下文窗口，防止信息爆炸
3. **简单性**：层级委托模式比复杂的 DAG/工作流更容易理解和调试
4. **实用性**：覆盖了最常见的多 Agent 需求（并行子任务、多模型投票），而非追求通用编排

### 9.2 缺失的能力

如果需要以下场景，Hermes 当前不支持，需要外部编排：

1. **多 Agent 对等讨论**：Agent A 和 Agent B 就同一问题进行多轮辩论
2. **工作流编排**：Agent A → Agent B → Agent C 的确定性流程
3. **共享白板**：多个 Agent 同时读写共享状态
4. **动态组队**：根据任务动态组建 Agent 团队
5. **Agent 角色特化**：为不同 Agent 配置不同的系统提示和人格

### 9.3 ACP 作为扩展点

ACP 协议是 Hermes 与外部多 Agent 系统集成的关键扩展点：

```
外部编排器 (CrewAI/AutoGen/自研)
  ├── Hermes Agent (ACP)
  ├── Claude Code (ACP)
  └── 其他 ACP 兼容 Agent
```

通过 ACP，Hermes 可以成为更大多 Agent 系统中的一个节点，由外部编排器负责 Agent 间的协调和通信。

---

## 十、源码关键路径索引

| 功能 | 文件 | 关键行/类 |
|------|------|-----------|
| Delegate Tool 主入口 | `tools/delegate_tool.py` | `delegate_task()` 第 623 行 |
| 子 Agent 构建 | `tools/delegate_tool.py` | `_build_child_agent()` 第 238 行 |
| 子 Agent 运行 | `tools/delegate_tool.py` | `_run_single_child()` 第 399 行 |
| 进度回调中继 | `tools/delegate_tool.py` | `_build_child_progress_callback()` 第 158 行 |
| 凭证池解析 | `tools/delegate_tool.py` | `_resolve_child_credential_pool()` 第 816 行 |
| MoA 主入口 | `tools/mixture_of_agents_tool.py` | `mixture_of_agents_tool()` 第 233 行 |
| MoA Reference 调用 | `tools/mixture_of_agents_tool.py` | `_run_reference_model_safe()` 第 104 行 |
| MoA Aggregator 调用 | `tools/mixture_of_agents_tool.py` | `_run_aggregator_model()` 第 179 行 |
| ACP Server | `acp_adapter/server.py` | `HermesACPAgent` 第 93 行 |
| ACP Session 管理 | `acp_adapter/session.py` | `SessionManager` 第 70 行 |
| Batch Runner | `batch_runner.py` | `BatchRunner` 第 514 行 |
| 工具集定义 | `toolsets.py` | `TOOLSETS` 第 68 行 |
| 核心 Agent 类 | `run_agent.py` | `AIAgent` 第 526 行 |
| 中断传播 | `run_agent.py` | `_active_children` 约 2928 行 |
