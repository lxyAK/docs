# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

纯中文技术文档仓库，无源代码、无构建系统。文档围绕以下系统展开：

- **OpenClaw** — AI 网关/Agent 平台，连接飞书、企业微信、钉钉、Telegram 等消息渠道与 AI Agent
- **CLI 工具** — feishu-cli (Go)、wecom-cli (Rust)、bayiyuan-cli (规划中)，供 OpenClaw Agent 调用获取业务数据
- **Coze Studio** — 开源 AI Agent 开发平台的部署指南
- **DeerFlow** — 研究自动化工具，与 Professor Agent 集成
- **bayiyuan-project** — 医院患者管理系统，与 OpenClaw 集成

远程仓库：`https://gitee.com/liuxiaoyou/docs.git`

## 目录结构约定

文档按四个固定类别组织：

| 目录 | 用途 | 文件命名后缀示例 |
|------|------|-----------------|
| `analysis/` | 技术深度分析文档 | `深度分析`、`可行性分析`、`对比分析` |
| `architecture/` | 架构设计/实施方案 | `实施方案`、`架构方案`、`完整指南` |
| `deployment/` | 部署指南 | `部署指南` |
| `guides/` | 使用指南 | `使用指南`、`使用说明` |

## 核心架构概念

文档描述的系统采用分层架构：

1. **消息渠道层**：飞书、企业微信、钉钉、Telegram
2. **AI 网关层**：OpenClaw — 多 Agent 平台，工作空间隔离、记忆管理、工具执行
3. **CLI 工具层**：通过 `SKILL.md + exec` 模式工作 — OpenClaw 的 LLM 读取 SKILL.md frontmatter 匹配用户请求，加载完整内容获取命令格式/示例，然后通过 shell 执行 CLI 工具
4. **后端服务层**：医院管理微服务、Coze Studio 后端等

**SKILL.md 模式**是贯穿多个文档的核心设计：CLI 工具通过 SKILL.md 描述文件向 Agent 暴露能力，Agent 匹配后执行 CLI，CLI 调用 HTTP API 并返回精简结果（~200 tokens），LLM 生成自然语言响应。

## 文档编写规范

- 所有文档使用中文编写，Markdown 格式
- 文件名使用中文，需体现主题和文档类型
- Commit 遵循 Conventional Commits：`docs(scope): 简短中文描述`
- 文档中引用外部 GitHub 仓库时使用完整 URL

## 文档间关联

分析文档（analysis/）为架构文档（architecture/）提供技术依据，架构文档指导部署（deployment/）和使用（guides/）。跨文档引用时使用相对路径链接。
