# feishu-cli vs wecom-cli 技术方案对比

## 一、核心技术栈对比

| 维度 | feishu-cli（飞书） | wecom-cli（企业微信） |
|------|-------------------|---------------------|
| **开发语言** | Go 1.23+ | Rust 2024 Edition |
| **命令行框架** | spf13/cobra | clap (derive) |
| **包管理** | Go Modules | Cargo |
| **发布方式** | npm + 源码编译 | npm（预编译二进制） |
| **Node.js 绑定** | 是（npm 包） | 是（TypeScript wrapper） |
| **Skill 系统** | 20 个内置 Skills | 6 个内置 Skills |

---

## 二、架构设计对比

### feishu-cli 架构（Go）

```
lark-cli/
├── cmd/                    # 命令层
│   ├── api/               # Raw API 调用
│   ├── auth/              # 认证相关
│   ├── config/            # 配置管理
│   ├── schema/            # API 元数据
│   └── service/           # 业务域命令（12个）
├── internal/              # 内部实现
│   ├── auth/              # OAuth 认证
│   ├── client/            # API 客户端
│   ├── credential/        # 凭证管理（Keyring）
│   ├── output/            # 多格式输出
│   └── registry/          # 命令注册
├── skills/                # 20 个 Agent Skills
└── main.go
```

**核心设计亮点：**
- ✅ **三层命令系统**：Shortcuts（人类/AI 友好）→ API Commands（平台同步）→ Raw API（全覆盖）
- ✅ **官方 SDK 集成**：使用 `larksuite/oapi-sdk-go/v3`
- ✅ **多格式输出**：JSON/Table/CSV/NDJSON/Pretty
- ✅ **安全设计**：Keyring 存储、注入防护、输出脱敏
- ✅ **Schema 自省**：`lark-cli schema <endpoint>` 查看 API 元数据

---

### wecom-cli 架构（Rust）

```
wecom-cli/
├── src/                   # Rust 核心
│   ├── main.rs
│   ├── auth/              # 认证
│   ├── cmd/               # 命令定义
│   ├── crypto/            # 加密（AES-GCM）
│   ├── mcp/               # MCP 协议支持
│   └── media/             # 多媒体处理
├── packages/              # 预编译二进制
│   ├── darwin-arm64/
│   ├── darwin-x64/
│   ├── linux-arm64/
│   ├── linux-x64/
│   └── win32-x64/
├── skills/                # 6 个 Agent Skills
├── bin/                   # Node.js wrapper
│   └── wecom.js
└── Cargo.toml
```

**核心设计亮点：**
- ✅ **Rust 原生性能**：零成本抽象、内存安全
- ✅ **预编译分发**：5 个平台预编译二进制，无需编译
- ✅ **MCP 协议支持**：内置 MCP (Model Context Protocol)
- ✅ **加密存储**：AES-GCM 加密凭证
- ✅ **TypeScript 轻量包装**：Node.js 生态友好

---

## 三、功能覆盖对比

| 功能域 | feishu-cli | wecom-cli |
|--------|-----------|-----------|
| 📅 日历/日程 | ✅ 日程、忙闲、时间建议 | ✅ 日程、闲忙查询 |
| 💬 消息/IM | ✅ 收发、群管理、搜索、话题 | ✅ 会话、消息、多媒体 |
| 📄 文档 | ✅ Markdown 读写、媒体、画板 | ✅ 文档 CRUD |
| 📁 云空间 | ✅ 上传下载、权限、评论 | ❌ |
| 📊 Base/智能表格 | ✅ 表/字段/记录/视图/仪表盘 | ✅ 智能表格 |
| 📈 Sheets/表格 | ✅ 读写追加查找导出 | ❌ |
| ✅ Tasks/待办 | ✅ 任务/清单/子任务/提醒 | ✅ 待办 CRUD |
| 📚 Wiki/知识库 | ✅ 空间/节点/文档 | ❌ |
| 👤 通讯录 | ✅ 用户搜索、 profile | ✅ 成员列表、搜索 |
| 📧 邮件 | ✅ 收发搜索草稿 | ❌ |
| 🎥 会议 | ✅ 会议记录、纪要、录音 | ✅ 预约会议 |
| ✍️ 审批 | ✅ 审批任务、流程 | ❌ |
| 🎨 白板 | ✅ 画板 DSL 渲染 | ❌ |
| 🔌 事件订阅 | ✅ WebSocket 实时事件 | ❌ |

**统计：**
- feishu-cli：12 个业务域，200+ 命令
- wecom-cli：7 个业务域，命令数量未知

---

## 四、AI Agent 支持对比

### feishu-cli Skills（20 个）

| Skill | 描述 |
|-------|------|
| `lark-shared` | 共享：配置、认证、身份切换、安全规则 |
| `lark-calendar` | 日历：日程、忙闲、时间建议 |
| `lark-im` | 消息：收发、群管理、搜索、媒体下载 |
| `lark-doc` | 文档：Markdown 增删改查 |
| `lark-drive` | 云空间：上传下载、权限、评论 |
| `lark-sheets` | 电子表格：读写追加查找导出 |
| `lark-base` | 多维表格：表/字段/记录/视图/分析 |
| `lark-task` | 任务：任务/清单/子任务/提醒 |
| `lark-mail` | 邮件：浏览、搜索、收发、草稿 |
| `lark-contact` | 通讯录：用户搜索、profile |
| `lark-wiki` | 知识库：空间、节点、文档 |
| `lark-event` | 事件：WebSocket 订阅、正则路由 |
| `lark-vc` | 会议：会议记录、纪要 |
| `lark-whiteboard` | 白板：DSL 渲染 |
| `lark-minutes` | 纪要：元数据、AI 产物 |
| `lark-openapi-explorer` | API 探索：官方文档 |
| `lark-skill-maker` | 自定义 Skill 框架 |
| `lark-approval` | 审批：审批任务、流程 |
| `lark-workflow-meeting-summary` | 工作流：会议纪要聚合 |
| `lark-workflow-standup-report` | 工作流：站会总结 |

---

### wecom-cli Skills（6 个）

| Skill | 描述 |
|-------|------|
| `wecomcli-contact` | 通讯录 |
| `wecomcli-doc` | 文档 |
| `wecomcli-meeting` | 会议 |
| `wecomcli-msg` | 消息 |
| `wecomcli-schedule` | 日程 |
| `wecomcli-todo` | 待办 |

---

## 五、依赖库对比

### feishu-cli 核心依赖（Go）

```go
// 命令行
github.com/spf13/cobra v1.10.2

// TUI（交互式 UI）
github.com/charmbracelet/huh v1.0.0
github.com/charmbracelet/lipgloss v1.1.0

// 官方 SDK
github.com/larksuite/oapi-sdk-go/v3 v3.5.3

// JSON 处理
github.com/itchyny/gojq v0.12.17
github.com/tidwall/gjson v1.18.0

// 安全
github.com/zalando/go-keyring v0.2.8

// 其他
github.com/smartystreets/goconvey v1.8.1  // 测试
github.com/stretchr/testify v1.11.1       // 断言
```

---

### wecom-cli 核心依赖（Rust）

```toml
# 异步运行时
tokio = { version = "1.50.0", features = ["full"] }

# 命令行
clap = { version = "4", features = ["derive"] }

# 序列化
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# 日志
tracing = "0.1"
tracing-subscriber = "0.3"

# HTTP
reqwest = { version = "0.13.2", features = ["json", "multipart", "stream", "rustls"] }

# 安全
aes-gcm = "0.10"  # 加密
keyring = "3"      # 系统密钥链
sha2 = "0.10"      # 哈希

# TUI
cliclack = "0.5"   # 交互式提示
qr2term = "0.3"    # 终端二维码

# 工具
anyhow = "1"       # 错误处理
dirs = "6.0.0"     # 目录操作
tempfile = "3.27.0"
```

---

## 六、安全设计对比

| 安全特性 | feishu-cli | wecom-cli |
|---------|-----------|-----------|
| 凭证存储 | OS Keyring（go-keyring） | OS Keyring + AES-GCM 加密 |
| 注入防护 | ✅ 输入注入防护 | ❓ 未知 |
| 输出脱敏 | ✅ 终端输出脱敏 | ❓ 未知 |
| 权限范围 | ✅ OAuth 细粒度 scope | ✅ Bot Secret |
| 身份切换 | ✅ 用户/机器人身份切换 | ❓ 未知 |
| 审计日志 | ❓ 未知 | ✅ tracing 日志 |

---

## 七、部署 & 分发对比

### feishu-cli

```bash
# 方式 1：npm 安装（推荐）
npm install -g @larksuite/cli
npx skills add larksuite/cli -y -g

# 方式 2：源码编译（需 Go 1.23+）
git clone https://github.com/larksuite/cli.git
cd cli
make install
```

**特点：**
- ✅ npm 分发，Node.js 生态友好
- ⚠️ 源码编译需 Go 环境
- ✅ `npx skills` 集成 Skill 管理

---

### wecom-cli

```bash
# npm 安装（自动下载对应平台二进制）
npm install -g @wecom/cli
npx skills add WeComTeam/wecom-cli -y -g
```

**特点：**
- ✅ 5 平台预编译二进制，无需编译
- ✅ 安装速度快
- ✅ TypeScript wrapper 无缝集成

---

## 八、技术决策建议

### 选择 feishu-cli 如果：

✅ 你用的是飞书/Lark  
✅ 需要完整的业务域覆盖（12 个 vs 7 个）  
✅ 需要 AI Agent Skills（20 个 vs 6 个）  
✅ 需要三层命令系统（Shortcuts → API → Raw）  
✅ 需要多格式输出（JSON/Table/CSV/NDJSON）  
✅ 需要 Schema 自省、事件订阅、白板等高级特性  
✅ 团队熟悉 Go 生态  

---

### 选择 wecom-cli 如果：

✅ 你用的是企业微信  
✅ 需要极致的原生性能（Rust）  
✅ 需要 MCP 协议支持  
✅ 希望安装即用（预编译二进制）  
✅ 团队熟悉 Rust 生态  

---

## 九、总结

| 维度 | feishu-cli 优势 | wecom-cli 优势 |
|------|----------------|---------------|
| **功能完整性** | 🏆 12 域、200+ 命令、20 Skills | 7 域、6 Skills |
| **性能** | Go 良好 | 🏆 Rust 极致 |
| **分发体验** | npm + 源码 | 🏆 预编译二进制 |
| **AI 友好** | 🏆 20 Skills、三层命令 | 6 Skills、MCP |
| **学习曲线** | Go 生态广泛 | Rust 门槛较高 |
| **官方支持** | 🏆 飞书官方维护 | 企业微信团队 |

---

**最终建议：**
- **飞书用户**：无脑选 `feishu-cli`，功能、生态、文档都碾压
- **企业微信用户**：目前只有 `wecom-cli` 可用，期待后续功能完善

---

## 十、通信架构对比

### feishu-cli（飞书）- 两层 HTTP

```
┌─────────────────────────────────────────────────────────────┐
│                        feishu-cli                            │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  1. 官方 SDK 层（99% 的请求）                        │  │
│  │  larksuite/oapi-sdk-go/v3                            │  │
│  │  → 封装好的 Lark Open API                             │  │
│  │  → 自动处理认证、重试、错误码                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                          ↓ HTTP/HTTPS                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  2. 原生 HTTP 层（特殊场景）                          │  │
│  │  net/http.Client                                      │  │
│  │  → OAuth 认证流程                                      │  │
│  │  → 文件下载（DoStream）                                │  │
│  │  → MCP 协议（如果有）                                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
                          ↓
              Lark/Feishu Open Platform
```

**核心特点：**
- ✅ **99% 请求走官方 SDK**：`larksuite/oapi-sdk-go/v3`
- ✅ 原生 HTTP 仅用于：OAuth 认证、流式下载（`DoStream`）
- ✅ 自动处理：Token 刷新、重试、错误码解析
- ✅ 支持：用户身份（User Access Token）+ 机器人身份（Tenant Access Token）

---

### wecom-cli（企业微信）- 三层架构

```
┌─────────────────────────────────────────────────────────────┐
│                       wecom-cli                              │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  1. CLI 层（Rust）                                    │  │
│  │  clap + JSON-RPC 2.0                                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                          ↓ JSON-RPC                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  2. MCP Server 层                                     │  │
│  │  Model Context Protocol                                │  │
│  │  → 工具调用统一入口                                    │  │
│  │  → 跨语言、跨平台协议                                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                          ↓ HTTP/HTTPS                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  3. 企业微信 API 层                                    │  │
│  │  reqwest (Rust HTTP 客户端)                           │  │
│  │  → 直接调用企业微信开放平台 API                         │  │
│  │  → JSON 序列化/反序列化                                │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
                          ↓
              企业微信开放平台 API
```

**核心特点：**
- ✅ **MCP 协议层**：JSON-RPC 2.0 over HTTP
- ✅ **HTTP 客户端**：`reqwest`（Rust 原生异步）
- ✅ **超时控制**：默认 30s，媒体下载 120s
- ✅ **分层清晰**：CLI → MCP → 企业微信 API

---

## 十一、通信方式总结

| 维度 | feishu-cli | wecom-cli |
|------|-----------|-----------|
| **主要协议** | 官方 SDK（封装 HTTP） | JSON-RPC 2.0 over HTTP |
| **HTTP 客户端** | Go `net/http` | Rust `reqwest` |
| **认证方式** | SDK 自动管理（User/Tenant Token） | Bot Secret + MCP |
| **架构层数** | 2 层（SDK + 原生 HTTP） | 3 层（CLI → MCP → API） |
| **是否全是 HTTP？** | ✅ 是（SDK 内部也是 HTTP） | ✅ 是（MCP 底层也是 HTTP） |

**核心区别：**
- **feishu-cli**：用官方 SDK 省事儿，自动处理认证、重试
- **wecom-cli**：通过 MCP 协议层，更灵活但多了一层

---

## 十二、SDK 确认

**是的，`larksuite/oapi-sdk-go/v3` 就是官方 SDK！**

仓库地址：https://github.com/larksuite/oapi-sdk-go

这是飞书官方维护的 Go SDK，feishu-cli 99% 的 API 调用都走这个 SDK。

---

## 十三、业务对接实现对比

### feishu-cli 的查询/写入处理

#### 查询流程（示例：获取日程列表）

```go
// 1. 用户输入
lark-cli calendar events instance_view \
  --params '{"calendar_id":"primary","start_time":"1700000000","end_time":"1700086400"}'

// 2. 内部处理（service.go）
func serviceMethodRun(opts *ServiceMethodOptions) error {
    // 解析参数
    request := buildServiceRequest(opts)
    
    // 调用 SDK
    resp, err := ac.DoAPI(ctx, request)
    
    // 处理响应
    return client.HandleResponse(resp, ...)
}

// 3. SDK 层（client.go）
func (c *APIClient) DoSDKRequest(ctx context.Context, req *larkcore.ApiReq, as core.Identity) (*larkcore.ApiResp, error) {
    // 自动获取 Token
    token, _ := c.resolveAccessToken(ctx, as)
    
    // 身份选择
    if as.IsBot() {
        opts = append(opts, larkcore.WithTenantAccessToken(token))
    } else {
        opts = append(opts, larkcore.WithUserAccessToken(token))
    }
    
    // 执行请求
    return c.SDK.Do(ctx, req, opts...)
}
```

**查询特点：**
- ✅ **自动分页**：`--page-all` 自动遍历所有页
- ✅ **多格式输出**：JSON/Table/CSV/NDJSON
- ✅ **JQ 过滤**：`--jq` 实时过滤结果
- ✅ **Schema 自省**：`lark-cli schema calendar.events.instance_view` 查看参数定义

---

#### 写入流程（示例：发送消息）

```go
// 1. 用户输入
lark-cli im messages create \
  --params '{"receive_id_type":"chat_id"}' \
  --data '{"receive_id":"oc_xxx","msg_type":"text","content":"{\"text\":\"Hello\"}"}'

// 2. 内部处理
func buildServiceRequest(opts *ServiceMethodOptions) (client.RawApiRequest, error) {
    // 解析 --data JSON
    data, _ := cmdutil.ParseOptionalBody(httpMethod, opts.Data)
    
    return client.RawApiRequest{
        Method: "POST",
        URL:    "/open-apis/im/v1/messages",
        Params: queryParams,
        Data:   data,  // 写入数据
        As:     opts.As,
    }
}

// 3. 执行前检查
if opts.DryRun {
    return serviceDryRun(f, request, config, opts.Format)
}
```

**写入特点：**
- ✅ **Dry Run 预览**：`--dry-run` 先看请求不执行
- ✅ **参数验证**：自动检查必填参数、scope 权限
- ✅ **错误增强**：返回权限不足时提示登录命令

---

### wecom-cli 的查询/写入处理

#### 查询流程（示例：获取通讯录）

```rust
// 1. 用户输入
wecom-cli contact get_userlist '{}'

// 2. CLI 层（call.rs）
pub async fn handle_call_cmd(category_name: &str, matches: &ArgMatches) -> Result<()> {
    // 解析 JSON 参数
    let parsed_args = if let Some(args) = args {
        serde_json::from_str(args)?
    } else {
        json!({})
    };
    
    // 构造 JSON-RPC 请求
    let params = json!({
        "name": method,
        "arguments": parsed_args,
    });
    
    // 发送到 MCP Server
    let res = json_rpc::send(category_name, "tools/call", Some(params), timeout_ms).await?;
}

// 3. JSON-RPC 层（json_rpc.rs）
pub async fn send(category: &str, method: &str, params: Option<Value>) -> Result<Value> {
    let mcp_url = mcp::get_mcp_url(category).await?;
    
    let body = JsonRpcRequest {
        jsonrpc: "2.0",
        id: mcp::gen_req_id("mcp_rpc"),
        method: method.to_string(),
        params,
    };
    
    // HTTP POST 到 MCP Server
    let response = reqwest::Client::builder()
        .build()?
        .post(&mcp_url)
        .json(&body)
        .send()
        .await?;
}
```

**查询特点：**
- ✅ **JSON-RPC 2.0 协议**：标准协议，跨语言友好
- ✅ **超时控制**：默认 30s，媒体下载 120s
- ⚠️ **单层结构**：没有 SDK 封装，直接调用 MCP

---

#### 写入流程（示例：发送消息）

```rust
// 1. 用户输入
wecom-cli msg send_message '{"touser":"xxx","msgtype":"text","text":{"content":"Hello"}}'

// 2. 媒体下载特殊处理（call.rs）
if method == "get_msg_media" {
    res = media::intercept_media_response(res).await?;
}
```

**写入特点：**
- ✅ **媒体拦截**：下载文件时特殊处理
- ⚠️ **无 Dry Run**：没有预览机制
- ⚠️ **参数简单**：直接透传 JSON，验证少

---

## 十四、业务对接优劣势对比

| 维度 | feishu-cli 优势 | feishu-cli 劣势 | wecom-cli 优势 | wecom-cli 劣势 |
|------|----------------|----------------|----------------|----------------|
| **查询体验** | 自动分页、多格式、JQ 过滤 | 复杂度稍高 | 简单直接 | 功能少，无分页 |
| **写入安全** | Dry Run、参数验证、scope 检查 | — | — | 无预览、验证少 |
| **SDK 封装** | 官方 SDK，自动重试、Token 刷新 | — | — | 无 SDK，直接 HTTP |
| **协议设计** | 三层命令（Shortcuts/API/Raw） | 学习曲线 | MCP 标准协议 | 多一层 overhead |
| **身份支持** | 用户+机器人双身份 | — | — | 仅 Bot Secret |
| **生态集成** | npm + skills 生态 | — | 预编译二进制 | Skill 少 |

---

## 十五、代码示例对比

### 场景：查询日程列表

#### feishu-cli

```bash
# 方式 1：Shortcut（推荐，人类/AI 友好）
lark-cli calendar +agenda

# 方式 2：API Command
lark-cli calendar events instance_view \
  --params '{"calendar_id":"primary","start_time":"1700000000","end_time":"1700086400"}' \
  --format table \
  --page-all

# 方式 3：Raw API（完全自定义）
lark-cli api GET /open-apis/calendar/v4/calendars/primary/events
```

**优势：** 三层选择，从简单到复杂
**劣势：** 选项多，新手可能困惑

---

#### wecom-cli

```bash
# 仅一种方式：直接调用
wecom-cli schedule get_schedule_list '{"start_time":1700000000,"end_time":1700086400}'
```

**优势：** 简单直接
**劣势：** 无分页、无格式化、无过滤

---

### 场景：发送消息

#### feishu-cli

```bash
# 先预览（安全）
lark-cli im +messages-send \
  --chat-id oc_xxx \
  --text "Hello" \
  --dry-run

# 实际发送
lark-cli im +messages-send \
  --chat-id oc_xxx \
  --text "Hello"
```

**优势：** Dry Run 预览、参数友好
**劣势：** 需要多打几个字

---

#### wecom-cli

```bash
# 直接发送（无预览）
wecom-cli msg send_message \
  '{"touser":"ou_xxx","msgtype":"text","text":{"content":"Hello"}}'
```

**优势：** 一行搞定
**劣势：** 手滑就发出去了，JSON 容易写错

---

## 十六、最终总结

| 项目 | 适用场景 | 推荐度 |
|------|---------|-------|
| **feishu-cli** | 飞书用户、需要生产级工具、AI Agent 集成 | ⭐⭐⭐⭐⭐ |
| **wecom-cli** | 企业微信用户、简单脚本、MCP 协议集成 | ⭐⭐⭐ |

**核心差异：**
- **feishu-cli**：工程化完善，安全、可靠、功能全
- **wecom-cli**：简洁直接，潜力大但目前功能少
