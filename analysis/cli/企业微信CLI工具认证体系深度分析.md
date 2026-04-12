# 企业微信 CLI 工具认证体系深度分析

> 源码仓库：https://github.com/WecomTeam/wecom-cli
> 分析时间：2026-04-08
> 语言：Rust 1.75+，约 17 次提交

## 一、整体架构概览

wecom-cli 的认证体系是一个**基于 Bot ID + Secret 的轻量级凭证模型**，没有传统的 OAuth access_token 刷新机制。它的认证流程分为三层：

```
┌─────────────────────────────────────────────────────┐
│                    用户交互层                         │
│          init 命令 (扫码 / 手动输入)                  │
├─────────────────────────────────────────────────────┤
│                    凭证管理层                         │
│    Bot { id, secret, create_time }                  │
│    ├─ 加密存储: ~/.config/wecom/bot.enc              │
│    └─ 密钥管理: keyring / .encryption_key            │
├─────────────────────────────────────────────────────┤
│                    请求签名层                         │
│    SHA256(secret + bot_id + time + nonce)           │
│    → MCP Config 获取 (一次性)                        │
│    → JSON-RPC 调用 (每次通过 MCP URL)                │
└─────────────────────────────────────────────────────┘
```

## 二、核心模块结构

| 文件 | 职责 |
|------|------|
| `src/auth/mod.rs` | 认证模块入口，导出 Bot 和扫码功能 |
| `src/auth/bot.rs` | Bot 凭证结构体、读写、清除 |
| `src/auth/qrcode.rs` | 扫码接入完整流程 |
| `src/cmd/init.rs` | init 子命令入口，编排初始化流程 |
| `src/config.rs` | 配置目录、环境变量覆盖、User-Agent |
| `src/constants.rs` | 常量定义（配置目录路径、MCP 端点） |
| `src/crypto/mod.rs` | 加密模块入口 |
| `src/crypto/cipher.rs` | AES-256-GCM 加解密实现 |
| `src/crypto/keystore.rs` | 密钥生成、存储、加载（keyring + 文件双通道） |
| `src/mcp/config.rs` | MCP 配置请求签名、获取、持久化 |
| `src/json_rpc.rs` | JSON-RPC 2.0 业务请求发送 |

## 三、凭证初始化流程 (`wecom-cli init`)

`src/cmd/init.rs` 定义了两种接入方式：

### 方式一：扫码接入（推荐）

```
用户执行 init → 选择扫码
  → 请求 https://work.weixin.qq.com/ai/qc/generate?source=wecom_cli_external&plat={os}
  → 获取 scode + auth_url
  → 终端渲染二维码 (qr2term)
  → 轮询 https://work.weixin.qq.com/ai/qc/query_result?scode={scode}
  → 每3秒轮询一次，5分钟超时
  → 扫码成功 → 返回 { botid, secret }
```

关键细节：
- 平台识别码：macOS=1, Windows=2, Linux=3（`src/auth/qrcode.rs:58-68`）
- `source` 固定为 `wecom_cli_external`
- 轮询间隔 3s，超时 300s（`src/auth/qrcode.rs:18-19`）
- 扫码结果由服务端直接下发 Bot ID 和 Secret，用户无需手动复制

### 方式二：手动输入

用户直接输入 Bot ID 和 Secret（Secret 使用密码掩码输入，`src/cmd/init.rs:33-39`）

### 凭证验证与回滚

初始化后立即调用 `fetch_mcp_config` 验证凭证有效性（`src/cmd/init.rs:45-80`）：
- 验证失败 → **自动回滚**：删除已保存的 `bot.enc` 和 `mcp_config.enc`
- 验证成功 → 凭证和 MCP 配置都持久化到磁盘

## 四、凭证存储与加密体系

### 存储位置

默认配置目录可通过环境变量 `WECOM_CLI_CONFIG_DIR` 覆盖，否则为 `~/.config/wecom`。

| 文件 | 路径 | 内容 |
|------|------|------|
| `bot.enc` | `~/.config/wecom/bot.enc` | 加密的 Bot 凭证（JSON: id, secret, create_time） |
| `mcp_config.enc` | `~/.config/wecom/mcp_config.enc` | 加密的 MCP 服务端点配置 |
| `.encryption_key` | `~/.config/wecom/.encryption_key` | Base64 编码的 AES-256 密钥（文件回退方案） |

### 加密方案：AES-256-GCM

```
密钥生成 → rand::rng() 生成 32 字节随机密钥
密钥存储 → 优先 keyring (系统密钥链)，回退到文件
加密过程 → JSON序列化 → AES-256-GCM 加密 (随机 nonce)
文件写入 → tempfile 原子写入 + sync_all + 权限 0o600
```

加密数据格式：`nonce (12 bytes) || ciphertext || tag (16 bytes)`

### 密钥管理双通道策略

`src/crypto/keystore.rs:52-91` 实现：

```
优先级：文件密钥 → 系统密钥链 (keyring)
         ↑                         ↑
    总是写入                  尝试写入，失败仅 warn
    .encryption_key          macOS Keychain / Windows Credential Manager / Linux Secret Service
```

- `load_existing_key()`：先读文件，再读 keyring
- `save_key()`：文件总是写入；keyring 尝试写入但失败不阻断
- `try_decrypt_data()`：先尝试文件密钥解密，失败再重试

### 安全性设计亮点

1. **AES-256-GCM 认证加密**：同时提供保密性和完整性（防篡改）
2. **随机 nonce**：每次加密使用 OS 随机数生成器产生 96-bit nonce
3. **原子写入**：tempfile + persist 模式，防止半写状态
4. **文件权限**：Unix 下 0o600（仅 owner 可读写）
5. **Secret 不明文落盘**：所有敏感数据加密后才写入磁盘

## 五、请求签名与认证注入

### MCP Config 获取（唯一需要签名的请求）

`src/mcp/config.rs:37-63` 实现签名请求：

```
签名算法: SHA256(secret + bot_id + time + nonce)
请求参数: {
    bot_id:        Bot ID
    time:          Unix 时间戳 (秒)
    nonce:         随机请求ID "mcp_{timestamp_ms}_{random_hex}"
    signature:     上述签名
    bind_source:   1=手动输入, 2=扫码
    cli_version:   CLI 版本号
}
```

POST 到 `https://qyapi.weixin.qq.com/cgi-bin/aibot/cli/get_mcp_config`

响应返回各业务品类的 MCP 服务端点列表：
```json
{
  "errcode": 0,
  "list": [
    {
      "url": "https://...",
      "type": "streamable-http",
      "biz_type": "contact",
      "is_authed": true
    }
  ]
}
```

这个列表被加密保存到 `mcp_config.enc`，后续业务调用不再需要签名。

### 业务请求（JSON-RPC）

`src/json_rpc.rs:16-59` 实现：

```
1. 从 mcp_config.enc 读取品类对应的 MCP URL
2. 构造 JSON-RPC 2.0 请求: { jsonrpc: "2.0", id, method: "tools/call", params }
3. POST 到 MCP URL，无额外认证头
```

**关键发现**：业务请求本身不携带任何认证信息。认证完全依赖 MCP 服务端通过 URL 进行的会话绑定。MCP URL 本身就是一个已认证的端点（类似一个带 token 的 URL），拿到 URL 就等于拿到了访问权限。

## 六、认证流程完整时序图

```
用户                  CLI                         企微服务器
 │                    │                              │
 │  wecom-cli init    │                              │
 │───────────────────>│                              │
 │                    │  GET /ai/qc/generate         │
 │                    │─────────────────────────────>│
 │                    │  { scode, auth_url }         │
 │                    │<─────────────────────────────│
 │  展示二维码         │                              │
 │<───────────────────│                              │
 │  用户扫码           │                              │
 │──────────────────────────────────────────────────>│
 │                    │  GET /ai/qc/query_result     │
 │                    │─────────────────────────────>│  (轮询，3s间隔)
 │                    │  { status: "success",        │
 │                    │    bot_info: {botid,secret} } │
 │                    │<─────────────────────────────│
 │                    │                              │
 │                    │  POST get_mcp_config         │
 │                    │  {bot_id, time, nonce,       │
 │                    │   signature, bind_source}    │
 │                    │─────────────────────────────>│
 │                    │  { errcode:0, list:[...] }   │
 │                    │<─────────────────────────────│
 │                    │                              │
 │                    │  保存 bot.enc + mcp_config.enc
 │  初始化完成 ✅      │                              │
 │<───────────────────│                              │
 │                    │                              │
 │  contact get_userlist '{}'                        │
 │───────────────────>│                              │
 │                    │  POST {mcp_url}              │
 │                    │  JSON-RPC { tools/call, ... } │
 │                    │  (无额外认证头)                │
 │                    │─────────────────────────────>│
 │                    │  结果                         │
 │                    │<─────────────────────────────│
 │  输出结果           │                              │
 │<───────────────────│                              │
```

## 七、关键设计特点总结

| 维度 | 设计选择 |
|------|---------|
| **凭证模型** | Bot ID + Secret（非传统的 corp_id + corp_secret + access_token） |
| **Token 管理** | 无 access_token 概念，MCP URL 本身即认证令牌 |
| **凭证获取** | 扫码（服务端下发）优先，手动输入为备选 |
| **凭证存储** | AES-256-GCM 加密 + 原子写入 |
| **密钥管理** | 系统密钥链优先，文件回退 |
| **请求签名** | 仅 MCP Config 获取时使用 SHA256 签名 |
| **业务认证** | 依赖已认证的 MCP URL，无额外认证头 |
| **会话模型** | 无状态，每次调用独立（URL 即凭证） |
| **凭证生命周期** | 无过期机制，Secret 一旦获取长期有效 |
| **安全回退** | 验证失败自动清除已保存凭证 |

## 八、与飞书 CLI 认证的差异对比

| 对比维度 | 企业微信 CLI | 飞书 CLI |
|---------|------------|---------|
| 凭证模型 | Bot ID + Secret | app_id + app_secret |
| Token 类型 | 无 access_token | tenant_access_token / user_access_token |
| Token 刷新 | 不需要 | 需要定期刷新（2小时过期） |
| 凭证获取 | 扫码/手动输入 | 管理后台手动创建应用 |
| 业务认证 | MCP URL 隐式认证 | access_token 请求头 |
| 加密存储 | AES-256-GCM + keyring | 明文配置文件 |
| 多租户 | 不支持 | 支持（tenant/user token 区分） |
| 凭证过期 | 无（长期有效） | 有（token 过期需刷新） |

企业微信 CLI 的认证设计明显更精简：无需管理 token 过期和刷新，通过 MCP URL 实现隐式认证，用户体验更顺滑。

## 九、潜在安全风险

1. **MCP URL 泄露即凭证泄露**：URL 本身承载了认证能力，一旦泄露等同于 token 泄露
2. **无凭证轮换机制**：Bot Secret 长期有效，没有自动轮换
3. **加密密钥与密文同目录**：`.encryption_key` 和 `bot.enc` 都在 `~/.config/wecom/`，文件权限是主要防线
4. **无 PIN/密码保护**：密钥存储依赖系统 keyring，无用户层密码保护
