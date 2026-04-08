
# Professor Agent 与 DeerFlow 集成使用指南

## 🎯 两种使用方式

### 方式一：直接使用 DeerFlow Web 界面（推荐）

**最简单、最强大的方式**

```
访问：http://localhost:2026
```

**优势：**
- ✅ 完整的 DeerFlow 功能
- ✅ 17 个技能可用
- ✅ 可视化交互界面
- ✅ 支持文件上传、记忆管理等

---

### 方式二：通过 Professor Agent 代码调用

**适合自动化场景**

#### 1. 检查 DeerFlow 状态

```javascript
const { getHelper } = require('./tools/professor_deerflow.js');

const helper = getHelper();
const status = await helper.status();
console.log(status);
```

#### 2. 列出研究技能

```javascript
const skills = await helper.listResearchSkills();
console.log('可用研究技能:', skills);
```

#### 3. 获取研究建议

```javascript
const suggestion = helper.suggestResearchTask('AI Agent 技术架构');
console.log(suggestion);
```

---

## 📋 可用工具文件

| 文件 | 说明 |
|------|------|
| `tools/deerflow_client.js` | DeerFlow API 底层客户端 |
| `tools/professor_deerflow.js` | Professor Agent 专用封装 |
| `tools/test_deerflow.js` | API 测试脚本 |
| `tools/deerflow_integration_example.js` | 完整集成示例 |

---

## 🚀 快速开始

### 第一步：确认 DeerFlow 运行

```bash
# 检查状态
curl -I http://localhost:2026

# 如果没运行，启动它
cd /root/.openclaw/workspace-professor/opensource/deer-flow
export PATH=/usr/sbin:$PATH
make dev-daemon
```

### 第二步：使用

**选项 A - Web 界面（推荐）**
- 浏览器打开：http://localhost:2026
- 直接开始深度研究

**选项 B - 代码调用**
- 在你的代码中引入 `professor_deerflow.js`
- 调用 helper 函数

---

## 🛑 停止 DeerFlow

```bash
cd /root/.openclaw/workspace-professor/opensource/deer-flow
make stop
```

---

## 📊 当前 DeerFlow 能力

- **模型**：Doubao-Seed-2.0-Code
- **技能**：17 个（deep-research、consulting-analysis、github-deep-research 等）
- **访问地址**：http://localhost:2026

---

## 💡 典型使用场景

1. **深度技术研究** → 用 DeerFlow Web 界面 + deep-research 技能
2. **GitHub 仓库分析** → 用 DeerFlow Web 界面 + github-deep-research 技能
3. **数据文件分析** → 用 DeerFlow Web 界面 + data-analysis 技能
4. **咨询报告生成** → 用 DeerFlow Web 界面 + consulting-analysis 技能

---

**最后更新**：2026-03-26

