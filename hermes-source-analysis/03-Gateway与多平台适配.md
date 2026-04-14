# Gateway 与多平台适配

> 核心文件：`gateway/run.py` (9K行) + `gateway/session.py` + `gateway/platforms/*.py`

---

## 一、Gateway 架构

### 1.1 GatewayRunner

```
GatewayRunner
├── start_gateway()         # 启动所有平台适配器
├── _run_platform()         # 运行单个平台
├── _handle_message()       # 消息处理入口
├── _run_agent_for_message() # 为消息创建Agent实例
├── _deliver_response()     # 投递响应
└── _cleanup()              # 关闭清理
```

### 1.2 启动流程

```python
async def start_gateway():
    # 1. SSL 证书自动检测（NixOS 等非标准系统）
    _ensure_ssl_certs()
    
    # 2. 加载配置 → 确定启用的平台
    platforms = load_enabled_platforms()
    
    # 3. 初始化 Hooks
    hooks = load_hooks()
    
    # 4. 并行启动各平台适配器
    tasks = [platform.start() for platform in platforms]
    await asyncio.gather(*tasks)
```

---

## 二、平台适配器体系

### 2.1 基类 (`gateway/platforms/base.py`)

```python
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def start(self): ...        # 启动连接
    
    @abstractmethod
    async def send_message(self, ...): ...  # 发送消息
    
    @abstractmethod
    async def stop(self): ...         # 停止连接
    
    # 通用功能
    def truncate_message(self, text, limit):  # 消息截断
    def split_message(self, text, limit):     # 长消息拆分
```

**关键细节 — UTF-16 消息长度计算**：

Telegram 的消息限制 4096 是 UTF-16 编码单元数，不是 Unicode 码点数。emoji（如 😀）在 UTF-16 中占 2 个单元。基类实现了 `utf16_len()` 和二分搜索安全截断。

### 2.2 已实现平台

| 平台 | 文件 | 行数 | 特性 |
|------|------|------|------|
| Telegram | `telegram.py` | 2,786 | 语音转写、贴纸缓存、内联按钮 |
| Discord | `discord.py` | 2,963 | Slash 命令、线程、嵌入消息 |
| Slack | `slack.py` | ~1,500 | Socket Mode、Block Kit |
| WhatsApp | `whatsapp.py` | ~1,500 | 桥接模式、语音备忘录 |
| Signal | `signal.py` | ~1,200 | signal-cli 桥接 |
| **飞书** | `feishu.py` | **3,950** | 事件订阅、卡片消息、群聊 |
| **微信** | `weixin.py` | **1,829** | 桥接模式 |
| Matrix | `matrix.py` | 2,005 | Element 生态 |
| Mattermost | `mattermost.py` | ~1,200 | WebSocket |
| HomeAssistant | `homeassistant.py` | ~800 | 智能家居 |
| Email | `email.py` | ~1,000 | SMTP/IMAP |
| API Server | `api_server.py` | 1,904 | HTTP REST API |
| DingTalk | `dingtalk.py` | ~1,200 | 钉钉 |
| WeCom | `wecom.py` | ~1,200 | 企业微信 |
| Webhook | `webhook.py` | ~600 | 通用 Webhook |

### 2.3 飞书适配器 (`feishu.py`) — 最大平台文件

3,950 行，仅次于 `run_agent.py` 和 `gateway/run.py`，说明飞书是重点支持平台：

- 事件订阅（消息、提及、入群）
- 卡片消息（Interactive Card）
- 群聊/单聊适配
- 文件/图片收发
- Bot 身份管理

---

## 三、会话管理 (`gateway/session.py`)

### 3.1 会话生命周期

```
用户消息 → 查找/创建 Session → 加入消息队列 → Agent 处理 → 响应投递
                ↓
        会话超时/过期 → 清理
```

### 3.2 会话上下文 (`gateway/session_context.py`)

每个会话维护独立的：
- 对话历史
- 工作目录
- Agent 实例
- 中断状态

---

## 四、消息路由

### 4.1 入站流程

```
平台消息 → PlatformAdapter.on_message()
    ↓
GatewayRunner._handle_message()
    ↓
Hooks.pre_process()  ← 前置钩子（过滤、转换）
    ↓
DM 配对检查（pairing.py）
    ↓
Session 获取/创建
    ↓
GatewayRunner._run_agent_for_message()
    ↓
AIAgent.run_conversation()
    ↓
Hooks.post_process()  ← 后置钩子
    ↓
PlatformAdapter.send_message()  ← 投递响应
```

### 4.2 出站消息处理

- **长消息拆分**：按平台限制自动拆分
- **流式输出**：支持逐 token 流式投递（CLI）
- **Mirror 模式**（`mirror.py`）：消息镜像到多个平台
- **贴纸缓存**（`sticker_cache.py`）：Telegram 贴纸转换缓存

---

## 五、Hooks 系统 (`gateway/hooks.py`)

### 5.1 内置 Hooks (`gateway/builtin_hooks/`)

钩子可在消息处理前后插入自定义逻辑：
- 消息过滤
- 格式转换
- 日志记录
- 限流

### 5.2 配置

```yaml
# config.yaml
gateway:
  hooks:
    pre_process:
      - name: spam_filter
      - name: rate_limiter
    post_process:
      - name: message_logger
```

---

## 六、DM 配对 (`gateway/pairing.py`)

安全机制：只有经过配对的用户才能通过消息平台与 Agent 交互。

流程：
1. 用户在平台发送 `/start` 或任意消息
2. Agent 验证是否在允许列表
3. 未配对 → 返回配对码
4. 用户在 CLI 中确认配对
5. 后续消息自动放行

---

## 七、平台状态与监控 (`gateway/status.py`)

- 各平台连接状态
- 消息统计
- 错误计数
- 运行时间

通过 `/status` 命令或 API 查询。
