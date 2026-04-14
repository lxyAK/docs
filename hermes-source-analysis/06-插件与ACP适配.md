# 插件与 ACP 适配

> 核心文件：`plugins/` + `acp_adapter/` + `tools/mcp_tool.py`

---

## 一、插件系统

### 1.1 插件目录结构

```
plugins/
├── __init__.py
├── context_engine/        # 上下文引擎插件
└── memory/                # 记忆后端插件
    ├── mem0/              # Mem0 集成
    │   ├── __init__.py
    │   ├── plugin.yaml    # 元数据
    │   └── README.md
    ├── supermemory/       # Supermemory 集成
    ├── holographic/       # 全息记忆（本地向量）
    │   ├── __init__.py
    │   ├── plugin.yaml
    │   ├── holographic.py
    │   ├── store.py
    │   └── retrieval.py
    ├── retaindb/          # RetainDB
    └── byterover/         # Byterover
```

### 1.2 插件发现

放置在 `plugins/<type>/<name>/` 目录即自动发现：
- `context_engine/` → 替换默认压缩引擎
- `memory/` → 替换/增强记忆后端

### 1.3 plugin.yaml

```yaml
name: holographic
type: memory
version: "1.0"
description: "本地全息向量记忆"
dependencies:
  - chromadb>=0.4.0
  - sentence-transformers
```

### 1.4 插件加载流程

```
1. 扫描 plugins/ 目录
2. 读取 plugin.yaml 元数据
3. 配置中指定激活插件
4. import 插件模块
5. 调用 register() 注册
6. 注入到对应管理器
```

---

## 二、ACP 适配器 (`acp_adapter/`)

### 2.1 ACP (Agent Client Protocol)

Hermes 实现了 [ACP 协议](https://github.com/AaronWong1999/agent-client-protocol)，暴露为标准 ACP Agent Server。

### 2.2 模块结构

```
acp_adapter/
├── __init__.py
├── __main__.py       # 入口
├── server.py         # ACP 服务器
├── session.py        # 会话管理
├── auth.py           # 认证
├── permissions.py    # 权限/审批
├── entry.py          # 入口处理
├── events.py         # 事件转换
└── tools.py          # 工具桥接
```

### 2.3 server.py 核心功能

```python
class AcpAgentServer:
    """将 Hermes Agent 暴露为 ACP 服务"""
    
    async def initialize(request) → InitializeResponse
    async def new_session(request) → NewSessionResponse
    async def prompt(request) → PromptResponse
    async def fork_session(request) → ForkSessionResponse
    async def resume_session(request) → ResumeSessionResponse
    async def list_sessions(request) → ListSessionsResponse
```

### 2.4 会话管理 (`acp_adapter/session.py`)

```python
class SessionManager:
    """管理 ACP 会话到 Hermes Agent 实例的映射"""
    
    # 每个 ACP session 对应一个独立 Agent
    sessions: Dict[str, SessionState]
    
class SessionState:
    agent: AIAgent          # 独立 Agent 实例
    session_id: str
    created_at: datetime
    last_active: datetime
```

### 2.5 事件转换 (`acp_adapter/events.py`)

Hermes 内部事件 → ACP 标准事件：

```python
make_message_cb()      # Agent 回复 → ACP MessageEvent
make_step_cb()         # 工具调用 → ACP StepEvent  
make_thinking_cb()     # 推理过程 → ACP ThinkingEvent
make_tool_progress_cb() # 工具进度 → ACP ToolProgressEvent
```

### 2.6 权限与审批 (`acp_adapter/permissions.py`)

```python
def make_approval_callback():
    """将 ACP 审批流对接到 Hermes 的命令审批系统"""
```

---

## 三、MCP 集成 (`tools/mcp_tool.py`)

### 3.1 MCP 协议支持

Model Context Protocol，连接外部 MCP Server 扩展工具能力。

### 3.2 传输方式

| 传输 | 适用场景 |
|------|---------|
| `stdio` | 本地进程通信 |
| `SSE` | 远程 HTTP 服务 |

### 3.3 MCP 配置 (`hermes_cli/mcp_config.py`)

```yaml
# config.yaml
mcp:
  servers:
    - name: filesystem
      transport: stdio
      command: "npx"
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
    - name: github
      transport: sse
      url: "https://mcp.github.com/sse"
      auth:
        type: oauth
```

### 3.4 OAuth 支持 (`tools/mcp_oauth.py`)

MCP Server 的 OAuth 2.0 认证流程。

---

## 四、ACP vs MCP 定位

```
                    ACP                    MCP
                 ┌───────┐            ┌───────┐
 方向            │ Agent │ ←─────→    │ 工具  │
                 │ 协议  │            │ 协议  │
                 └───────┘            └───────┘
 用途      Agent 编排/通信      工具发现/调用
 角色      Hermes 作为服务端    Hermes 作为客户端
 对标      OpenClaw ACP         Anthropic MCP
```

- **ACP**：Hermes 被其他系统调用（如 IDE 插件、编排系统）
- **MCP**：Hermes 调用外部工具服务

---

## 五、插件开发指南

### 5.1 创建记忆插件

```python
# plugins/memory/my-memory/__init__.py
from agent.memory_provider import MemoryProvider

class MyMemoryProvider(MemoryProvider):
    @property
    def name(self):
        return "my-memory"
    
    def get_system_prompt(self):
        return "你有一个外部记忆系统..."
    
    def prefetch(self, user_message):
        # 检索相关记忆
        return retrieved_context
    
    def sync(self, user_msg, assistant_resp):
        # 存储新记忆
        pass
    
    def register(self):
        # 注册到 MemoryManager
        pass
```

### 5.2 创建上下文引擎插件

```python
# plugins/context_engine/my-engine/__init__.py
from agent.context_engine import ContextEngine

class MyContextEngine(ContextEngine):
    @property
    def name(self):
        return "my-engine"
    
    def should_compress(self, prompt_tokens=None):
        # 自定义压缩触发逻辑
        pass
    
    def compress(self, messages):
        # DAG 压缩、RAG 替代等
        pass
```
