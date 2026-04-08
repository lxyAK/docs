# 自研 CLI 工具架构决策分析

## 深度推理：先明确你的场景

在选择 feishu-cli 还是 wecom-cli 架构之前，先回答以下问题：

---

### 问题 1：你的业务系统是什么样的？

| 特征 | 推荐架构 | 原因 |
|------|---------|------|
| **已有官方 SDK** | feishu-cli | 直接封装 SDK，省事儿 |
| **只有 HTTP API** | wecom-cli 或混合 | 需要自己封装 HTTP 层 |
| **API 数量 < 20** | wecom-cli（简化版） | 简单直接，不用搞太复杂 |
| **API 数量 > 50** | feishu-cli | 需要分层、自动化生成 |
| **需要支持 AI Agent** | feishu-cli | 三层命令 + Skills 生态 |
| **只是内部工具** | wecom-cli | 够用就行，快速上线 |

---

### 问题 2：你的团队技术栈是什么？

| 团队栈 | 推荐语言 | 推荐架构 |
|-------|---------|---------|
| **Go 为主** | Go | feishu-cli 架构 |
| **Rust 为主** | Rust | wecom-cli 架构 |
| **Node/TypeScript** | TypeScript | 两者都可参考，用 Node 实现 |
| **Python 为主** | Python | 参考架构思想，用 Python 实现 |

---

## 推荐方案：混合架构（取两者精华）

我建议你**不要完全照搬某一个**，而是采用混合架构：

```
┌─────────────────────────────────────────────────────────┐
│                    你的 CLI 工具                         │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌───────────────────────────────────────────────────┐  │
│  │  1. 用户体验层（参考 feishu-cli）                  │  │
│  │  ├── Shortcuts（人类/AI 友好的简化命令）          │  │
│  │  ├── Dry Run（安全预览）                           │  │
│  ├── 多格式输出（JSON/Table/CSV）                   │  │
│  └── Schema 自省（查看 API 文档）                   │  │
│  └───────────────────────────────────────────────────┘  │
│                          ↓                                │
│  ┌───────────────────────────────────────────────────┐  │
│  │  2. 业务逻辑层（可参考任意一个）                  │  │
│  │  ├── 参数验证                                     │  │
│  │  ├── 分页处理                                     │  │
│  │  └── 错误增强（给用户有用的提示）                 │  │
│  └───────────────────────────────────────────────────┘  │
│                          ↓                                │
│  ┌───────────────────────────────────────────────────┐  │
│  │  3. 协议层（灵活选择）                            │  │
│  │  ├── 方案 A：直接 HTTP（简单场景）                │  │
│  │  ├── 方案 B：封装 SDK（如果有官方 SDK）          │  │
│  │  └── 方案 C：JSON-RPC（如果需要跨语言）          │  │
│  └───────────────────────────────────────────────────┘  │
│                                                           │
└─────────────────────────────────────────────────────────┘
                          ↓
                    你的业务系统 API
```

---

## 分阶段实施建议

### 阶段 1：MVP（最小可行性产品）- 参考 wecom-cli

**目标**：快速上线，能用就行

**架构选择**：简化版 wecom-cli

**实现内容**：
1. 简单的命令行框架（cobra / clap / commander）
2. 直接 HTTP 调用你的业务 API
3. JSON 格式输出
4. 基础的认证（Token / API Key）

**时间估计**：1-2 周

---

### 阶段 2：增强体验 - 参考 feishu-cli

**目标**：让工具更好用

**添加内容**：
1. **Shortcuts**：简化常用命令
   ```bash
   # 之前
   mycli order list --status pending --page-size 100
   
   # 之后
   mycli order +pending
   ```

2. **Dry Run**：安全预览
   ```bash
   mycli order create --data "{...}" --dry-run
   ```

3. **多格式输出**：
   ```bash
   mycli order list --format table
   mycli order list --format csv > orders.csv
   ```

4. **分页支持**：
   ```bash
   mycli order list --page-all
   ```

**时间估计**：2-3 周

---

### 阶段 3：AI Agent 友好 - 参考 feishu-cli

**目标**：让 AI 也能用

**添加内容**：
1. **Skills 系统**：结构化的 Skill 定义
2. **Schema 自省**：`mycli schema order.create` 查看参数
3. **机器可读输出**：NDJSON / JSON Lines

**时间估计**：2-3 周

---

## 具体技术选型建议

### 如果你用 Go

| 组件 | 推荐库 | 参考来源 |
|------|--------|---------|
| 命令行框架 | spf13/cobra | feishu-cli |
| HTTP 客户端 | net/http 或 resty | feishu-cli |
| 配置管理 | spf13/viper | 自选 |
| 输出格式化 | tidwall/gjson, gojq | feishu-cli |
| 凭证存储 | zalando/go-keyring | feishu-cli |

**目录结构（参考 feishu-cli）**：
```
mycli/
├── cmd/
│   ├── order/           # 订单相关命令
│   ├── user/            # 用户相关命令
│   └── root.go          # 根命令
├── internal/
│   ├── client/          # API 客户端封装
│   ├── auth/            # 认证
│   ├── output/          # 输出格式化
│   └── config/          # 配置管理
├── skills/              # AI Skills（可选）
└── main.go
```

---

### 如果你用 Rust

| 组件 | 推荐库 | 参考来源 |
|------|--------|---------|
| 命令行框架 | clap (derive) | wecom-cli |
| HTTP 客户端 | reqwest | wecom-cli |
| 配置管理 | config-rs | 自选 |
| 序列化 | serde + serde_json | wecom-cli |
| 凭证存储 | keyring | wecom-cli |

**目录结构（参考 wecom-cli）**：
```
mycli/
├── src/
│   ├── cmd/              # 命令定义
│   ├── client/           # API 客户端
│   ├── auth/             # 认证
│   └── main.rs
└── Cargo.toml
```

---

### 如果你用 TypeScript/Node.js

| 组件 | 推荐库 | 说明 |
|------|--------|------|
| 命令行框架 | commander.js / oclif | oclif 更工程化 |
| HTTP 客户端 | axios / fetch | axios 功能全 |
| 配置管理 | cosmiconfig | 业界标准 |
| 输出格式化 | chalk + table | 美化输出 |
| 凭证存储 | keytar | 系统 Keychain |

**目录结构（混合参考）**：
```
mycli/
├── src/
│   ├── commands/         # 命令（参考 feishu-cli 分层）
│   │   ├── order.ts
│   │   └── user.ts
│   ├── client/           # API 客户端
│   ├── auth/             # 认证
│   └── index.ts
├── skills/               # AI Skills（可选）
└── package.json
```

---

## 关键决策树

```
开始
  │
  ├─→ 团队是否熟悉 Go？
  │   ├─ 是 → 用 Go + feishu-cli 架构
  │   └─ 否 → 继续
  │
  ├─→ 团队是否熟悉 Rust？
  │   ├─ 是 → 用 Rust + wecom-cli 架构
  │   └─ 否 → 继续
  │
  ├─→ 团队是否熟悉 Node/TS？
  │   ├─ 是 → 用 TypeScript + 混合架构
  │   └─ 否 → 继续
  │
  └─→ 选团队最熟悉的语言，参考架构思想即可
```

---

## 最终建议（一句话）

| 场景 | 推荐 |
|------|------|
| **快速上线、内部工具** | 简化版 wecom-cli 架构 |
| **生产级、需要 AI 集成** | feishu-cli 架构（或混合） |
| **已有官方 SDK** | 参考 feishu-cli 的 SDK 封装方式 |
| **只有 HTTP API** | 参考 wecom-cli 的直接 HTTP 方式 |

**我的推荐：先上简化版 wecom-cli 架构（1-2 周），用起来之后再逐步加 feishu-cli 的高级特性！**
