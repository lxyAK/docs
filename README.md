# Docs

技术文档仓库，按项目维度组织。

远程仓库：https://gitee.com/liuxiaoyou/docs.git

## 目录结构

```
docs/
├── analysis/                        # 技术分析文档
│   ├── openclaw/                    #   OpenClaw 平台
│   ├── cli/                         #   CLI 工具集
│   ├── bayiyuan/                    #   八亿元项目
│   ├── deerflow/                    #   DeerFlow
│   ├── taro/                        #   Taro
│   ├── claude-code/                 #   Claude Code
│   └── colleague-skill/             #   Colleague.Skill 开源项目
│
├── architecture/                    # 架构文档
│   ├── cli/                         #   CLI 工具集
│   └── bayiyuan/                    #   八亿元项目
│
├── deployment/                      # 部署文档
│   └── coze-studio/                 #   Coze Studio
│
├── guides/                          # 使用指南
│   ├── deerflow/                    #   DeerFlow
│   ├── openclaw/                    #   OpenClaw
│   └── security/                    #   安全工具
│
└── openclaw-source-analysis/        # OpenClaw 源码深度分析（系列连载）
```

## 文档索引

### analysis/ — 技术分析

| 项目 | 文档 | 简介 |
|------|------|------|
| **openclaw** | [Skill系统使用分析](./analysis/openclaw/Skill系统使用分析.md) | Skill 触发机制与目录组织 |
| | [权限管控分析](./analysis/openclaw/权限管控分析.md) | 权限体系与管控方案 |
| | [用户会话隔离与权限控制方案](./analysis/openclaw/用户会话隔离与权限控制方案.md) | 会话隔离机制 |
| | [终端命令大全](./analysis/openclaw/终端命令大全.md) | OpenClaw CLI 完整命令 |
| **cli** | [CLI业务对接深度分析](./analysis/cli/CLI业务对接深度分析.md) | CLI 与业务系统对接方案 |
| | [CLI技术方案对比](./analysis/cli/CLI技术方案对比.md) | Go/Rust/Python 方案对比 |
| | [自研CLI架构决策分析](./analysis/cli/自研CLI架构决策分析.md) | 自研 vs 现有方案决策 |
| | [企业微信CLI工具认证体系深度分析](./analysis/cli/企业微信CLI工具认证体系深度分析.md) | 企微认证机制 |
| | [飞书CLI工具认证体系深度分析](./analysis/cli/飞书CLI工具认证体系深度分析.md) | 飞书认证机制 |
| **bayiyuan** | [OpenClaw结合应用方案](./analysis/bayiyuan/OpenClaw结合应用方案.md) | 八亿元项目与 OpenClaw 集成 |
| **deerflow** | [OpenClaw集成方案](./analysis/deerflow/OpenClaw集成方案.md) | DeerFlow 集成方案 |
| **taro** | [多端架构重构可行性分析](./analysis/taro/多端架构重构可行性分析.md) | Taro 重构可行性评估 |
| **claude-code** | [Haha-工具系统深度分析](./analysis/claude-code/Haha-工具系统深度分析.md) | Claude Code Haha 工具系统 |
| **colleague-skill** | [深度分析](./analysis/colleague-skill/深度分析.md) | Colleague.Skill 开源项目分析 |

### architecture/ — 架构文档

| 项目 | 文档 | 简介 |
|------|------|------|
| **cli** | [CLI工具完整指南](./architecture/cli/CLI工具完整指南.md) | CLI 工具完整使用指南 |
| | [CLI定制架构方案](./architecture/cli/CLI定制架构方案.md) | 定制化架构设计 |
| **bayiyuan** | [CLI实施方案](./architecture/bayiyuan/CLI实施方案.md) | 八亿元项目 CLI 实施方案 |

### deployment/ — 部署文档

| 项目 | 文档 | 简介 |
|------|------|------|
| **coze-studio** | [仅后端部署指南](./deployment/coze-studio/仅后端部署指南.md) | Coze Studio 后端部署 |
| | [发版部署指南](./deployment/coze-studio/发版部署指南.md) | Coze Studio 完整发版流程 |

### guides/ — 使用指南

| 项目 | 文档 | 简介 |
|------|------|------|
| **deerflow** | [Professor集成使用指南](./guides/deerflow/Professor集成使用指南.md) | DeerFlow 与 Professor Agent 集成 |
| **openclaw** | [知识库使用指南](./guides/openclaw/知识库使用指南.md) | OpenClaw 知识库使用说明 |
| **security** | [供应链攻击检测脚本使用说明](./guides/security/供应链攻击检测脚本使用说明.md) | 安全检测工具使用 |

### openclaw-source-analysis/ — 源码专题

| 文档 | 简介 |
|------|------|
| [README](./openclaw-source-analysis/README.md) | 系列索引与阅读路径 |
| [00-总体架构概览](./openclaw-source-analysis/00-总体架构概览.md) | 项目定位、目录结构、启动链路 |
| [01-核心模块详解](./openclaw-source-analysis/01-核心模块详解.md) | Gateway/Agent/Channel/Session |
| [02-插件与扩展系统](./openclaw-source-analysis/02-插件与扩展系统.md) | Extensions/Plugin SDK/Skills |
| [03-消息流转链路](./openclaw-source-analysis/03-消息流转链路.md) | 消息处理的完整流程 |
| [04-配置与安全体系](./openclaw-source-analysis/04-配置与安全体系.md) | Config/Secrets/Security |
| [05-部署与运维](./openclaw-source-analysis/05-部署与运维.md) | Docker/Daemon/CLI 运维 |
| [06-安全策略深度分析](./openclaw-source-analysis/06-安全策略深度分析.md) | 7 层纵深防御体系 |
