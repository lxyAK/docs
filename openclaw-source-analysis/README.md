# OpenClaw 源码深度分析

> 📚 系统性的 OpenClaw 源码架构分析文档
>
> 分析版本：OpenClaw 2026.4.11 | 分析日期：2026-04-11

---

## 📖 文档索引

| 序号 | 文档 | 内容概览 |
|------|------|---------|
| 00 | [总体架构概览](./00-总体架构概览.md) | 项目定位、目录结构、启动链路、核心数据流 |
| 01 | [核心模块详解](./01-核心模块详解.md) | Gateway/Agent/Channel/Session/Config/Plugins 深入分析 |
| 02 | [插件与扩展系统](./02-插件与扩展系统.md) | Extensions/Plugin SDK/Skills 三层扩展机制 |
| 03 | [消息流转链路](./03-消息流转链路.md) | 从用户消息到回复的完整处理流程 |
| 04 | [配置与安全体系](./04-配置与安全体系.md) | Config/Secrets/Security/Tool Policy |
| 05 | [部署与运维](./05-部署与运维.md) | Docker/Daemon/CLI 运维与监控 |
| 06 | [安全策略深度分析](./06-安全策略深度分析.md) | 7层纵深防御体系、工具策略管线、Skill扫描 |

---

## 🎯 快速上手指南

### 1. 阅读顺序

**初学者路径**：
```
00-总体架构概览 → 01-核心模块详解 → 05-部署与运维
```

**开发者路径**：
```
00 → 01 → 02-插件与扩展系统 → 03-消息流转链路
```

**运维路径**：
```
00 → 05-部署与运维 → 04-配置与安全体系
```

### 2. 核心概念

- **Gateway** — WebSocket/HTTP 网关，消息路由
- **Agent** — LLM 调用、工具执行、流处理
- **Channel** — 消息平台抽象（飞书/TG/Discord）
- **Session** — 会话管理（Transcript、生命周期）
- **Plugin** — 扩展机制（Provider/Channel/工具）

---

## 📊 项目数据

| 指标 | 数值 |
|------|------|
| 核心源码 | ~3800 个 TS 文件，~657K 行 |
| 扩展源码 | ~2650 个 TS 文件 |
| 总文件数 | ~6500+ TS 文件 |
| Channel 插件 | 24 个（飞书、Telegram、Discord 等） |
| Provider 插件 | ~25 个（OpenAI、Anthropic、Google 等） |
| 内置 Skills | 53 个 |

---

## 🔧 技术栈

- **语言**：TypeScript (ESM, strict mode)
- **运行时**：Node.js ≥ 22.14.0
- **包管理**：pnpm
- **构建**：tsdown (esbuild/rollup)
- **测试**：Vitest
- **容器**：Docker + Docker Compose
- **配置**：Zod Schema + JSON
- **存储**：SQLite + 文件系统

---

## 🌐 官方资源

| 资源 | 链接 |
|------|------|
| 文档 | https://docs.openclaw.ai |
| GitHub | https://github.com/openclaw/openclaw |
| Discord | https://discord.com/invite/clawd |
| ClawHub | https://clawhub.com |

---

## 📝 说明

本文档基于 OpenClaw 2026.4.11 版本源码分析，仅用于学习和研究目的。

如有错误或遗漏，欢迎提交 Issue 或 PR。
