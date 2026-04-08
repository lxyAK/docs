# OpenClaw Skill系统使用分析

## Skill触发机制

OpenClaw的Skill系统通过以下方式触发：

1. **任务匹配**：根据用户请求的内容与Skill的description进行匹配
2. **明确加载**：用户明确要求使用某个Skill时
3. **场景触发**：特定的工作场景（如创建文档、技术分析等）

## 目录组织

文档应按照以下结构存放：
- `docs/analysis/` - 技术分析类
- `docs/architecture/` - 架构设计类
- `docs/deployment/` - 部署指南类
- `docs/guides/` - 使用指南类
