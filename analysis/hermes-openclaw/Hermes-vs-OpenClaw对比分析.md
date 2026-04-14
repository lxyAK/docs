# Hermes Agent vs OpenClaw 架构对比分析

> **对比版本**：Hermes Agent v0.9.0 / OpenClaw v2026.4.11
> **分析日期**：2026-04-14
> **源码路径**：
> - Hermes：`D:\CSBR\opensource\hermes-agent-main`
> - OpenClaw：`D:\CSBR\opensource\openclaw-main`

---

## 目录

1. [项目定位与核心理念](#1-项目定位与核心理念)
2. [技术栈与工程规模](#2-技术栈与工程规模)
3. [整体架构对比](#3-整体架构对比)
4. [Agent 核心引擎](#4-agent-核心引擎)
5. [消息渠道与多平台适配](#5-消息渠道与多平台适配)
6. [工具系统与技能扩展](#6-工具系统与技能扩展)
7. [内存与上下文管理](#7-内存与上下文管理)
8. [安全策略](#8-安全策略)
9. [插件与扩展生态](#9-插件与扩展生态)
10. [部署与运维](#10-部署与运维)
11. [AI 模型接入](#11-ai-模型接入)
12. [多 Agent 配置与多通道消息互通](#12-多-agent-配置与多通道消息互通)
13. [总结与适用场景](#13-总结与适用场景)

---

## 1. 项目定位与核心理念

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **定位** | 自我进化的 AI Agent 平台 | 多渠道 AI 网关 / Agent 平台 |
| **核心理念** | 闭环学习——Agent 从经验中创建技能，使用中优化技能，搜索历史对话 | 网关优先——统一连接消息平台与 AI 模型，Plugin-first 架构 |
| **开发团队** | Nous Research（开源社区） | OpenClaw（开源项目） |
| **许可证** | MIT | 开源 |
| **一句话概括** | "一个会学习的 Agent" | "一个连接一切的 AI 网关" |

**核心差异**：Hermes 的差异化在于**自我学习能力**（自动创建技能、RL 训练环境），而 OpenClaw 的差异化在于**连接广度**（30+ 消息平台、20+ AI 模型提供商）。

---

## 2. 技术栈与工程规模

### 2.1 语言与运行时

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **主语言** | Python 3.11+ | TypeScript (ESM, strict mode) |
| **运行时** | CPython | Node.js >= 22.14.0（推荐 Node 24） |
| **包管理** | pip / uv（pyproject.toml） | pnpm 10.32.1（monorepo workspace） |
| **构建工具** | 无编译（解释型） | tsdown 0.21.7（esbuild/rollup） |
| **测试框架** | Pytest（~3000 测试） | Vitest 4.1.4 |
| **代码检查** | Ruff | oxlint + oxfmt |

### 2.2 代码规模

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **核心源码** | ~389K 行 Python | ~657K 行 TypeScript（src/）+ 扩展 |
| **文件数量** | 核心 ~200+ Python 文件 | 核心 ~3800 TS 文件 + ~2650 扩展 TS 文件 |
| **最大单文件** | `run_agent.py`：10,865 行 | `config/schema.ts`：27,117 行（自动生成） |
| **入口点** | 3 个 CLI 入口 | 1 个 CLI 入口 + 子命令分发 |
| **仓库类型** | 单仓库 | Monorepo（pnpm workspace） |

**分析**：Hermes 以 Python 的简洁换取了更高的代码密度（核心 Agent 逻辑集中在少数大文件），OpenClaw 以 TypeScript 的类型安全换取了更模块化的代码组织。两者都有"巨型文件"问题，但 Hermes 的 `run_agent.py`（10,865 行，单类包含完整 Agent 循环）比 OpenClaw 的 `chat.ts`（2,384 行）更为集中。

---

## 3. 整体架构对比

### 3.1 架构分层

```
┌─────────────────────────────────────────────────────────────┐
│                    Hermes Agent 架构                         │
├─────────────────────────────────────────────────────────────┤
│  用户界面层：CLI (TUI) / Gateway (多平台) / Web / ACP         │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│  Agent 核心：run_agent.py (AIAgent 类，单文件)                │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│  工具执行：50+ 工具 (导入时自注册) + MCP 客户端               │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│  基础设施：Config / Cron / Plugins / RL Training             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     OpenClaw 架构                            │
├─────────────────────────────────────────────────────────────┤
│  渠道插件层：111 个扩展目录，25+ 消息平台适配                  │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│  Gateway 控制平面：WebSocket/HTTP 服务器，60+ RPC 方法        │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│  Agent 引擎：pi-embedded-runner (内嵌 LLM 循环)              │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│  扩展系统：Extensions / Plugin SDK / Skills (53 内置)        │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│  基础设施：Config (Zod) / Sessions / Secrets / Daemon        │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 数据流对比

**Hermes 数据流**：
```
平台消息 -> GatewayRunner -> BasePlatformAdapter
  -> Hooks.pre_process -> DM 配对检查 -> Session 查找
  -> AIAgent.run_conversation() [同步循环]
  -> Hooks.post_process -> 平台发送
```

**OpenClaw 数据流**：
```
平台消息 -> Channel Plugin -> Gateway chat.ts
  -> 权限检查 -> Session 查找/创建
  -> auto-reply 引擎 -> pi-embedded-runner [异步流]
  -> 回复分发 -> 去重 -> 线程管理 -> Channel Plugin 发送
```

**关键差异**：Hermes 的 Agent 循环是**同步的**（`run_conversation` 是一个 ~2300 行的同步方法），OpenClaw 的 Agent 循环是**异步的**（基于 Node.js 事件循环）。这意味着 OpenClaw 在高并发场景下天然更适合，而 Hermes 的同步模型更简单直接。

---

## 4. Agent 核心引擎

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **核心文件** | `run_agent.py`（10,865 行） | `agents/pi-embedded-runner/run.ts`（1,803 行）+ `run/attempt.ts`（2,421 行） |
| **Agent 循环** | 同步 while 循环，~2300 行内联在一个方法中 | 异步流处理，分阶段组织 |
| **上下文压缩策略** | 保护前 3 + 保护后 20 + 中间由辅助模型摘要 | 基于 Session 的上下文管理，支持压缩 |
| **智能模型路由** | 关键词复杂度检测，简单查询走廉价模型，复杂查询走强模型 | 模型回退链（fallback chain） |
| **故障转移** | `classify_api_error()` + `jittered_backoff()`，内联在对话循环中 | 内置于 provider transport 层 |
| **System Prompt 组装** | 9 阶段管道：身份→平台提示→SOUL.md→技能索引→上下文文件→环境提示→记忆→工具指导→模型指导 | 基于 Agent 配置的系统提示组装 |
| **子 Agent** | Delegate 工具：隔离上下文，受限工具集，最大深度 2，最大并发 3 | sub-agent 管理：层级工具限制 |
| **流式输出** | 支持（streaming tool output） | 支持（WebSocket 流式推送） |

**核心设计哲学差异**：

- **Hermes** 倾向于"大泥球"风格——将 Agent 的完整逻辑（LLM 调用、工具分发、上下文压缩、故障转移）集中在单一方法中，代码可读性较低但执行路径清晰，适合深度调试。
- **OpenClaw** 倾向于分层模块化——将 Agent 运行拆分为 runner、attempt、provider 等独立模块，代码组织更清晰但需要跨文件追踪执行流。

### 4.1 工具调用解析器

这是一个 Hermes 独有的重要特性：

| 特性 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **解析器数量** | 11 个模型专用解析器 | 依赖 provider 标准化 |
| **支持格式** | Hermes/Llama/Qwen/Qwen3-Coder/DeepSeek V3/V3.1/GLM-4.5/4.7/Kimi K2/Mistral/LongCat | OpenAI 标准格式 |
| **设计目的** | 兼容各种非标准 tool call 输出格式 | 依赖 provider SDK 统一 |

Hermes 的多解析器设计体现了其"兼容一切模型"的理念，尤其是对中国模型（GLM、Kimi、DeepSeek）的深度支持。

---

## 5. 消息渠道与多平台适配

### 5.1 平台覆盖

| 类别 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **平台数量** | 17+ 适配器 | 25+ 渠道插件（111 个扩展目录） |
| **中国平台** | 飞书、企业微信、微信、钉钉 | 飞书、钉钉 |
| **国际 IM** | Telegram、Discord、Slack、WhatsApp、Signal、Matrix、Mattermost | Telegram、Discord、Slack、WhatsApp、Signal、Matrix、IRC、Line、Google Chat、Nostr、Twitch、Zalo |
| **特殊渠道** | Email、SMS、HomeAssistant、API Server、Webhook、BlueBubbles (iMessage) | MS Teams、Web API、Companion Apps |
| **移动端** | 无原生 App | Android / iOS / macOS 客户端 |

### 5.2 平台适配器架构

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **适配器基类** | `BasePlatformAdapter`（82K） | Channel Plugin SDK（100+ 导出） |
| **最大适配器** | 飞书（164K） | 各渠道独立扩展包 |
| **消息标准化** | 适配器内处理，UTF-16 长度计算 | 插件内标准化，线程/去重 |
| **加载方式** | `asyncio.gather` 并行启动所有平台 | Plugin Loader 发现并加载 |
| **DM 配对** | 支持（CLI 确认配对码） | 支持（`openclaw pairing approve`） |

**分析**：Hermes 在中国平台覆盖上更有优势（微信、企业微信、钉钉、飞书），而 OpenClaw 在国际平台广度上领先。OpenClaw 的插件化架构使得添加新平台更加标准化，Hermes 的适配器虽然功能完整但耦合度更高。

---

## 6. 工具系统与技能扩展

### 6.1 工具系统

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **工具注册方式** | 导入时自注册（`registry.register()`） | Plugin SDK 声明式注册 |
| **工具数量** | 50+ 工具文件 | 内置工具 + 扩展工具 + MCP |
| **终端执行** | 6 种后端：local/Docker/Modal/SSH/Daytona/Singularity | 内置 exec 工具 + Docker sandbox |
| **浏览器** | CamoFox 反指纹浏览器 | Playwright 浏览器自动化 |
| **MCP 角色** | MCP **客户端**（连接外部 MCP 服务器） | MCP **服务器**（暴露自身工具） |
| **ACP 支持** | ACP **服务器**（IDE 集成） | 无 |

### 6.2 技能系统

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **技能格式** | SKILL.md（YAML frontmatter + Markdown） | SKILL.md（`<description>`、`<available_tools>`、`<when>` 区块） |
| **内置技能** | 27 个类别 | 53 个内置技能 |
| **技能市场** | Skills Hub（agentskills.io 兼容），含安全扫描 | 无独立市场 |
| **自动创建** | Agent 从复杂任务中自动创建技能 | 无自动创建 |
| **安全扫描** | 安装前扫描恶意代码、网络外泄、危险函数调用 | Skill Scanner（静态分析） |

**核心差异**：Hermes 的技能系统是**闭环的**——Agent 不仅能使用技能，还能创建和优化技能，这正是"自我进化"理念的体现。OpenClaw 的技能系统更偏向**声明式触发**——用 Markdown 定义何时激活哪些工具。

---

## 7. 内存与上下文管理

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **内置记忆** | MEMORY.md（声明式长期记忆）+ USER.md（用户画像） | Memory Host SDK（向量嵌入 + SQLite） |
| **搜索能力** | FTS5 全文搜索所有历史会话（SQLite） | SQLite-vec 向量搜索 + LanceDB |
| **外部记忆后端** | 8 个：Mem0、Supermemory、Holographic、RetainDB、Byterover、Honcho、Hindsight、OpenViking | 无独立外部后端 |
| **上下文压缩** | 75% token 阈值触发，摘要目标比 0.20，保护前 3 + 保护后 20 | Session 上下文压缩，长对话支持 |
| **上下文引擎** | 可插拔（LCM DAG 压缩等） | 内置 Context Engine |
| **记忆管理** | MemoryManager 编排内置 + 最多 1 个外部提供者 | Memory Host SDK 统一管理 |

**分析**：Hermes 在记忆系统上的投入更深——支持 8 种外部记忆后端，FTS5 全文搜索，以及可插拔的上下文引擎。OpenClaw 则使用向量数据库（SQLite-vec / LanceDB）进行语义搜索，在向量检索场景下可能更优。

---

## 8. 安全策略

### 8.1 安全层级对比

| 安全层 | Hermes Agent | OpenClaw |
|--------|-------------|----------|
| **输入防护** | 10+ 正则检测 Prompt 注入 + 不可见 Unicode 检测（零宽空格、Bidi 控制、BOM） | 上下文可见性控制 |
| **执行防护** | 命令审批（allow/ask/deny）+ 6 种沙箱后端 | 工具策略管道（4 层）+ 沙箱隔离 |
| **输出防护** | 路径安全、凭证保护、结果过滤 | 消息去重、线程管理 |
| **DM 策略** | 配对码机制 | 4 种策略：disabled/allowlist/pairing/open |
| **工具限制** | 子 Agent 受限工具集 | 7 步策略管道：全局→Agent→Session→每次调用 |
| **沙箱后端** | Docker（默认禁网）/ Modal / Daytona / SSH / Singularity / local | Docker / SSH |
| **凭证管理** | 加密存储 + 多密钥轮换池 + 健康检查 | 加密存储 + 系统密钥链集成 |
| **技能安全** | 安装前扫描（恶意代码、网络外泄、危险函数） | Skill Scanner 静态分析（危险执行、动态代码、加密挖掘、环境采集、混淆代码） |
| **网关 HTTP 拒绝列表** | 无 | 默认阻止 exec/spawn/shell/fs_write/fs_delete/apply_patch 等 |
| **URL 安全** | 恶意域名黑名单 + 私有 IP 检测（SSRF） | 无特殊处理 |
| **安全审计** | 无内置命令 | `openclaw doctor security`（quick/deep/fs 三级） |

**总结**：两者都实现了纵深防御，但侧重点不同：

- **Hermes** 更注重**输入侧防护**（Prompt 注入检测、Unicode 隐写检测、URL 安全）和**执行环境隔离**（6 种沙箱后端，覆盖从本地到 HPC 场景）
- **OpenClaw** 更注重**工具权限控制**（7 步策略管道、HTTP 拒绝列表、工具 Profile）和**运维安全**（内置安全审计命令、系统密钥链集成）

---

## 9. 插件与扩展生态

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **插件类型** | memory（8 个后端）、context_engine | Channel、Provider、Capability、Tool |
| **插件发现** | 目录扫描 `plugins/<type>/<name>/` | Plugin Loader（2,212 行），完整生命周期管理 |
| **插件 SDK** | 抽象基类（ABC） | `@openclaw/plugin-sdk`（100+ 子路径导出） |
| **扩展数量** | ~10 个插件 | 111 个扩展目录 |
| **技能数量** | 27 类内置 + 可选技能（大量） | 53 个内置 |
| **Plugin Manifest** | `plugin.yaml` | `openclaw.plugin.json` |
| **插件市场** | Skills Hub（agentskills.io） | 无 |
| **三方生态** | ACP（IDE 集成）、MCP（外部工具） | MCP 服务器、Companion Apps |

**分析**：OpenClaw 的插件生态更成熟、更标准化——有专门的 Plugin SDK、完整的生命周期管理、4 种插件类型。Hermes 的扩展更灵活但更松散——依赖 ABC 约定而非 SDK 框架。

---

## 10. 部署与运维

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **部署模式** | CLI 本地 / Docker 容器 / ACP 服务 | CLI 本地 / Daemon（systemd/launchd/schtasks）/ Docker |
| **Docker** | 单容器，Python 基础镜像 | 多阶段构建，Node 24 bookworm，SHA256 固定 |
| **Docker Compose** | 无官方 compose 文件 | 有（gateway + CLI 服务，健康检查） |
| **守护进程** | 无 | 支持 systemd / launchd / schtasks 注册 |
| **健康检查** | 无内置 | `/healthz`、`/readyz`、`/metrics`（Prometheus） |
| **端口** | 可配置 | 18789（Gateway WS/HTTP）、18790（Bridge） |
| **配置存储** | `~/.hermes/` 目录 | `~/.openclaw/` 目录 |
| **配置验证** | Pydantic + YAML | Zod 4.x + JSON Schema |
| **配置优先级** | CLI > 环境变量 > config.yaml > .env > 默认 | 默认 > 环境变量 > 配置文件 > CLI 标志 |
| **监控** | 无内置 | Prometheus metrics 端点 |
| **配套 App** | 无 | Android / iOS / macOS 客户端 |

**分析**：OpenClaw 在运维方面明显更成熟——Daemon 守护进程、Prometheus 监控、健康检查端点、Docker Compose 编排、多平台配套 App。Hermes 更偏向个人开发者/研究场景，部署选项较少但更轻量。

---

## 11. AI 模型接入

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **模型提供商** | 10+（OpenRouter 200+ 模型、Nous Portal、OpenAI、Anthropic、Google、z.ai/GLM、Kimi、MiniMax、HuggingFace、小米 MiMo、本地 vLLM/SGLang） | ~30 个 provider 插件（OpenAI、Anthropic、Google、DeepSeek、Groq、Mistral、Ollama、AWS Bedrock、NVIDIA、HuggingFace 等） |
| **模型路由** | 智能路由（关键词复杂度检测） | 模型回退链（fallback chain） |
| **认证方式** | API Key + OAuth + 多密钥轮换池 | API Key + OAuth + Auth Profile 轮换 |
| **流式支持** | 所有提供商 | 所有提供商 |
| **中国模型** | 深度支持（GLM、Kimi、MiniMax、MiMo） | DeepSeek |
| **本地模型** | vLLM / SGLang | Ollama |
| **特殊能力** | 11 种模型专用 tool call 解析器 | 标准 OpenAI 格式 |

**分析**：Hermes 在模型接入上更**广**（通过 OpenRouter 单一接口接入 200+ 模型）且对中国模型支持更深。OpenClaw 在 provider 插件上更**多**（~30 个独立 provider），但每个 provider 需要独立维护。

---

## 12. 多 Agent 配置与多通道消息互通

这是两个项目架构差异最大的维度之一。Hermes 采用**多实例隔离**模式，OpenClaw 采用**单进程多 Agent 路由**模式。

### 12.1 多 Agent 配置方式

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **配置模式** | 多 Profile（多进程多实例） | 单进程多 Agent（配置数组） |
| **配置入口** | `hermes profile create <name>` | `agents.list` 配置数组 |
| **实例隔离** | 每个 Profile 独立 HERMES_HOME、独立数据库、独立凭证 | 共享 Gateway 进程，各自 Agent ID、独立工作空间 |
| **并行运行** | 多个 Profile 可同时运行独立 Gateway 进程 | 单个 Gateway 进程内并发运行，受 `maxConcurrent` 限制 |
| **Agent 人格** | 每个 Profile 独立 SOUL.md、config.yaml | 每个 Agent 独立 `systemPromptOverride`、`identity` |
| **模型配置** | 每个 Profile 独立配置 | 每个 Agent 独立 `model`（支持 primary + fallbacks） |
| **工具集** | 每个 Profile 独立配置 | 每个 Agent 独立 `skills` allowlist、`tools` 配置 |
| **Shell 别名** | 自动创建 `~/.local/bin/<profile_name>` 快捷命令 | 无需，单命令管理所有 Agent |

#### Hermes：Profile 多实例架构

```
~/.hermes/                          ← 默认 Profile (default)
├── config.yaml / .env / SOUL.md
├── state.db                        ← 独立 SQLite
├── sessions/ / memories/ / skills/
└── gateway.pid                     ← 独立 Gateway 进程

~/.hermes/profiles/coder/           ← coder Profile
├── config.yaml / .env / SOUL.md    ← 完全独立的配置
├── state.db
└── ...

~/.hermes/profiles/assistant/       ← assistant Profile
└── ...                             ← 又一套完整配置
```

- 创建方式：`hermes profile create coder --clone`（可从现有 Profile 克隆）
- 激活方式：`hermes -p coder chat` 或 `hermes profile use coder`（sticky default）
- 底层机制：`_apply_profile_override()` 在所有模块导入**之前**解析 `-p` 参数，设置 `HERMES_HOME` 环境变量
- 每个 Profile 拥有独立的 `gateway.pid`，因此多个 Gateway 可并行运行

#### OpenClaw：配置数组多 Agent 架构

```jsonc
// config.json
{
  "agents": {
    "defaults": { "model": "anthropic/claude-sonnet-4-6" },
    "list": [
      {
        "id": "coder",
        "default": true,
        "systemPromptOverride": "你是编程助手...",
        "model": { "primary": "anthropic/claude-opus-4-6", "fallbacks": ["openai/gpt-4o"] },
        "skills": ["code-review", "debug"],
        "workspace": "/home/user/projects"
      },
      {
        "id": "assistant",
        "systemPromptOverride": "你是通用助手...",
        "model": "deepseek/deepseek-chat",
        "skills": ["web-search", "summarize"]
      }
    ]
  }
}
```

- 每个 Agent 拥有独立 ID、工作空间（`~/.openclaw/workspace-{id}/`）、会话存储
- `agents.defaults` 提供全局默认值，各 Agent 可覆盖
- 通过 `bindings` 配置将特定渠道/用户/群组路由到特定 Agent

**对比结论**：Hermes 的多实例方式隔离更彻底（独立进程、独立数据库），但管理成本更高（每个 Profile 独立维护）。OpenClaw 的配置数组方式更集中化（单进程统一管理），资源开销更低，但隔离粒度较弱。

---

### 12.2 Agent 间通信与协作

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **通信模型** | 父子委托（Parent-Child Delegation） | 层级式子 Agent（Main/Orchestrator/Leaf） |
| **最大深度** | 2（子 Agent 不可再委派） | 默认 1，可配置 `maxSpawnDepth` |
| **最大并发** | 3（`max_concurrent_children`，ThreadPoolExecutor） | 5（`maxChildrenPerAgent`，异步并发） |
| **子 Agent 工具限制** | 禁止 delegate/clarify/memory/send_message/execute_code | 层级拒绝列表（always-denied vs leaf-only-denied） |
| **跨模型委派** | 支持（`delegation.provider`/`delegation.model`，子 Agent 可用不同模型） | 支持（`spawnSubagentDirect` 可指定 `model` 参数） |
| **跨 Agent 委派** | 不支持（只能 delegate 给自身实例） | 支持（`allowAgents: ["*"]`，子 Agent 可指定不同 `agentId`） |
| **进度回传** | 子 Agent 工具调用回传到父 Agent 显示 | Push-based 完成通知自动推送到父 Agent |
| **结果回传** | 子 Agent 完成后返回文本结果 | 子 Agent 完成后自动 announce 到父 Agent |
| **运行时控制** | 无主动控制（kill/steer 不明确） | 支持 steer（中途改指令）、kill（终止）、list（查看活跃子 Agent） |
| **ACP 委派** | 支持（可 spawn Claude Code 等外部 Agent） | 支持（ACP binding 将请求委派给外部 Agent 进程） |
| **混合专家模式** | Mixture-of-Agents Tool（多模型并行推理 → 聚合） | 无 |

**关键差异**：

- **Hermes** 的子 Agent 是**进程内线程**（ThreadPoolExecutor），更轻量但受 GIL 限制
- **OpenClaw** 的子 Agent 是**异步任务**，天然适合 I/O 密集型操作
- **OpenClaw 独有**：跨 Agent 委派（子 Agent 可以是配置中的另一个完全不同的 Agent）、运行时控制（steer/kill/list）
- **Hermes 独有**：Mixture-of-Agents（多模型并行推理后聚合，适合复杂推理任务）

---

### 12.3 多通道消息互通

#### 12.3.1 跨通道消息发送

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **发送工具** | `send_message` 工具 | `message` 工具 |
| **支持平台** | 17 个平台 | 25+ 个平台 |
| **目标寻址** | 平台名 / `平台:频道名` / `平台:chat_id:thread_id` | `channel` + `target` + `accountId` |
| **Home Channel** | 每个平台可配置默认频道（`TELEGRAM_HOME_CHANNEL`） | 渠道绑定（`bindings` 路由） |
| **消息镜像** | 发送后自动镜像到目标 Session 的对话记录 | 回复分发器处理去重和线程管理 |
| **触发方式** | 仅 Agent 主动调用（无自动转发） | Agent 调用 + 自动回复引擎 |

#### 12.3.2 通道→Agent 路由

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **路由方式** | 固定路由到当前 Profile 的唯一 Agent | 9 层 Binding 匹配：peer→parent→wildcard→guild+roles→guild→team→account→channel→default |
| **路由粒度** | 平台级（整个平台消息由一个 Agent 处理） | 精细到用户/群组/频道/Guild/角色级别 |
| **群组路由** | 按平台 + chat_type + chat_id 路由 | 支持 Discord Role-based 路由、Slack Team-based 路由 |
| **广播模式** | 无（单 Agent 单次回复） | 支持 Broadcast（一条消息触发多个 Agent 同时处理） |

**OpenClaw 路由示例**：

```jsonc
// 将 Telegram 私聊路由到 coder Agent
{ "agentId": "coder", "match": { "channel": "telegram", "peer": { "kind": "dm" } } }

// 将 Discord 特定 Guild 路由到 assistant Agent
{ "agentId": "assistant", "match": { "channel": "discord", "guildId": "123456" } }

// 将 Slack 特定团队中有 admin 角色的用户路由到 admin Agent
{ "agentId": "admin", "match": { "channel": "slack", "teamId": "T789", "roles": ["admin"] } }

// 将所有群组消息路由到 group-handler Agent
{ "agentId": "group-handler", "match": { "channel": "*", "peer": { "kind": "group" } } }
```

#### 12.3.3 跨通道会话管理

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **Session Key 格式** | `agent:main:{platform}:{chat_type}:{chat_id}[:{user_id}]` | `agent:{agentId}:{scope}:{peerId}` |
| **默认隔离** | 按平台 + 聊天隔离，不可跨平台共享 | `dmScope` 可选：`main`（合并）/ `per-peer` / `per-channel-peer` / `per-account-channel-peer` |
| **跨通道统一** | 不支持（Session Key 始终包含 platform 名） | 支持 `identityLinks` 将同一用户在不同平台映射到同一 Session |
| **群组会话** | 默认 per-user 隔离（`group_sessions_per_user=True`） | 按 channel + peerKind + peerId 隔离 |
| **线程共享** | 默认所有用户共享线程 Session（标记为"Multi-user thread"） | 支持线程回复（`thread-reply`） |

**OpenClaw identityLinks 示例**（跨通道会话统一）：

```jsonc
// 同一用户在 Telegram/Discord/WhatsApp 上共享同一个 Session
"identityLinks": {
  "alice": ["telegram:123456", "discord:789012", "whatsapp:654321"]
}
```

Hermes **不支持**跨通道 Session 共享——同一个用户在 Telegram 和 Discord 上与同一个 Hermes Profile 对话，会产生两个完全独立的 Session。

#### 12.3.4 自动消息桥接/转发

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **被动桥接** | 不支持（无自动转发机制） | 不支持（无自动桥接） |
| **主动发送** | Agent 通过 `send_message` 工具主动跨平台发送 | Agent 通过 `message` 工具主动跨平台发送 |
| **定时投递** | Cron Job 可指定 `deliver_to` 跨平台投递 | Cron 系统支持跨渠道定时任务 |
| **Hub-Spoke 模型** | 是（Hermes 是中枢，各平台是辐射端） | 是（Gateway 是中枢，Channel Plugin 是辐射端） |

**共同局限**：两者都不支持"被动桥接"——即自动将 A 平台收到的消息转发到 B 平台。所有跨通道通信都是 **Agent 主动行为**（调用发送工具）或**定时触发**（Cron Job）。

---

### 12.4 配套客户端与多节点

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **移动客户端** | 无 | Android / iOS 原生 App |
| **桌面客户端** | 无 | macOS 菜单栏 App |
| **多节点同时连接** | 不适用 | 支持（Node Registry 管理多个连接节点） |
| **节点能力** | — | 摄像头、定位、SMS、通讯录、通知、语音唤醒等 |
| **离线任务队列** | 无 | 支持（节点离线时入队，重连后自动 drain） |
| **节点调用** | — | Gateway 可 `invoke` 命令到特定节点并等待结果 |

---

### 12.5 多 Agent 多通道互通能力总结

| 能力 | Hermes Agent | OpenClaw | 优势方 |
|------|-------------|----------|--------|
| 多 Agent 配置便捷性 | ★★★☆☆（多 Profile 管理） | ★★★★★（配置数组） | OpenClaw |
| Agent 间隔离彻底性 | ★★★★★（独立进程/数据库） | ★★★☆☆（共享进程） | Hermes |
| 子 Agent 并发能力 | ★★★☆☆（ThreadPool，受 GIL） | ★★★★★（异步，5 并发） | OpenClaw |
| 子 Agent 运行时控制 | ★★☆☆☆ | ★★★★★（steer/kill/list） | OpenClaw |
| 跨 Agent 委派 | ★☆☆☆☆（仅自身实例） | ★★★★☆（可指定不同 AgentId） | OpenClaw |
| 路由精细度 | ★★☆☆☆（平台级） | ★★★★★（9 层匹配，Role/Guild 级） | OpenClaw |
| 跨通道 Session 统一 | ★☆☆☆☆（不支持） | ★★★★☆（identityLinks） | OpenClaw |
| 消息跨平台发送 | ★★★★☆（17 平台） | ★★★★★（25+ 平台） | OpenClaw |
| 混合专家推理 | ★★★★★（Mixture-of-Agents） | ☆☆☆☆☆ | Hermes |
| 配套客户端 | ☆☆☆☆☆ | ★★★★★（Android/iOS/macOS） | OpenClaw |
| 多模型子 Agent | ★★★★☆（delegation 配置） | ★★★★☆（spawn 时指定 model） | 平手 |

**总体结论**：在多 Agent 多通道互通这个维度上，**OpenClaw 明显领先**——单进程多 Agent 配置、9 层路由匹配、跨通道 Session 统一、子 Agent 运行时控制等能力都是 Hermes 所不具备的。Hermes 的优势在于实例隔离更彻底（适合需要完全独立运行不同人格 Agent 的场景）以及独有 Mixture-of-Agents 多模型推理。

---

## 13. 总结与适用场景

### 13.1 核心差异矩阵

| 能力维度 | Hermes Agent | OpenClaw | 优势方 |
|----------|-------------|----------|--------|
| 自我学习与进化 | ★★★★★ | ★★☆☆☆ | Hermes |
| 中国消息平台覆盖 | ★★★★☆ | ★★☆☆☆ | Hermes |
| 国际消息平台覆盖 | ★★★☆☆ | ★★★★★ | OpenClaw |
| 工具系统深度 | ★★★★★ | ★★★★☆ | Hermes |
| 插件生态标准化 | ★★★☆☆ | ★★★★★ | OpenClaw |
| 安全纵深防御 | ★★★★☆ | ★★★★☆ | 平手 |
| 记忆系统深度 | ★★★★★ | ★★★☆☆ | Hermes |
| 运维成熟度 | ★★☆☆☆ | ★★★★★ | OpenClaw |
| AI 模型兼容性 | ★★★★★ | ★★★★☆ | Hermes |
| 代码架构清晰度 | ★★★☆☆ | ★★★★☆ | OpenClaw |
| 部署便捷性 | ★★★☆☆ | ★★★★★ | OpenClaw |
| RL 训练环境 | ★★★★★ | ☆☆☆☆☆ | Hermes |
| 配套客户端 | ★☆☆☆☆ | ★★★★★ | OpenClaw |
| **多 Agent 配置** | ★★★☆☆ | ★★★★★ | **OpenClaw** |
| **多通道消息互通** | ★★☆☆☆ | ★★★★★ | **OpenClaw** |
| **子 Agent 协作控制** | ★★★☆☆ | ★★★★★ | **OpenClaw** |

### 13.2 适用场景推荐

**选择 Hermes Agent 的场景**：
- 需要 Agent 具备自我学习能力（自动创建/优化技能）
- 需要深度接入中国消息平台（微信、企业微信、钉钉、飞书）
- 需要兼容大量非标准 AI 模型（尤其是中国模型：GLM、Kimi、MiniMax、MiMo）
- 需要 RL 训练环境（Atropos、SWE-bench）
- 个人研究或实验场景，偏好 Python 生态
- 需要 ACP 协议集成（IDE 插件）

**选择 OpenClaw 的场景**：
- 需要接入大量国际消息平台（Discord、Slack、Telegram、WhatsApp、Matrix 等 25+ 平台）
- 需要标准化插件生态（Plugin SDK、4 种插件类型、完整生命周期）
- 需要生产级运维能力（Daemon、健康检查、Prometheus、Docker Compose）
- 需要 macOS/iOS/Android 配套客户端
- 团队协作场景，偏好 TypeScript 生态
- 需要内置安全审计命令（`doctor security`）

### 13.3 架构风格总结

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **架构风格** | 大内核 + 薄插件（Monolithic Core） | 小内核 + 厚插件（Microkernel） |
| **设计哲学** | 功能完整性优先 | 扩展性优先 |
| **演进方向** | Agent 能力深化（RL、自动技能） | 平台连接广化（更多渠道、更多 provider） |

两者代表了 AI Agent 平台的两种典型架构路径：**"全能 Agent"**（Hermes）vs **"万能网关"**（OpenClaw）。选择哪个取决于核心需求是"Agent 本身够强"还是"连接足够多"。
