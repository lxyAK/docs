# 飞书认证体系深度分析：oapi-sdk-go 与 cli 架构详解

> **源码仓库：**
> - oapi-sdk-go: https://github.com/larksuite/oapi-sdk-go
> - cli: https://github.com/larksuite/cli

## 一、整体架构关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              larksuite/cli                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────────────┐ │
│  │  credential     │  │  auth           │  │  client                │ │
│  │  (凭证提供者)   │  │  (Token存储)    │  │  (SDK封装层)          │ │
│  └────────┬────────┘  └────────┬────────┘  └───────────┬────────────┘ │
└───────────┼────────────────────┼───────────────────────────┼──────────────┘
            │                    │                           │
            ▼                    ▼                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         larksuite/oapi-sdk-go                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────────────┐ │
│  │  tokenmanager   │  │  reqtranslator  │  │  httptransport         │ │
│  │  (Token管理)    │  │  (请求翻译)     │  │  (HTTP传输)            │ │
│  └─────────────────┘  └─────────────────┘  └────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 二、oapi-sdk-go 认证核心流程

### 2.1 Token 类型体系
oapi-sdk-go 支持 4 种 Token 类型（`AccessTokenType`）：

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| `AccessTokenTypeNone` | 不带 Token | 获取 Token 本身的 API |
| `AccessTokenTypeApp` | 应用级 Token | 应用身份调用 |
| `AccessTokenTypeTenant` | 租户级 Token | 企业自建应用调用 |
| `AccessTokenTypeUser` | 用户级 Token | 用户身份调用（OAuth） |

### 2.2 TokenManager 核心逻辑 (`core/tokenmanager.go`)

```go
// TokenManager 统一管理所有 Token 的获取与缓存
type TokenManager struct {
    cache Cache  // 默认内存缓存，可自定义
}
```

**Token 获取流程：**

1. **先查缓存**：按 `appAccessTokenKey(appId)` 或 `tenantAccessTokenKey(appId, tenantKey)` 从缓存获取
2. **缓存未命中则远程获取**：
   - **自建应用 (AppTypeSelfBuilt)**：
     - AppAccessToken: `POST /open-apis/auth/v3/app_access_token/internal`
     - TenantAccessToken: `POST /open-apis/auth/v3/tenant_access_token/internal`
   - **应用商店应用 (AppTypeMarketplace)**：
     - AppAccessToken: 需要 app_ticket
     - TenantAccessToken: 先用 app_access_token 换取
3. **缓存 Token**：TTL = expire - 30s（预留缓冲）

### 2.3 请求翻译与 Token 注入 (`core/reqtranslator.go`)

关键函数：`newHTTPRequest()` → 根据 Token 类型选择注入方式：

```go
switch accessTokenType {
case AccessTokenTypeApp:
    // 获取或使用传入的 AppAccessToken
    appAccessToken := option.AppAccessToken
    if config.EnableTokenCache && appAccessToken == "" {
        appAccessToken, err = tokenManager.getAppAccessToken(...)
    }
    authorizationToHeader(httpRequest, appAccessToken)  // Authorization: Bearer xxx

case AccessTokenTypeTenant:
    // 获取或使用传入的 TenantAccessToken
    tenantAccessToken := option.TenantAccessToken
    if config.EnableTokenCache {
        tenantAccessToken, err = tokenManager.getTenantAccessToken(...)
    }
    authorizationToHeader(httpRequest, tenantAccessToken)

case AccessTokenTypeUser:
    // 直接使用传入的 UserAccessToken（不缓存）
    authorizationToHeader(httpRequest, option.UserAccessToken)
}
```

### 2.4 传输层与错误重试 (`core/httptransport.go`)

```go
func Request(ctx context.Context, req *ApiReq, config *Config, options ...RequestOptionFunc) (*ApiResp, error)
```

**核心逻辑：**
1. 确定 Token 类型 (`determineTokenType`)
2. 验证配置 (`validate`)
3. 执行请求（最多重试 2 次）：
   - 重试条件：连接错误（dial error）或 Token 失效（`errCodeAccessTokenInvalid` 等）
   - Token 失效时自动清空缓存并重新获取

---

## 三、larksuite/cli 认证体系

### 3.1 三层架构

```
cli 认证体系 = credential（凭证解析） + auth（Token 存储） + client（SDK 调用）
```

### 3.2 CredentialProvider (`internal/credential/credential_provider.go`)

**统一凭证入口，支持多来源：**

1. **Extension 来源**：从 `extension/credential.Provider` 获取
2. **Default 来源**：本地配置文件

**核心方法：**
- `ResolveAccount()`: 解析应用凭证（AppID/AppSecret）
- `ResolveToken()`: 解析访问令牌（UAT/Tenant/App Token）
- `ResolveIdentityHint()`: 解析身份偏好（以用户还是机器人身份运行）

### 3.3 Token 存储 (`internal/auth/token_store.go`)

```go
// StoredUAToken 存储用户访问令牌（User Access Token）
type StoredUAToken struct {
    UserOpenId       string
    AppId            string
    AccessToken      string
    RefreshToken     string
    ExpiresAt        int64  // 过期时间（毫秒）
    RefreshExpiresAt int64
    Scope            string
    GrantedAt        int64
}
```

**存储位置：**
- 使用系统 keychain（macOS Keychain / Windows Credential Locker / Linux libsecret）
- Key 格式：`appId:userOpenId`

**Token 状态判断：**
| 状态 | 条件 |
|------|------|
| valid | now < ExpiresAt - 5min |
| needs_refresh | ExpiresAt -5min ≤ now < RefreshExpiresAt |
| expired | now ≥ RefreshExpiresAt |

### 3.4 Client 层与 SDK 集成 (`internal/client/client.go`)

```go
// APIClient 封装了 lark.Client
type APIClient struct {
    SDK        *lark.Client               // oapi-sdk-go 的客户端
    Credential *credential.CredentialProvider
}
```

**关键流程：**

```go
// 1. 从 CredentialProvider 解析 Token
token, err := c.Credential.ResolveToken(ctx, credential.NewTokenSpec(as, c.Config.AppID))

// 2. 根据身份选择 Token 类型并注入
if as.IsBot() {
    req.SupportedAccessTokenTypes = []larkcore.AccessTokenType{larkcore.AccessTokenTypeTenant}
    opts = append(opts, larkcore.WithTenantAccessToken(token))  // 关闭 SDK 自动获取，直接用这个
} else {
    req.SupportedAccessTokenTypes = []larkcore.AccessTokenType{larkcore.AccessTokenTypeUser}
    opts = append(opts, larkcore.WithUserAccessToken(token))
}

// 3. 调用 SDK 发送请求
c.SDK.Do(ctx, req, opts...)
```

**重要设计点：**
cli 关闭了 SDK 的自动 Token 获取（通过直接传入 Token），Token 的生命周期完全由 cli 自己的 credential 层管理。

---

## 四、认证依赖关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                         larksuite/cli                            │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  credential.CredentialProvider                           │  │
│  │  - ResolveAccount() → Account{AppID, AppSecret}        │  │
│  │  - ResolveToken()   → TokenResult{Token, Scopes}       │  │
│  └────────────────────┬────────────────────────────────────┘  │
│                       │                                         │
│  ┌────────────────────▼────────────────────────────────────┐  │
│  │  auth (Token Store)                                      │  │
│  │  - GetStoredToken(appId, userOpenId) → StoredUAToken   │  │
│  │  - SetStoredToken(token)                                 │  │
│  │  - TokenStatus(token) → valid|needs_refresh|expired     │  │
│  └────────────────────┬────────────────────────────────────┘  │
│                       │                                         │
│  ┌────────────────────▼────────────────────────────────────┐  │
│  │  client.APIClient                                         │  │
│  │  - resolveAccessToken(as) → token                        │  │
│  │  - DoSDKRequest(req, as)                                 │  │
│  └────────────────────┬────────────────────────────────────┘  │
└───────────────────────┼──────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     larksuite/oapi-sdk-go                        │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  lark.Client.Do(ctx, req, opts...)                       │  │
│  │  - WithTenantAccessToken(token) / WithUserAccessToken()  │  │
│  └────────────────────┬────────────────────────────────────┘  │
│                       │                                         │
│  ┌────────────────────▼────────────────────────────────────┐  │
│  │  larkcore.Request()                                       │  │
│  │  - determineTokenType()                                   │  │
│  │  - doRequest() → (最多重试2次)                            │  │
│  └────────────────────┬────────────────────────────────────┘  │
│                       │                                         │
│  ┌────────────────────▼────────────────────────────────────┐  │
│  │  reqtranslator.translate()                                │  │
│  │  - authorizationToHeader(req, token)                     │  │
│  │  - → "Authorization: Bearer <token>"                     │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、关键设计亮点

### 1. SDK 侧：Token 两级缓存策略
- **内存缓存**：避免频繁请求 API
- **自动重试**：Token 失效时自动刷新并重试
- **提前过期**：`expiryDelta = 30s`，在 Token 真正过期前就刷新

### 2. CLI 侧：Token 系统级安全存储
- 使用系统 keychain，不存明文在配置文件
- 支持 Refresh Token 机制

### 3. CLI 侧：CredentialProvider 扩展机制
- 通过 `extension/credential.Provider` 接口支持第三方凭证来源
- 默认实现支持本地配置文件

### 4. 两层 Token 管理解耦
- SDK 层：负责通用 Token 获取、缓存、重试
- CLI 层：负责 CLI 特有 Token 存储（keychain）、身份选择（User/Bot）

---

## 六、自建应用 vs 应用商店应用认证对比

| 维度 | 自建应用 (SelfBuilt) | 应用商店应用 (Marketplace) |
|------|-----------------------|----------------------------|
| AppAccessToken | `/auth/v3/app_access_token/internal` (仅需 AppID+AppSecret) | `/auth/v3/app_access_token` (需 AppID+AppSecret+AppTicket) |
| TenantAccessToken | `/auth/v3/tenant_access_token/internal` (仅需 AppID+AppSecret) | `/auth/v3/tenant_access_token` (需 AppAccessToken+TenantKey) |
| AppTicket | 不需要 | 需要，通过事件推送获取，缓存在 AppTicketManager |
