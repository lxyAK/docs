# LangGraph 源码深度分析

> 基于 [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) v1.1.7a1
> 分析日期：2026-04-14

---

## 一、项目概览

| 指标 | 数据 |
|------|------|
| 版本 | 1.1.7a1 |
| 许可证 | MIT |
| Python 要求 | ≥ 3.10 |
| Python 文件总数 | 319 |
| 核心库代码量（langgraph） | ~20,455 行 |
| 最重单文件 | `pregel/main.py` — 3,773 行 |

### 定位

LangGraph 是一个**有状态 Agent 编排框架**，基于 Pregel/BSP 计算模型，将 LLM Agent 组织为有向图，支持：

- **Durable Execution（持久化执行）**：通过 Checkpoint 机制实现容错
- **Human-in-the-loop**：中断/恢复原语
- **记忆管理**：短期工作记忆 + 长期持久化记忆
- **生产部署**：内置可扩展基础设施

---

## 二、Monorepo 结构

```
langgraph/
├── libs/
│   ├── langgraph/              # 核心框架（图编排引擎）
│   ├── checkpoint/             # Checkpoint 抽象接口 + Store + Cache
│   ├── checkpoint-postgres/    # PostgreSQL 持久化实现
│   ├── checkpoint-sqlite/      # SQLite 持久化实现
│   ├── checkpoint-conformance/ # Checkpoint 合规性测试套件
│   ├── prebuilt/               # 高级 API（create_agent、工具集成）
│   ├── cli/                    # 命令行工具（langgraph CLI）
│   ├── sdk-py/                 # Python SDK（LangGraph Server API 客户端）
│   └── sdk-js/                 # JS/TS SDK
├── docs/                       # 文档
├── examples/                   # 示例代码
├── CLAUDE.md                   # AI 编码助手指引
└── AGENTS.md                   # 多 Agent 协作说明
```

### 依赖关系

```
checkpoint (基础接口)
├── checkpoint-postgres
├── checkpoint-sqlite
├── prebuilt
└── langgraph

prebuilt
└── langgraph

sdk-py
├── langgraph
└── cli

sdk-js (独立)
```

---

## 三、核心架构

### 3.1 整体架构图

```
┌──────────────────────────────────────────────────────┐
│                    用户 API 层                         │
│  ┌─────────────────┐  ┌──────────────────────────┐   │
│  │  StateGraph      │  │  Functional API          │   │
│  │  (Graph API)     │  │  @entrypoint / @task     │   │
│  └────────┬─────────┘  └────────────┬─────────────┘   │
│           │    compile()            │                  │
│           └──────────┬──────────────┘                  │
│                      ▼                                │
│  ┌──────────────────────────────────────────────────┐ │
│  │              Pregel (执行引擎)                     │ │
│  │  ┌─────────┐  ┌───────────┐  ┌───────────────┐  │ │
│  │  │  Plan   │→│  Execute  │→│   Update      │  │ │
│  │  │ (调度)   │  │ (并行执行) │  │ (Channel更新) │  │ │
│  │  └─────────┘  └───────────┘  └───────────────┘  │ │
│  │         ▲                              │          │ │
│  │         └──────── Loop ◄───────────────┘          │ │
│  └──────────────────────────────────────────────────┘ │
│           │              │              │              │
│  ┌────────▼──────┐ ┌────▼─────┐ ┌─────▼──────┐      │
│  │  Channels     │ │Checkpoint│ │   Store    │      │
│  │  (状态通道)    │ │(持久化)   │ │ (KV存储)   │      │
│  └───────────────┘ └──────────┘ └────────────┘      │
└──────────────────────────────────────────────────────┘
```

### 3.2 三层 API 设计

LangGraph 提供三个抽象层次：

| 层次 | API | 适用场景 |
|------|-----|----------|
| **底层** | `Pregel` + `NodeBuilder` | 高度定制化，直接操作 Channel |
| **中层** | `StateGraph`（Graph API） | 最常用，基于状态的有向图 |
| **高层** | `@entrypoint` + `@task`（Functional API） | 函数式风格，简洁直观 |

---

## 四、核心模块详解

### 4.1 Pregel 执行引擎

**文件**：`libs/langgraph/langgraph/pregel/main.py`（3,773 行）

Pregel 是整个框架的核心，实现了 **BSP（Bulk Synchronous Parallel）计算模型**。

#### 执行周期

每个超步（Superstep）包含三个阶段：

```
┌──────────────────────────────────────────────┐
│  Step N                                       │
│                                               │
│  1. PLAN    → 确定本步需要执行的 Actor 集合     │
│     - 首步：选择订阅 input channels 的 Actor   │
│     - 后续：选择订阅了上步更新 channel 的 Actor │
│                                               │
│  2. EXECUTE → 并行执行所有选中的 Actor         │
│     - 同步/异步双模式                          │
│     - 写入在当前步内对其他 Actor 不可见         │
│     - 超时/失败/中断处理                       │
│                                               │
│  3. UPDATE   → 将 Actor 写入应用到 Channel     │
│     - 按 Channel 聚合所有写入                  │
│     - 更新 Channel 版本号                      │
│     - 触发 Checkpoint 持久化                   │
│                                               │
└──────────────────────────────────────────────┘
         ▲                    │
         └──── Repeat ◄──────┘
         直到无 Actor 被选中 或 达到最大步数
```

#### 关键类

**`Pregel`** — 主类，实现 `Runnable` 接口

```python
class Pregel(PregelProtocol, Generic[StateT, ContextT, InputT, OutputT]):
    nodes: dict[str, PregelNode]          # 图中的所有节点
    channels: dict[str, BaseChannel]      # 所有通道
    input_channels: str | Sequence[str]   # 输入通道
    output_channels: str | Sequence[str]  # 输出通道
    checkpointer: Checkpointer            # 持久化器
    store: BaseStore | None               # KV 存储
    cache: BaseCache | None               # 结果缓存
    stream_mode: StreamMode = "values"    # 流式输出模式
    step_timeout: float | None            # 单步超时
```

**`NodeBuilder`** — 节点构建器（Builder 模式）

```python
node = (
    NodeBuilder()
    .subscribe_only("input")      # 订阅通道
    .do(lambda x: process(x))     # 绑定处理函数
    .write_to("output")           # 写入通道
    .meta(tags=["llm"])           # 添加元数据
    .add_retry_policies(policy)   # 重试策略
    .build()                      # 构建为 PregelNode
)
```

#### `PregelLoop`（执行循环）

**文件**：`libs/langgraph/langgraph/pregel/_loop.py`

管理单次 `invoke`/`stream` 的完整执行周期：

- 初始化 Channel 状态（从 Checkpoint 恢复或全新创建）
- 循环执行 Plan → Execute → Update
- 管理 Checkpoint 持久化时机（sync/async/exit 三种 durability 模式）
- 处理中断（interrupt）和恢复（resume）

提供同步版本 `SyncPregelLoop` 和异步版本 `AsyncPregelLoop`。

#### `PregelRunner`（任务执行器）

**文件**：`libs/langgraph/langgraph/pregel/_runner.py`

负责并发执行一组 `PregelExecutableTask`：

```python
class PregelRunner:
    def tick(self, tasks, *, reraise, timeout, retry_policy, ...):
        """执行一批任务，提交写入，中断处理"""
```

- 使用线程池（sync）或 asyncio（async）并行执行
- 支持 `FuturesDict` 追踪所有 Future 的完成状态
- 异常传播：任一任务失败时中断其他任务
- 简化 traceback：过滤框架内部栈帧，仅展示用户代码

#### `_algo.py`（核心算法）

**文件**：`libs/langgraph/langgraph/pregel/_algo.py`

包含关键调度算法：

| 函数 | 作用 |
|------|------|
| `prepare_next_tasks()` | 根据上步 Channel 更新，确定下一步要执行的节点 |
| `should_interrupt()` | 判断是否需要中断（基于 `interrupt_before/after` 配置） |
| `apply_writes()` | 将所有节点的写入应用到 Channel |
| `local_read()` | 读取当前状态（含本节点自己的未提交写入） |

**调度算法核心逻辑**：

```python
# prepare_next_tasks 简化逻辑
for node_name, node in nodes.items():
    for trigger in node.triggers:
        if channel_versions[trigger] > versions_seen[node_name][trigger]:
            # 该 channel 有新更新 → 触发此节点
            tasks.append(create_task(node_name, node))
            break
```

### 4.2 StateGraph（状态图）

**文件**：`libs/langgraph/langgraph/graph/state.py`（1,752 行）

StateGraph 是最常用的 API，基于 TypedDict 定义状态，通过有向图编排节点。

#### 状态定义

```python
from typing_extensions import Annotated, TypedDict

def reducer(a: list, b: int | None) -> list:
    if b is not None:
        return a + [b]
    return a

class State(TypedDict):
    messages: Annotated[list, reducer]  # 带 reducer 的状态字段
    count: int                           # 覆盖写（默认行为）
```

**Reducer 机制**：

- 无注解的字段默认使用 `LastValue` Channel（覆盖写）
- 使用 `Annotated[type, reducer]` 注解的字段使用 `BinaryOperatorAggregate` Channel
- Reducer 签名：`(current_value, update_value) -> new_value`

#### 编译流程

```python
graph = StateGraph(State)
graph.add_node("think", think_fn)
graph.add_node("act", act_fn)
graph.add_edge(START, "think")
graph.add_edge("think", "act")
graph.add_edge("act", END)

compiled = graph.compile(checkpointer=InMemorySaver())
# compiled 是 CompiledStateGraph，继承自 Pregel
```

`compile()` 将 StateGraph 转换为 `CompiledStateGraph`（→ Pregel 实例）：

1. 为每个 State 字段创建对应的 Channel
2. 为每个节点创建 `PregelNode`，配置订阅/写入
3. 根据边信息设置触发条件
4. 配置 Checkpointer、Store、Cache
5. 验证图结构（无悬空边、可达性检查）

#### 边类型

| 边类型 | API | 说明 |
|--------|-----|------|
| 普通边 | `add_edge(A, B)` | A 执行完必定到 B |
| 条件边 | `add_conditional_edges(A, fn)` | 根据 fn 返回值路由 |
| 等待边 | `add_edge((A, B), C)` | A 和 B 都完成后才执行 C |

#### `Command` 路由

StateGraph 还支持 `Command` 对象进行动态路由，无需预先定义边：

```python
def router(state) -> Command:
    if state["needs_review"]:
        return Command(goto="review", update={"status": "pending"})
    return Command(goto=END)

graph.add_node("router", router, destinations=("review", END))
```

### 4.3 Functional API（函数式 API）

**文件**：`libs/langgraph/langgraph/func/__init__.py`（575 行）

提供 `@entrypoint` 和 `@task` 装饰器，以函数式风格定义工作流。

```python
from langgraph.func import entrypoint, task
from langgraph.checkpoint.memory import InMemorySaver

@task
def compose(topic: str) -> str:
    return f"Essay about {topic}"

@entrypoint(checkpointer=InMemorySaver())
def workflow(topic: str) -> dict:
    essay = compose(topic).result()          # task 返回 Future
    feedback = interrupt("Review this: " + essay)  # 中断等待人工
    return {"essay": essay, "feedback": feedback}
```

**设计特点**：

- `@task` 装饰的函数返回 `SyncAsyncFuture`，支持并行调用
- `@entrypoint` 定义工作流入口，支持 `previous` 参数访问上次返回值
- `entrypoint.final` 可分离「返回值」和「持久化值」
- 底层同样编译为 Pregel 实例

### 4.4 Channel 体系（状态通道）

**目录**：`libs/langgraph/langgraph/channels/`

Channel 是 LangGraph 状态管理的核心抽象。每个 Channel 有三个维度：

```
Channel<Type>
├── ValueType   → 存储的值类型
├── UpdateType  → 接收的更新类型
└── update()    → 聚合函数：(current, updates[]) → new_value
```

#### Channel 类型一览

| Channel | 文件 | 行为 | 典型用途 |
|---------|------|------|----------|
| `LastValue` | `last_value.py` (151行) | 覆盖写，只保留最后一个值 | 默认状态字段、输入/输出 |
| `LastValueAfterFinish` | `last_value.py` | 同上，但仅在超步结束时可见 | 延迟广播 |
| `BinaryOperatorAggregate` | `binop.py` (134行) | 二元运算符归约（如 `operator.add`） | 计数器、列表追加 |
| `EphemeralValue` | `ephemeral_value.py` (79行) | 不跨步持久化，每步重置 | 临时数据、输入通道 |
| `Topic` | `topic.py` (94行) | 发布/订阅，可累积、可去重 | 消息队列、并行聚合 |
| `NamedBarrierValue` | `named_barrier_value.py` (167行) | 屏障同步，等待所有命名写入 | 等待边（多个节点完成后再触发） |
| `NamedBarrierValueAfterFinish` | `named_barrier_value.py` | 同上，延迟版 | — |
| `AnyValue` | `any_value.py` (72行) | 接受任意类型，不校验 | 灵活状态 |
| `UntrackedValue` | `untracked_value.py` (73行) | 不触发版本追踪 | 辅助数据 |

#### BaseChannel 接口

```python
class BaseChannel(ABC, Generic[Value, Update, Checkpoint]):
    def get(self) -> Value                    # 读取当前值
    def update(self, values: Sequence[Update]) -> bool  # 批量更新
    def checkpoint(self) -> Checkpoint        # 序列化状态
    def from_checkpoint(self, cp) -> Self     # 从快照恢复
    def consume(self) -> bool                 # 消费通知（防止重复触发）
    def finish(self) -> bool                  # 结束通知
```

#### Channel 在 Pregel 中的生命周期

```
Step 0: 初始化 → channels = {key: channel.from_checkpoint(cp) or channel()}
Step N:
  1. PLAN    → 检查 channel 版本变化，确定触发哪些节点
  2. EXECUTE → 节点执行，写入暂存到 scratchpad（不直接更新 channel）
  3. UPDATE  → apply_writes() 批量更新所有 channel
              → 更新 channel_versions
              → 持久化 checkpoint
```

### 4.5 Checkpoint 系统（持久化）

**目录**：`libs/checkpoint/`

#### Checkpoint 数据结构

```python
class Checkpoint(TypedDict):
    v: int                              # 格式版本（当前为 1）
    id: str                             # 单调递增的唯一 ID
    ts: str                             # ISO 8601 时间戳
    channel_values: dict[str, Any]      # 各 Channel 的当前值
    channel_versions: ChannelVersions   # 各 Channel 的版本号
    versions_seen: dict[str, ChannelVersions]  # 各节点已看到的版本
    updated_channels: list[str] | None  # 本步更新的 channel 列表
```

#### CheckpointMetadata

```python
class CheckpointMetadata(TypedDict):
    source: Literal["input", "loop", "update", "fork"]
    step: int           # -1=初始输入, 0=第一步, ...
    parents: dict       # 父级 checkpoint 映射
    run_id: str         # 创建此 checkpoint 的 run ID
```

#### BaseCheckpointSaver 接口

```python
class BaseCheckpointSaver(Generic[V]):
    def get_tuple(self, config) -> CheckpointTuple | None
    def list(self, config, *, before, limit) -> Iterator[CheckpointTuple]
    def put(self, config, checkpoint, metadata, new_versions) -> RunnableConfig
    def put_writes(self, config, writes, task_id) -> None
```

**实现类**：

| 实现 | 位置 | 适用场景 |
|------|------|----------|
| `InMemorySaver` | `checkpoint/memory/` | 开发测试 |
| `AsyncPostgresSaver` | `checkpoint-postgres/` | 生产环境 |
| `SqliteSaver` | `checkpoint-sqlite/` | 轻量级持久化 |

#### 持久化时机（Durability）

```python
Durability = Literal["sync", "async", "exit"]
```

| 模式 | 行为 | 性能 | 安全性 |
|------|------|------|--------|
| `sync` | 每步同步写入后再执行下一步 | 慢 | 最高 |
| `async` | 异步写入，与下一步并行 | 快 | 中等 |
| `exit` | 仅在图退出时写入 | 最快 | 最低 |

### 4.6 中断与恢复（Human-in-the-loop）

**文件**：`libs/langgraph/langgraph/types.py` 中的 `Interrupt`、`Command`

```python
from langgraph.types import interrupt, Command

def human_review(state):
    # 暂停执行，等待人工输入
    feedback = interrupt("Please review the output")
    return {"feedback": feedback}

# 恢复执行
graph.invoke(Command(resume="Looks good!"), config={"configurable": {"thread_id": "t1"}})
```

**工作原理**：

1. 节点调用 `interrupt()` → 抛出 `GraphInterrupt` 异常
2. PregelLoop 捕获异常，将当前状态（含 interrupt 信息）写入 Checkpoint
3. 返回 `StateSnapshot`（包含 interrupt 信息）给调用方
4. 用户通过 `Command(resume=value)` 恢复
5. PregelLoop 从 Checkpoint 恢复，将 resume 值注入到中断点

**`Command` 对象**：

```python
class Command:
    update: dict | None          # 状态更新
    goto: str | list[str]        # 跳转到指定节点
    resume: Any                  # 中断恢复值
    graph: str | Command | None  # 嵌套图命令
```

### 4.7 Store 与 Cache

**Store**（键值存储）：

- 通用 KV 存储，支持语义搜索（通过 `index` 配置）
- 跨 thread 共享的长期记忆
- 内置实现：`InMemoryStore`

**Cache**（结果缓存）：

- 缓存节点执行结果，避免重复计算
- 内置实现：`MemoryCache`、`RedisCache`
- 通过 `CachePolicy` 配置缓存键和 TTL

---

## 五、流式输出

**文件**：`libs/langgraph/langgraph/types.py`

```python
StreamMode = Literal[
    "values",       # 每步后的完整状态
    "updates",      # 仅节点返回的增量更新
    "messages",     # LLM 逐 token 流式
    "custom",       # 用户自定义流式数据（通过 StreamWriter）
    "checkpoints",  # Checkpoint 创建事件
    "tasks",        # 任务开始/完成事件
    "debug",        # checkpoints + tasks（调试用）
]
```

**使用方式**：

```python
# 流式输出 LLM token
for chunk in graph.stream(input, config, stream_mode="messages"):
    print(chunk, end="", flush=True)

# 同时使用多种模式
for mode, chunk in graph.stream(input, config, stream_mode=["values", "custom"]):
    if mode == "values":
        handle_state(chunk)
    elif mode == "custom":
        handle_custom(chunk)
```

---

## 六、关键设计模式

### 6.1 Pregel/BSP 模型

LangGraph 将 LLM Agent 编排抽象为图计算问题：

- **节点 = Actor**（独立计算单元）
- **Channel = 通信介质**（状态聚合）
- **超步 = 调度周期**（Plan → Execute → Update）
- **边 = 触发条件**（Channel 版本变化驱动）

这带来了：
- **天然并行性**：无依赖的节点在同一超步内并行执行
- **确定性**：同样的输入 + Checkpoint 产生同样的输出
- **可恢复性**：任何超步都可从 Checkpoint 恢复

### 6.2 Builder 模式

`NodeBuilder` 和 `StateGraph` 都使用 Builder 模式，支持链式调用：

```python
# NodeBuilder
node = NodeBuilder().subscribe_to("input").do(fn).write_to("output").build()

# StateGraph
graph.add_node("a", fn_a).add_node("b", fn_b).add_edge("a", "b")
```

### 6.3 Runnable 协议

Pregel 实现了 LangChain 的 `Runnable` 协议，统一了调用接口：

```python
# 同步
result = graph.invoke({"input": "hello"}, config)

# 异步
result = await graph.ainvoke({"input": "hello"}, config)

# 流式
for chunk in graph.stream({"input": "hello"}, config):
    process(chunk)

# 批量
results = graph.batch([input1, input2], config)
```

### 6.4 Scratchpad 模式

节点执行期间的写入先存入 `PregelScratchpad`（暂存区），超步结束后才批量应用到 Channel。这保证了：

- 同一超步内的节点不会看到彼此的未提交写入
- 节点可以通过 `local_read()` 读取自己的写入（用于条件边）

---

## 七、依赖分析

### 核心依赖

| 依赖 | 版本要求 | 作用 |
|------|----------|------|
| `langchain-core` | ≥ 0.1 | Runnable 协议、Callback 体系 |
| `langgraph-checkpoint` | ≥ 2.1.0, < 5.0.0 | Checkpoint 抽象接口 |
| `langgraph-sdk` | ≥ 0.3.0, < 0.4.0 | Server API 客户端 |
| `langgraph-prebuilt` | ≥ 1.0.9, < 1.1.0 | 高级 API |
| `pydantic` | ≥ 2.7.4 | 数据校验、模型创建 |
| `xxhash` | ≥ 3.5.0 | 高性能哈希（Channel 版本、Checkpoint ID） |

### 为什么强依赖 LangChain

Pregel 继承自 `langchain_core.runnables.Runnable`，整个 Callback、Tracer、配置传递体系都基于 LangChain Core。这意味着：

- ✅ 无缝集成 LangChain 生态（LLM、Tool、Retriever 等）
- ❌ 无法脱离 LangChain 独立使用
- ❌ 引入了不必要的抽象层（对简单场景而言）

---

## 八、代码质量评估

### 优点

1. **架构清晰**：Pregel/BSP 模型简洁优雅，Actor + Channel 分离关注点
2. **类型完善**：大量使用 Generic、TypedDict、overload，类型安全性高
3. **文档丰富**：每个公共 API 都有详细 docstring 和示例
4. **双运行时**：同步/异步完全对称实现
5. **可测试性**：`checkpoint-conformance` 套件确保各实现的正确性

### 不足

1. **`pregel/main.py` 过于庞大**（3,773 行）：混合了配置、验证、调试、IO 映射等多种职责，应拆分为更小的模块
2. **`_internal` 包的循环引用风险**：大量内部常量和工具函数散布在 `_internal/_constants.py` 等文件中
3. **过度工程化**：对简单 Agent 场景，BSP 模型引入了不必要的复杂度
4. **序列化**：Checkpoint 序列化依赖 `JsonPlusSerializer`（msgpack + 自定义扩展），调试困难
5. **错误信息**：部分验证错误信息不够友好（如 Channel 类型不匹配时的报错）

### 复杂度热点

| 文件 | 行数 | 职责 |
|------|------|------|
| `pregel/main.py` | 3,773 | Pregel 核心类 + NodeBuilder |
| `graph/state.py` | 1,752 | StateGraph 编译逻辑 |
| `types.py` | 872 | 类型定义（Command、Interrupt、Stream 等） |
| `pregel/_loop.py` | ~800 | 执行循环 |
| `pregel/_algo.py` | ~700 | 调度算法 |

---

## 九、使用建议

### 适用场景

- 需要**多轮对话 + 状态持久化**的 Agent
- 需要**人工审批**的工作流（Human-in-the-loop）
- 需要**长时间运行**且**可恢复**的 Agent
- 需要**并行执行**多个子任务的复杂工作流

### 不适用场景

- 简单的单次 LLM 调用（直接用 LangChain 即可）
- 不需要状态管理的无状态服务
- 对延迟极度敏感的实时系统（BSP 模型有同步开销）

### 学习路径

1. **入门**：StateGraph + `add_node`/`add_edge` → `compile()` → `invoke()`
2. **进阶**：自定义 Reducer + Conditional Edges + Checkpoint
3. **高级**：Human-in-the-loop（`interrupt`/`Command`）+ Subgraph + Streaming
4. **专家**：直接使用 Pregel + NodeBuilder + 自定义 Channel

---

## 十、与同类框架对比

| 特性 | LangGraph | CrewAI | AutoGen | Dify |
|------|-----------|--------|---------|------|
| 编排模型 | 图（Pregel/BSP） | 角色 + 任务 | 多 Agent 对话 | 可视化流程 |
| 状态管理 | Channel + Checkpoint | 共享上下文 | 对话历史 | 变量系统 |
| 持久化 | ✅ 内置 Checkpoint | ❌ | ❌ | ✅ 数据库 |
| Human-in-the-loop | ✅ 原生支持 | ❌ | ✅ 基本支持 | ✅ |
| 可视化 | ✅ Mermaid/图片 | ❌ | ❌ | ✅ 原生 |
| 自定义程度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| 学习曲线 | 陡峭 | 平缓 | 中等 | 平缓 |

---

*分析完成于 2026-04-14 · 教授 🧑‍💻*
