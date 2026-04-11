# Colleague.Skill 深度分析报告

> **源码仓库**: https://github.com/titanwings/colleague-skill
> **分析日期**: 2026-04-12
> **作者**: 教授 🧑‍💻

---

## 一、项目定位

这是一个 **Meta-Skill（元技能）**——用来生成其他 Skill 的 Skill。核心理念是：**把真实同事的工作能力和性格"蒸馏"成 AI 可执行的 Persona**，实现"赛博永生"。

**一句话**：同事离职/调岗/毕业，把他/她的工作能力和性格打包成 AI Skill，让 AI 能像他/她一样工作。

---

## 二、架构设计 ⭐⭐⭐⭐

### 双轨并行架构

| 轨道 | 输出文件 | 作用 |
|------|---------|------|
| Part A — Work Skill | `work.md` | 技术能力、规范、流程，能干实事 |
| Part B — Persona | `persona.md` | 5 层性格模型，能像真人说话 |

**运行规则**：`Persona 决定态度 → Work Skill 执行任务 → 用 Persona 的风格输出`

这是整个项目最精妙的设计——**能力与性格解耦**，可以独立使用也可以组合。

---

## 三、核心模块分析

### 1. Prompt 系统（7 个 prompt 模板，共 802 行）

| 模板 | 行数 | 功能 |
|------|------|------|
| `intake.md` | 135 | 3 问极简录入，内置 9 家公司职级对照表 |
| `work_analyzer.md` | 181 | 按职位类型差异化提取（后端/前端/算法/产品/设计/数据） |
| `persona_analyzer.md` | 133 | 标签翻译表（15 种个性标签 → 具体行为规则） |
| `persona_builder.md` | 176 | 5 层 Persona 结构生成模板 |
| `work_builder.md` | 101 | Work Skill 生成模板 |
| `merger.md` | 91 | 增量合并逻辑 + 冲突检测 |
| `correction_handler.md` | 85 | 对话纠正处理 |

**核心亮点 — 标签翻译机制**：

把"甩锅高手""PUA 高手"等标签直接翻译成可执行的行为规则，而非停留在形容词层面。

示例：
```
标签：甩锅高手
→ Layer 0 规则：遇到问题第一反应是找外部原因；事前主动模糊自己的责任边界；
  被问责时先说"当时需求没说清楚"或"这块本来不是我的"
```

### 2. 工具链（9 个 Python 脚本，约 4271 行）

| 工具 | 行数 | 功能 |
|------|------|------|
| `feishu_auto_collector.py` | 957 | 飞书全自动采集（群聊+私聊+文档+多维表格） |
| `dingtalk_auto_collector.py` | 787 | 钉钉自动采集（文档+表格+浏览器消息） |
| `slack_auto_collector.py` | 719 | Slack 自动采集（含速率限制处理） |
| `email_parser.py` | 339 | .eml/.mbox 邮件解析 |
| `feishu_browser.py` | 374 | 飞书浏览器方案（Playwright） |
| `feishu_mcp_client.py` | 310 | 飞书 MCP 方案（App Token） |
| `skill_writer.py` | 380 | Skill 文件写入/管理 |
| `feishu_parser.py` | 251 | 飞书 JSON 导出解析 |
| `version_manager.py` | 154 | 版本存档与回滚 |

**技术亮点**：
- 飞书采集区分 `tenant_access_token`（群聊）和 `user_access_token`（私聊）两种模式
- Slack 采集有完善的速率限制处理（`RateLimitedClient` 类）
- 所有工具都支持 `--setup` 交互式配置

### 3. 示例数据（3 个同事示例）

- **佳秀** — AI Lab HRBP，ENFJ，星座达人
- **天一** — 示例人物
- **张三** — 示例人物

---

## 四、技术亮点

### 1. Persona 5 层架构（优先级从高到低）

```
Layer 0 — 硬覆盖层：手动标签翻译为行为规则，任何情况不得违背
Layer 1 — 身份层：姓名、公司、职级、MBTI、企业文化
Layer 2 — 表达风格层：口头禅、句式、emoji、正式程度
Layer 3 — 决策与判断层：优先级、推进/回避触发、说"不"的方式
Layer 4 — 人际行为层：对上级/下级/平级/压力下的不同表现
Layer 5 — Correction 层：对话纠正追加，滚动更新
```

优先级清晰，避免"人设崩塌"。

### 2. 进化机制（3 条路径）

```
追加文件 → 自动分析增量 → merge 进对应文件 → 版本存档
对话纠正 → 识别 correction → patch Correction 层 → 立即生效
版本回滚 → /colleague-rollback {slug} {version} → 恢复历史版本
```

### 3. 多源数据采集

飞书 / 钉钉 / Slack / 邮件 / PDF / 图片 / 文本 / 微信（第三方工具导出），覆盖主流办公场景。

---

## 五、项目结构

```
colleague-skill/
├── SKILL.md              # Skill 入口（中英双语，约 400 行）
├── prompts/              # Prompt 模板（7 个文件，802 行）
├── tools/                # Python 工具（9 个文件，4271 行）
├── colleagues/           # 生成的同事 Skills（3 个示例）
├── docs/PRD.md           # 产品需求文档（434 行）
├── colleague_skill.pdf   # 技术报告
├── requirements.txt      # 依赖
├── INSTALL.md            # 安装指南
├── README.md + 6 种语言版本
└── LICENSE               # MIT
```

**总量**：约 6400 行，体积 4.1MB

---

## 六、待改进 / 可优化点

| 方面 | 现状 | 建议 |
|------|------|------|
| **微信支持** | 标注"目前不稳定"，推荐第三方工具 | 可考虑集成 WeChatMsg 导出格式 |
| **Word/Excel** | 提示用户转 PDF/CSV | `python-docx`/`openpyxl` 已在 requirements，可直接支持 |
| **Correction 层上限** | 50 条后合并归纳 | 合并逻辑无自动化实现，需手动处理 |
| **测试覆盖** | 无测试代码 | 核心解析逻辑建议加单元测试 |
| **并发采集** | 单线程串行 | 大量数据时可考虑异步/多线程 |
| **私聊采集** | OAuth 流程复杂 | 用户体验门槛高，可考虑简化引导 |

---

## 七、代码质量评估

- **结构清晰**：SKILL.md 是"操作系统"，prompt 是"方法论"，tools 是"工具箱"
- **文档完善**：PRD 434 行，README 7 种语言，INSTALL 独立指南
- **标准兼容**：遵循 [AgentSkills](https://agentskills.io) 开放标准，支持 Claude Code 和 OpenClaw
- **License**：MIT，社区友好

---

## 八、总结评分

| 维度 | 评分 | 说明 |
|------|------|------|
| 创意 | ⭐⭐⭐⭐⭐ | "蒸馏同事"概念新颖，实用且有话题性 |
| 架构 | ⭐⭐⭐⭐ | 双轨解耦 + 5 层 Persona，设计精巧 |
| 完成度 | ⭐⭐⭐⭐ | 主流程完整，飞书/钉钉/Slack 全覆盖 |
| 代码质量 | ⭐⭐⭐ | 功能完整但缺少测试和错误处理 |
| 文档 | ⭐⭐⭐⭐⭐ | PRD + 多语言 README + 安装指南，非常完善 |
| 社区潜力 | ⭐⭐⭐⭐⭐ | MIT 协议，社区画廊已有多个贡献 |

---

## 九、结论

Colleague.Skill 是一个**完成度高、设计精巧的 Meta-Skill 项目**。核心创新在于把"人"拆解为可执行的行为规则，双轨架构让它既能干活又能演人。适合作为 Agent Skill 开发的参考范本。

**核心参考价值**：
1. Persona 5 层架构 → 可复用于任何角色扮演类 Skill
2. 标签翻译机制 → 优秀的"模糊描述 → 精确规则"转换范例
3. 进化机制设计 → 追加/纠正/回滚的完整闭环
4. 多源数据采集 → 飞书/钉钉/Slack 的 API 集成参考
