# Hermes Agent 源码深度分析

> 版本：main (2026-04-14) | 分析日期：2026-04-14
> 仓库：https://github.com/NousResearch/hermes-agent
> 技术栈：Python 3.11+ | 框架：OpenAI SDK + Fire CLI | 测试：pytest
> 许可证：MIT

## 系列文档

| 编号 | 文档 | 内容 |
|------|------|------|
| 00 | [总体架构概览](00-总体架构概览.md) | 项目定位、代码规模、顶层架构、与 OpenClaw 对比 |
| 01 | [核心引擎详解](01-核心引擎详解.md) | run_agent.py 主循环、Agent Loop、工具注册、上下文压缩 |
| 02 | [工具系统与 Skills](02-工具系统与Skills.md) | ToolRegistry、40+ 工具、Skill 创建/管理、Skills Hub |
| 03 | [Gateway 与多平台适配](03-Gateway与多平台适配.md) | 消息网关、10+ 平台适配器、会话管理、消息路由 |
| 04 | [记忆与上下文管理](04-记忆与上下文管理.md) | MemoryManager、插件化记忆、ContextCompressor、上下文引擎 |
| 05 | [CLI 与配置体系](05-CLI与配置体系.md) | hermes CLI 全命令、配置管理、模型路由、环境变量 |
| 06 | [插件与 ACP 适配](06-插件与ACP适配.md) | 插件系统、ACP 协议适配、MCP 集成、外部记忆后端 |
| 07 | [调度与自动化](07-调度与自动化.md) | Cron 调度器、委托子 Agent、RL 训练环境、批量运行 |
| 08 | [安全策略分析](08-安全策略分析.md) | Prompt 注入防护、命令审批、沙箱隔离、路径安全 |
| 09 | [多Agent协作机制深度分析](09-多Agent协作机制深度分析.md) | Delegate Tool 层级委托、MoA 多模型投票、ACP 协议、与 CrewAI/AutoGen/LangGraph 对比 |
| 10 | [Profile多实例隔离机制深度分析](10-Profile多实例隔离机制深度分析.md) | HERMES_HOME 重定向、Profile CRUD、Wrapper 别名、Gateway Profile 感知、子进程隔离、导出/导入 |
