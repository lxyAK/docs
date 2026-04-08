# 你的 CLI 定制架构方案

## 深度推理：基于你的场景分析

| 你的输入 | 我的分析 |
|---------|---------|
| **API 数量**：约 20 个 | 中等规模，不需要过度设计，但要预留扩展空间 |
| **技术栈**：Java/Node/Python | 推荐用 **Node/TypeScript**，团队熟悉，AI 生态好 |
| **目标用户**：专门给 AI Agent 用 | 这是关键！**优先参考 feishu-cli**，它是为 AI Agent 设计的 |
| **开发模式**：长期迭代 | 架构要考虑可扩展性，不要一次性做太复杂 |

---

## 🎯 结论：优先参考 feishu-cli 架构

**原因**：
1. ✅ **专门为 AI Agent 设计**：20 个 Skills、三层命令、Schema 自省
2. ✅ **Node/TypeScript 可实现**：feishu-cli 的架构思想可以用 Node 复刻
3. ✅ **长期迭代友好**：分层清晰，容易加新功能
4. ✅ **20 个 API 刚刚好**：不需要 wecom-cli 那么简单，但也不用 feishu-cli 那么重

---

## 推荐架构：Node/TypeScript + feishu-cli 思想

### 技术选型

| 组件 | 推荐库 | 说明 |
|------|--------|------|
| **命令行框架** | oclif | 比 commander 更工程化，TypeScript 优先 |
| **HTTP 客户端** | axios | 功能全，TypeScript 类型好 |
| **配置管理** | cosmiconfig | 业界标准 |
| **输出格式化** | chalk + cli-table3 | 美化输出（虽然主要给 AI 用） |
| **凭证存储** | keytar | 系统 Keychain |
| **Schema 验证** | zod | TypeScript 优先的验证库 |

---

### 目录结构（混合 feishu-cli 思想，用 Node 实现）

```
your-cli/
├── src/
│   ├── commands/              # 命令层（参考 feishu-cli 三层）
│   │   ├── shortcuts/         # 第一层：Shortcuts（AI 友好）
│   │   │   ├── order.pending.ts
│   │   │   └── user.active.ts
│   │   ├── api/               # 第二层：API Commands（1:1 对应）
│   │   │   ├── order/
│   │   │   │   ├── list.ts
│   │   │   │   ├── get.ts
│   │   │   │   └── create.ts
│   │   │   └── user/
│   │   │       ├── list.ts
│   │   │       └── get.ts
│   │   └── raw/               # 第三层：Raw API（完全自定义）
│   │       └── call.ts
│   │
│   ├── client/                # API 客户端封装（参考 feishu-cli）
│   │   ├── index.ts           # 主客户端
│   │   ├── auth.ts            # 认证（Token 管理、刷新）
│   │   └── pagination.ts      # 分页处理
│   │
│   ├── skills/                # AI Skills（参考 feishu-cli）
│   │   ├── yourcli-shared/    # 共享（配置、认证）
│   │   ├── yourcli-order/     # 订单 Skill
│   │   └── yourcli-user/      # 用户 Skill
│   │
│   ├── schema/                # Schema 自省（AI 用）
│   │   ├── index.ts
│   │   └── definitions/       # API 定义（JSON/YAML）
│   │       ├── order.json
│   │       └── user.json
│   │
│   ├── output/                # 输出格式化
│   │   ├── json.ts
│   │   ├── ndjson.ts          # Newline-delimited JSON（AI 友好）
│   │   └── table.ts
│   │
│   └── index.ts               # 入口
│
├── bin/
│   └── yourcli.js             # 可执行文件
│
├── package.json
└── tsconfig.json
```

---

## 分阶段实施计划（长期迭代友好）

### Phase 1：MVP（2 周）- 核心功能

**目标**：AI 能用，基础体验

| 功能 | 实现内容 | 参考来源 |
|------|---------|---------|
| **命令框架** | oclif 搭建基础命令 | feishu-cli 思想 |
| **API 封装** | axios 封装你的 20 个 API | wecom-cli 简单直接 |
| **认证** | Token 管理、自动刷新 | feishu-cli |
| **输出** | JSON / NDJSON（AI 友好） | feishu-cli |
| **Schema** | 简单的 API 定义（JSON） | feishu-cli |

**示例命令**：
```bash
# API Command 层
your-cli order list --status pending
your-cli order get --id 123
your-cli order create --data '{"amount":100}'

# Raw API 层
your-cli raw call GET /api/v1/orders
```

---

### Phase 2：AI 优化（2 周）- Agent 友好

**目标**：让 AI 更喜欢用

| 功能 | 实现内容 | 参考来源 |
|------|---------|---------|
| **Shortcuts** | 简化常用命令 | feishu-cli |
| **Dry Run** | 安全预览 | feishu-cli |
| **Schema 自省** | `your-cli schema order.create` | feishu-cli |
| **机器可读输出** | NDJSON 优化 | feishu-cli |
| **错误提示** | AI 友好的错误信息 | feishu-cli |

**示例命令**：
```bash
# Shortcut 层（AI 更喜欢这个）
your-cli order +pending
your-cli order +create --amount 100 --dry-run

# Schema 自省（AI 会用这个了解怎么调用）
your-cli schema order.create
```

---

### Phase 3：Skills 生态（2-3 周）- 结构化

**目标**：像 feishu-cli 一样有结构化 Skills

| 功能 | 实现内容 | 参考来源 |
|------|---------|---------|
| **Skill 定义** | 标准化 Skill 格式 | feishu-cli |
| **Skill 注册** | 自动发现和加载 Skills | feishu-cli |
| **npx skills 集成** | 可选，生态友好 | feishu-cli |

**Skill 目录结构**：
```
skills/
├── yourcli-shared/
│   └── skill.json
├── yourcli-order/
│   ├── skill.json
│   └── tools/
│       ├── list_pending_orders.md
│       └── create_order.md
└── yourcli-user/
    └── skill.json
```

---

### Phase 4+：长期迭代

根据需求添加：
- 多格式输出（Table/CSV）
- 分页自动化
- 更完善的错误处理
- 监控和日志
- 性能优化

---

## 关键设计要点（AI Agent 专用）

### 1. 输出优先用 NDJSON

```typescript
// NDJSON 示例（AI 最喜欢）
{"id":1,"status":"pending","amount":100}
{"id":2,"status":"completed","amount":200}
{"id":3,"status":"pending","amount":150}
```

**原因**：
- ✅ AI 容易解析
- ✅ 流式输出友好
- ✅ 不需要一次性加载全部

---

### 2. Schema 定义要详细

```json
// schema/definitions/order.create.json
{
  "name": "order.create",
  "description": "创建订单",
  "parameters": {
    "type": "object",
    "properties": {
      "amount": {
        "type": "number",
        "description": "订单金额",
        "required": true
      },
      "customer_id": {
        "type": "string",
        "description": "客户 ID",
        "required": true
      }
    }
  },
  "examples": [
    {
      "description": "创建一个 100 元的订单",
      "command": "your-cli order create --amount 100 --customer_id cus_123"
    }
  ]
}
```

---

### 3. 错误提示要给 AI 有用的信息

```typescript
// 不好的错误
"创建失败"

// 好的错误（AI 友好）
{
  "error": "missing_required_parameter",
  "parameter": "customer_id",
  "message": "缺少必填参数 customer_id",
  "hint": "使用 'your-cli schema order.create' 查看参数定义",
  "example": "your-cli order create --amount 100 --customer_id cus_123"
}
```

---

## 最终建议总结

| 决策项 | 我的建议 |
|--------|---------|
| **开发语言** | TypeScript（Node.js） |
| **核心架构** | 参考 feishu-cli（AI Agent 优先） |
| **命令分层** | 三层：Shortcuts → API → Raw |
| **输出格式** | 优先 NDJSON，其次 JSON |
| **Phase 1** | 2 周：基础命令 + API 封装 + Schema |
| **Phase 2** | 2 周：Shortcuts + Dry Run + Schema 自省 |
| **Phase 3** | 2-3 周：Skills 生态 |

**一句话：用 TypeScript + oclif + axios，复刻 feishu-cli 的 AI 友好设计！**
