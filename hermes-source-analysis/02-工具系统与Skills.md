# 工具系统与 Skills

> 核心文件：`tools/registry.py` + `tools/*.py` (40+) + `toolsets.py` + `skills/`

---

## 一、工具注册中心 (`tools/registry.py`)

### 1.1 注册模式

每个工具文件在模块级别调用 `registry.register()`，import 即注册：

```python
# tools/terminal_tool.py (示例)
from tools.registry import registry

registry.register(
    name="terminal",
    toolset="terminal",
    schema={...},           # OpenAI Function Calling JSON Schema
    handler=terminal_tool,  # 实际执行函数
    check_fn=lambda: True,  # 可用性检查
    requires_env=[],        # 依赖的环境
    is_async=False,         # 是否异步
    description="执行终端命令",
    emoji="💻",
)
```

### 1.2 ToolEntry 元数据

```python
class ToolEntry:
    name: str               # 工具名
    toolset: str            # 所属工具集
    schema: dict            # OpenAI JSON Schema
    handler: Callable       # 执行函数
    check_fn: Callable      # 可用性检查
    requires_env: list      # 环境依赖
    is_async: bool          # 异步标记
    description: str        # 描述
    emoji: str              # 图标
    max_result_size_chars: int  # 结果大小限制
```

### 1.3 导入链（防循环依赖）

```
tools/registry.py  (无外部导入)
       ↑
tools/*.py  (import registry，模块级注册)
       ↑
model_tools.py  (import tools.registry + 所有工具模块)
       ↑
run_agent.py, cli.py, batch_runner.py
```

---

## 二、工具集系统 (`toolsets.py`)

工具集（Toolset）是工具的逻辑分组，控制启用/禁用粒度：

```python
TOOLSETS = {
    "terminal": {"tools": ["terminal", "upload_file"]},
    "browser": {"tools": ["browser", "browser_simple"]},
    "web": {"tools": ["web_search", "web_fetch", ...]},
    "memory": {"tools": ["memory", "memory_search"]},
    "skills": {"tools": ["skills", "skill_manager"]},
    "mcp": {"tools": ["mcp_tool"]},
    "delegate": {"tools": ["delegate_task"]},
    "cron": {"tools": ["cron_create", "cron_list", ...]},
    "vision": {"tools": ["vision_analyze"]},
    "tts": {"tools": ["tts"]},
    ...
}
```

配置中可按工具集粒度启用/禁用：`hermes tools` 交互式管理。

---

## 三、核心工具详解

### 3.1 Terminal Tool (`tools/terminal_tool.py`)

最复杂的工具，支持 **6 种执行后端**：

| 后端 | 环境变量 | 特点 |
|------|---------|------|
| `local` | 默认 | 直接本机执行 |
| `docker` | `TERMINAL_ENV=docker` | 容器隔离 |
| `modal` | `TERMINAL_ENV=modal` | Serverless 云沙箱 |
| `ssh` | `TERMINAL_ENV=ssh` | 远程 SSH |
| `daytona` | `TERMINAL_ENV=daytona` | Daytona 开发环境 |
| `singularity` | `TERMINAL_ENV=singularity` | HPC 环境 |

关键能力：
- 前台/后台执行
- VM/容器生命周期管理
- 持久化文件系统（跨沙箱保留）
- 空闲自动清理
- 磁盘用量警告

### 3.2 Delegate Tool (`tools/delegate_tool.py`)

子 Agent 架构，核心设计：

```python
# 安全限制
DELEGATE_BLOCKED_TOOLS = {
    "delegate_task",   # 禁止递归委托
    "clarify",         # 禁止用户交互
    "memory",          # 禁止写入共享记忆
    "send_message",    # 禁止跨平台副作用
    "execute_code",    # 子Agent应逐步推理
}

MAX_DEPTH = 2  # parent(0) → child(1) → grandchild被拒绝
```

特性：
- 隔离上下文（子Agent不继承父历史）
- 独立终端会话
- 受限工具集
- ThreadPoolExecutor 并行执行
- 最大并发子Agent数可配置（默认3）

### 3.3 Browser Tool (`tools/browser_tool.py`)

基于 **CamoFox**（定制化 Firefox）的浏览器自动化：
- 反指纹检测
- 网站策略检查（`website_policy.py`）
- URL 安全检查（`url_safety.py`）

### 3.4 MCP Tool (`tools/mcp_tool.py`)

MCP（Model Context Protocol）协议桥：
- 连接任意 MCP Server
- 支持 stdio/SSE 传输
- OAuth 认证（`mcp_oauth.py`）
- 工具发现与调用

### 3.5 其他重要工具

| 工具 | 文件 | 功能 |
|------|------|------|
| `memory` | `memory_tool.py` | 读写 MEMORY.md / USER.md |
| `session_search` | `session_search_tool.py` | FTS5 全文搜索历史会话 |
| `skills` | `skills_tool.py` | 技能发现与执行 |
| `skill_manager` | `skill_manager_tool.py` | 技能创建/编辑/删除 |
| `cron_*` | `cronjob_tools.py` | 定时任务 CRUD |
| `web_search/fetch` | `web_tools.py` | 网页搜索与抓取 |
| `vision_analyze` | `vision_tools.py` | 图像分析 |
| `tts` | `tts_tool.py` | 语音合成 |
| `send_message` | `send_message_tool.py` | 跨平台消息发送 |
| `clarify` | `clarify_tool.py` | 向用户澄清问题 |
| `checkpoint` | `checkpoint_manager.py` | 会话检查点 |

---

## 四、Skills 系统

### 4.1 Skill 是什么

Skill 是 Agent 的 **程序性记忆**（Procedural Memory）——捕获"如何做特定类型任务"的经验。

与声明式记忆（MEMORY.md）的区别：
- **MEMORY.md**：宽泛的、声明式的知识
- **Skills**：具体的、可操作的流程

### 4.2 Skill 目录结构

```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md            # 主文件：描述 + 触发条件 + 指令
│   ├── references/         # 参考文档
│   ├── templates/          # 模板文件
│   ├── scripts/            # 辅助脚本
│   └── assets/             # 资源文件
```

### 4.3 SKILL.md 格式

```markdown
---
description: "当用户要求X时使用此技能"
triggers:
  - "关键词1"
  - "关键词2"
platforms: [cli, telegram, discord]
---

# 技能名称

## 步骤
1. ...
2. ...
```

Frontmatter 定义触发条件和平台适配。

### 4.4 Skill 生命周期

```
1. 创建：Agent 从复杂任务经验中自动创建（skill_manager_tool）
   或用户手动编写

2. 发现：prompt_builder 扫描所有 SKILL.md，构建索引
   → 匹配用户消息 → 注入系统提示

3. 执行：作为系统提示的一部分指导 Agent 行为

4. 改进：Agent 使用中可通过 skill_manager 编辑优化

5. 分享：通过 Skills Hub 发布
```

### 4.5 内置技能分类

| 分类 | 示例 |
|------|------|
| `software-development` | 代码开发辅助 |
| `devops` | Docker/K8s 运维 |
| `data-science` | 数据分析 |
| `research` | 研究辅助 |
| `mlops` | ML 运维 |
| `productivity` | 生产力工具 |
| `email` | 邮件处理 |
| `mcp` | MCP 集成 |
| `red-teaming` | 安全测试 |
| `gaming` | 游戏辅助 |
| `smart-home` | 智能家居 |
| `social-media` | 社交媒体 |
| `creative` | 创意写作 |

### 4.6 Skills Hub (`tools/skills_hub.py`)

社区技能市场，兼容 [agentskills.io](https://agentskills.io) 开放标准：
- 搜索/安装社区技能
- 安全扫描（`skills_guard.py`）— 检测恶意代码
- 依赖检查
- 版本管理

### 4.7 安全扫描 (`tools/skills_guard.py`)

安装前扫描：
- 恶意代码检测（网络外传、文件系统破坏）
- 危险模式匹配
- 可配置白名单/黑名单
