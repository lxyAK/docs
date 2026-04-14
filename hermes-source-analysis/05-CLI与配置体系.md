# CLI 与配置体系

> 核心文件：`cli.py` (10,013行) + `hermes_cli/` (47个文件) + `hermes_constants.py`

---

## 一、CLI 入口 (`cli.py`)

### 1.1 双入口设计

```
hermes                  # 启动 TUI 交互模式
hermes <subcommand>     # 子命令模式
```

### 1.2 TUI 特性

- **多行编辑**：原生文本编辑区域
- **斜杠命令自动补全**：`/` 触发补全菜单
- **对话历史**：上下键浏览
- **中断重定向**：Ctrl+C 中断当前工作，输入新消息
- **流式工具输出**：实时显示工具执行进度
- **KawaiiSpinner**：可爱的等待动画

### 1.3 子命令路由

`cli.py` 解析参数后路由到 `hermes_cli/` 下对应模块：

| 子命令 | 模块 | 功能 |
|--------|------|------|
| `hermes` | `cli.py` (TUI) | 启动交互对话 |
| `hermes model` | `models.py` (1,933行) | 模型/Provider 选择 |
| `hermes tools` | `tools_config.py` | 工具启用/禁用 |
| `hermes config` | `config.py` (3,306行) | 配置管理 |
| `hermes gateway` | `gateway.py` | 网关管理 |
| `hermes setup` | `setup.py` (3,138行) | 初始化向导 |
| `hermes doctor` | `doctor.py` | 诊断问题 |
| `hermes skills` | `skills_config.py` | 技能管理 |
| `hermes plugins` | `plugins_cmd.py` | 插件管理 |
| `hermes cron` | `cron.py` | 定时任务 |
| `hermes auth` | `auth.py` (3,270行) | 认证管理 |
| `hermes update` | — | 自更新 |
| `hermes claw migrate` | `claw.py` | OpenClaw 迁移 |

此外 `hermes_cli/` 还包含大量辅助模块（共 47 个 .py 文件）：
`auth_commands.py`、`backup.py`、`banner.py`、`callbacks.py`、`clipboard.py`、
`codex_models.py`、`colors.py`、`commands.py`、`copilot_auth.py`、`curses_ui.py`、
`debug.py`、`default_soul.py`、`dump.py`、`env_loader.py`、`logs.py`、`main.py`、
`mcp_config.py`、`memory_setup.py`、`model_normalize.py`、`model_switch.py`、
`nous_subscription.py`、`pairing.py`、`platforms.py`、`profiles.py`、`providers.py`、
`runtime_provider.py`、`skin_engine.py`、`status.py`、`tips.py`、`uninstall.py`、
`web_server.py`、`webhook.py` 等。

---

## 二、hermes_cli 核心模块

### 2.1 配置管理 (`hermes_cli/config.py`, 3,306行)

```yaml
# ~/.hermes/config.yaml 结构
model:
  provider: openrouter
  name: anthropic/claude-sonnet-4
  
gateway:
  telegram:
    enabled: true
    token: ...
  discord:
    enabled: false
    
terminal:
  env: local  # local|docker|modal|ssh|daytona
  
memory:
  provider: builtin  # builtin|mem0|holographic|...
  
delegation:
  max_concurrent_children: 3
  
cron:
  enabled: true
  
context:
  engine: compressor  # compressor|lcm
  threshold_percent: 0.75
```

### 2.2 模型管理 (`hermes_cli/models.py`, 1,933行)

支持的 Provider：

| Provider | Base URL | 特性 |
|----------|----------|------|
| OpenRouter | openrouter.ai | 200+ 模型 |
| Nous Portal | portal.nousresearch.com | Nous 自有 |
| OpenAI | api.openai.com | GPT 系列 |
| Anthropic | api.anthropic.com | Claude 系列 |
| Google | generativelanguage.googleapis.com | Gemini |
| z.ai/GLM | z.ai | GLM 系列 |
| Kimi/Moonshot | platform.moonshot.ai | Moonshot |
| MiniMax | minimax.io | MiniMax |
| Hugging Face | huggingface.co | 开源模型 |
| 小米 MiMo | xiaomimimo.com | MiMo |
| 本地 (vLLM/SGLang) | localhost | 自托管 |

**模型切换**：`hermes model` 交互式选择，或 `/model provider:model` 即时切换。

### 2.3 认证 (`hermes_cli/auth.py`, 3,270行)

- API Key 存储（加密）
- OAuth 流程（MCP 等）
- Provider 特定认证逻辑

### 2.4 Setup 向导 (`hermes_cli/setup.py`, 3,138行)

首次运行引导：
1. 选择 Provider
2. 输入 API Key
3. 选择默认模型
4. 配置工具
5. 可选：OpenClaw 迁移
6. 生成 config.yaml

---

## 三、配置加载机制

### 3.1 加载优先级

```
1. 命令行参数
2. 环境变量 (HERMES_*)
3. ~/.hermes/config.yaml
4. ~/.hermes/.env
5. 项目根目录 .env (开发模式)
6. 内置默认值
```

### 3.2 环境变量 (`hermes_cli/env_loader.py`)

```python
# 加载顺序
1. ~/.hermes/.env          # 用户级
2. <project_root>/.env     # 开发级（覆盖用户级）
```

### 3.3 常量 (`hermes_constants.py`)

```python
def get_hermes_home():
    """默认 ~/.hermes，可通过 HERMES_HOME 覆盖"""
    
def get_skills_dir():
    """技能目录：~/.hermes/skills/"""

OPENROUTER_BASE_URL = "https://openrouter.ai/api/v1"
```

---

## 四、Smart Model Routing (`agent/smart_model_routing.py`)

### 4.1 双模型策略

根据消息复杂度自动路由到 cheap 或 strong 模型：

```python
_COMPLEX_KEYWORDS = {
    "debug", "implement", "refactor", "traceback",
    "analyze", "architecture", "design", "benchmark",
    "optimize", "review", "terminal", "tool", "pytest",
    "plan", "delegate", "docker", "kubernetes",
}
```

- 简单问候/闲聊 → cheap 模型
- 复杂编码/调试 → strong 模型

### 4.2 配置

```yaml
model:
  routing:
    enabled: true
    cheap: "anthropic/claude-haiku-4"
    strong: "anthropic/claude-sonnet-4"
```

---

## 五、诊断工具 (`hermes_cli/doctor.py`)

自动检测：
- Python 版本
- 依赖完整性
- 配置有效性
- API 连通性
- 磁盘空间
- SSL 证书

命令：`hermes doctor`

---

## 六、Web Server (`hermes_cli/web_server.py`)

内置 Web 服务：
- 会话管理 UI
- 配置编辑
- 技能浏览
- 状态监控

React 前端位于 `web/` 目录。
