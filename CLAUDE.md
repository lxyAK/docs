# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

纯中文技术文档仓库，无源代码、无构建系统。文档围绕以下系统展开：

- **OpenClaw** — AI 网关/Agent 平台，连接飞书、企业微信、钉钉、Telegram 等消息渠道与 AI Agent
- **CLI 工具** — feishu-cli (Go)、wecom-cli (Rust)、bayiyuan-cli (规划中)，供 OpenClaw Agent 调用获取业务数据
- **Coze Studio** — 开源 AI Agent 开发平台的部署指南
- **DeerFlow** — 研究自动化工具，与 Professor Agent 集成
- **bayiyuan-project** — 医院患者管理系统，与 OpenClaw 集成
- **colleague-skill** — 开源 Meta-Skill 项目，蒸馏同事工作能力与性格

远程仓库：`https://gitee.com/liuxiaoyou/docs.git`

## 目录结构约定

文档按**类型分主目录，项目名分二级目录**组织：

| 主目录 | 用途 | 二级目录示例 |
|--------|------|-------------|
| `analysis/` | 技术深度分析 | `openclaw/`、`cli-tools/`、`bayiyuan/`、`deerflow/`、`taro/`、`claude-code/`、`colleague-skill/` |
| `architecture/` | 架构设计/实施方案 | `cli-tools/`、`bayiyuan/` |
| `deployment/` | 部署指南 | `coze-studio/` |
| `guides/` | 使用指南 | `deerflow/`、`openclaw/`、`security/` |
| `openclaw-source-analysis/` | OpenClaw 源码深度分析系列（独立专题） | — |

### 目录规则

1. **新建文档先确定项目名**：项目名作为二级目录名（小写英文，连字符连接）
2. **新建项目直接加目录**：在对应主目录下创建 `项目名/` 文件夹
3. **文件命名规范**：`[主题][类型后缀].md`，类型后缀如 `分析`、`方案`、`指南`、`说明`
4. **避免空格**：文件名中的空格用连字符替代
5. **源码分析类文档**：必须标注项目名 + Git 仓库地址

## 文档编写规范

- 所有文档使用中文编写，Markdown 格式
- Commit 遵循 Conventional Commits：`docs(项目名): 简短中文描述`
- 文档中引用外部 GitHub 仓库时使用完整 URL
- 跨文档引用使用相对路径链接

## 文档间关联

分析文档（analysis/）为架构文档（architecture/）提供技术依据，架构文档指导部署（deployment/）和使用（guides/）。同一项目的文档通过项目名二级目录自然关联。
