# feishu-cli vs wecom-cli 业务对接深度分析

## 问题 1：SDK 确认

**是的，`larksuite/oapi-sdk-go/v3` 就是官方 SDK！**

仓库地址：https://github.com/larksuite/oapi-sdk-go

这是飞书官方维护的 Go SDK，feishu-cli 99% 的 API 调用都走这个 SDK。

---

## 问题 2：业务对接实现对比

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

## 优劣势对比总结

| 维度 | feishu-cli 优势 | feishu-cli 劣势 | wecom-cli 优势 | wecom-cli 劣势 |
|------|----------------|----------------|----------------|----------------|
| **查询体验** | 自动分页、多格式、JQ 过滤 | 复杂度稍高 | 简单直接 | 功能少，无分页 |
| **写入安全** | Dry Run、参数验证、scope 检查 | — | — | 无预览、验证少 |
| **SDK 封装** | 官方 SDK，自动重试、Token 刷新 | — | — | 无 SDK，直接 HTTP |
| **协议设计** | 三层命令（Shortcuts/API/Raw） | 学习曲线 | MCP 标准协议 | 多一层 overhead |
| **身份支持** | 用户+机器人双身份 | — | — | 仅 Bot Secret |
| **生态集成** | npm + skills 生态 | — | 预编译二进制 | Skill 少 |

---

## 代码示例对比

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

## 最终总结

| 项目 | 适用场景 | 推荐度 |
|------|---------|-------|
| **feishu-cli** | 飞书用户、需要生产级工具、AI Agent 集成 | ⭐⭐⭐⭐⭐ |
| **wecom-cli** | 企业微信用户、简单脚本、MCP 协议集成 | ⭐⭐⭐ |

**核心差异：**
- **feishu-cli**：工程化完善，安全、可靠、功能全
- **wecom-cli**：简洁直接，潜力大但目前功能少
