# bayiyuan-cli 实施方案

## Context

八医院项目需要将微服务业务数据接入 OpenClaw AI 网关，使飞书/企微用户可以通过自然语言查询业务数据。直接用 MCP 方案 token 消耗大，参考企业微信 wecom-cli 的 **SKILL.md + exec** 模式，开发 CLI 工具供 OpenClaw 调用。

核心链路：`用户(飞书/企微) → OpenClaw → LLM读SKILL.md → exec bayiyuan-cli → HTTP → 微服务API`

MVP 范围：**饮食查询、患者搜索、体重打卡** 3 个命令
API 路由：**同时支持 Gateway 统一入口和 Nacos 直连两种模式**，通过配置切换
技术方案：**同时出 Node.js 和 Java 两个版本**，对比选型

***

## 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户触达层                                │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│   │  飞书机器人 │  │ 企业微信   │  │ 钉钉/其他  │                      │
│   └─────┬────┘  └─────┬────┘  └─────┬────┘                      │
│         └─────────────┼─────────────┘                            │
│                       ▼                                          │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                    OpenClaw (AI 网关)                       │   │
│  │                                                             │   │
│  │  1. 收到用户消息                                              │   │
│  │  2. LLM 读取 SKILL.md frontmatter → 匹配技能                 │   │
│  │  3. 加载完整 SKILL.md body → 获取命令格式和示例                │   │
│  │  4. exec 执行 bayiyuan-cli 命令                              │   │
│  │  5. 拿到精简结果 → LLM 生成自然语言回复                       │   │
│  └────────────────────────┬──────────────────────────────────┘   │
│                           │                                      │
│                      exec (shell)                                │
│                           │                                      │
│  ┌────────────────────────▼──────────────────────────────────┐   │
│  │              bayiyuan-cli (本方案开发的CLI)                  │   │
│  │                                                             │   │
│  │   bayiyuan-cli diet query '{"patient":"张三","days":3}'     │   │
│  │                                                             │   │
│  │   内部处理:                                                  │   │
│  │   ① JSON.parse 参数 + 校验                                  │   │
│  │   ② 姓名→externalUserId 映射                                │   │
│  │   ③ HTTP 请求微服务 API                                      │   │
│  │   ④ 数据精简（去系统字段、格式化摘要）                         │   │
│  │   ⑤ 输出精简文本（~200 tokens）                              │   │
│  └────────────────────────┬──────────────────────────────────┘   │
│                           │                                      │
│                    HTTP (内网)                                    │
│           ┌───────────────┼────────────────┐                     │
│           ▼               ▼                ▼                     │
│  ┌──────────────┐ ┌────────────────┐ ┌──────────────┐           │
│  │ Gateway模式   │ │  直连模式        │ │  Nacos 注册   │           │
│  │ (统一认证)    │ │  (内网直连)      │ │  中心          │           │
│  └──────┬───────┘ └───────┬────────┘ └──────────────┘           │
│         ▼                 ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │                  八医院微服务集群 (Nacos)                    │    │
│  │  ┌──────────────────┐  ┌──────────────┐  ┌────────────┐ │    │
│  │  │ nutritional-service│  │ personel-service│  │ user-service│ │    │
│  │  │  饮食/营养/随访/评估 │  │  人事/薪资/考勤  │  │  用户/企微  │ │    │
│  │  └──────────────────┘  └──────────────┘  └────────────┘ │    │
│  └──────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 一次查询的完整数据流

```
用户(飞书): "查一下张三最近3天的饮食记录"
        │
        ▼
① OpenClaw 看到 SKILL.md frontmatter:
   "bayiyuan-diet-query: 查询患者饮食记录..."  ← 匹配！(~20 tokens)
        │
        ▼
② 加载完整 SKILL.md body，LLM 看到命令格式和示例 (~200 tokens)
        │
        ▼
③ LLM 生成 exec 命令:
   bayiyuan-cli diet query '{"patient":"张三","days":3}'
        │
        ▼
④ CLI 内部处理:
   - JSON.parse → 校验 patient 必填 ✓
   - "张三" → 调 /nutr-user-info 搜索 → externalUserId
   - 构造请求体 → POST /diet-records/page-list
   - API 返回 ~2000 tokens 的完整 JSON
   - formatter 精简为 ~200 tokens 的文本摘要
        │
        ▼
⑤ 输出:
   "张三近3天饮食记录:
    4/5 早餐380kcal 午餐650kcal 晚餐520kcal
    4/6 早餐350kcal 午餐700kcal 晚餐480kcal
    4/7 早餐400kcal 午餐-(未记录)
    日均约1500kcal，低于目标1800kcal"
        │
        ▼
⑥ OpenClaw LLM 基于精简文本生成自然语言回复 → 飞书
```

### Token 消耗对比

| 环节                   | MCP 方案                     | SKILL.md + exec 方案              |
| -------------------- | -------------------------- | ------------------------------- |
| 工具注册 (system prompt) | \~800 tokens (JSON Schema) | **\~60 tokens** (3个frontmatter) |
| 技能详情加载               | 无（已全量注入）                   | **\~200 tokens** (按需加载body)     |
| LLM 推理               | \~200                      | \~200                           |
| 工具返回数据               | \~2000 (原始JSON)            | **\~200** (CLI精简后)              |
| 生成回复                 | \~500                      | \~300                           |
| **单次总计**             | **\~3500**                 | **\~960**                       |
| **节省**               | —                          | **\~73%**                       |

***

## 方案对比：Node.js vs Java

| 维度                    | Node.js (Commander.js) | Java (Picocli)          |
| --------------------- | ---------------------- | ----------------------- |
| **与 Skills 生态兼容性**    | 原生支持 `npx skills add`  | 需手动拷贝 SKILL.md          |
| **冷启动速度**             | \~200ms                | \~1.5s (JVM启动)          |
| **安装部署**              | `npm install -g`       | jar 包 + 需 JRE 17        |
| **产物体积**              | \~5MB (node\_modules)  | \~30MB (fat jar)        |
| **复用现有代码**            | 无法复用 Java VO/DTO       | 可直接引用 csbr-cloud-entity |
| **团队维护成本**            | 需熟悉 Node.js            | 团队现有技术栈                 |
| **OpenClaw exec 速度**  | 快（LLM 等待时间短）           | 慢（每次 exec 都要启动 JVM）     |
| **GraalVM Native 编译** | 不适用                    | 可编译为原生二进制(\~50ms启动)     |

**建议**: Node.js 版本做主方案（exec 响应快），Java 版本做备选（团队更熟悉，后续可用 GraalVM 编译为原生镜像提速）。

***

## 方案 A：Node.js 版本

### 项目结构

```
bayiyuan-cli-node/
├── package.json                      # bin: { "bayiyuan-cli": "./bin/cli.js" }
├── bin/
│   └── cli.js                        # 入口，注册子命令
├── src/
│   ├── commands/
│   │   ├── diet.js                   # bayiyuan-cli diet query '<json>'
│   │   ├── patient.js                # bayiyuan-cli patient search '<json>'
│   │   └── weight.js                 # bayiyuan-cli weight query '<json>'
│   ├── api/
│   │   └── client.js                 # axios 封装 (认证、超时、错误处理)
│   ├── formatter/
│   │   ├── diet.js                   # 饮食数据精简
│   │   ├── patient.js                # 患者数据精简
│   │   └── weight.js                 # 体重数据精简
│   └── config.js                     # 环境变量配置
├── skills/                           # SKILL.md 技能文件 (OpenClaw 读取)
│   ├── bayiyuan-diet-query/
│   │   └── SKILL.md
│   ├── bayiyuan-patient-search/
│   │   └── SKILL.md
│   └── bayiyuan-weight-query/
│       └── SKILL.md
└── test/
    ├── commands/
    │   ├── diet.test.js
    │   ├── patient.test.js
    │   └── weight.test.js
    └── formatter/
        ├── diet.test.js
        ├── patient.test.js
        └── weight.test.js
```

### 依赖

```json
{
  "name": "@csbr/bayiyuan-cli",
  "version": "1.0.0",
  "bin": { "bayiyuan-cli": "./bin/cli.js" },
  "dependencies": {
    "commander": "^12.0.0",
    "axios": "^1.7.0",
    "dayjs": "^1.11.0"
  },
  "devDependencies": {
    "vitest": "^2.0.0"
  }
}
```

### 核心文件设计

#### bin/cli.js — 主入口

```javascript
#!/usr/bin/env node
/**
 * 文件名：cli.js
 * 描述：bayiyuan-cli 主入口，注册子命令并处理全局错误
 * 作者：LR
 * 创建日期：2026-04-07
 */
const { program } = require('commander');
const dietCommand = require('../src/commands/diet');
const patientCommand = require('../src/commands/patient');
const weightCommand = require('../src/commands/weight');

program
  .name('bayiyuan-cli')
  .version('1.0.0')
  .description('八医院业务数据查询工具');

program.addCommand(dietCommand);
program.addCommand(patientCommand);
program.addCommand(weightCommand);

// 全局错误处理 — 返回 LLM 可理解的提示，不返回 stack trace
program.parseAsync(process.argv).catch((err) => {
  console.log(`错误: ${err.message}`);
  process.exit(1);
});
```

#### src/config.js — 配置管理

```javascript
/**
 * 文件名：config.js
 * 描述：从环境变量读取API配置，支持 Gateway 和 Direct 两种路由模式
 * 作者：LR
 * 创建日期：2026-04-07
 *
 * Gateway 模式: BAYIYUAN_API_URL=http://gateway:port
 *   请求路径: ${API_URL}/nutritional-service/diet-records/page-list
 *
 * Direct 模式: BAYIYUAN_API_URL_NUTRITIONAL=http://192.168.x.x:port
 *   请求路径: ${URL}/diet-records/page-list
 */
module.exports = {
  mode: process.env.BAYIYUAN_API_MODE || 'gateway',
  gatewayUrl: process.env.BAYIYUAN_API_URL,
  directUrls: {
    nutritional: process.env.BAYIYUAN_API_URL_NUTRITIONAL,
    personel: process.env.BAYIYUAN_API_URL_PERSONEL,
  },
  token: process.env.BAYIYUAN_API_TOKEN,
};
```

#### src/api/client.js — HTTP 客户端

```javascript
/**
 * 文件名：client.js
 * 描述：axios 实例封装，统一认证、超时、错误处理、路由模式切换
 * 作者：LR
 * 创建日期：2026-04-07
 *
 * 核心功能:
 * - 根据 config.mode 决定请求路径前缀
 * - 统一添加 Authorization: Bearer <token>
 * - 响应拦截器: 提取 CommonRes.data（微服务统一返回格式）
 * - 错误拦截器: 转为友好中文提示（不暴露技术细节给 LLM）
 * - 超时: 10000ms
 */
```

#### src/commands/diet.js — 饮食查询命令

```javascript
/**
 * 文件名：diet.js
 * 描述：饮食记录查询命令，调用 nutritional-service API 并精简返回数据
 * 作者：LR
 * 创建日期：2026-04-07
 *
 * 命令格式: bayiyuan-cli diet query '<json>'
 *
 * JSON 参数:
 *   patient (必填) — 患者姓名或 externalUserId
 *   days (可选, 默认1) — 查询天数
 *   date (可选, 默认今天) — 起始日期 YYYY-MM-DD
 *   meal (可选) — 餐次过滤: breakfast/lunch/dinner
 *
 * 执行流程:
 *   1. JSON.parse 解析参数, 校验 patient 必填
 *   2. 如果 patient 不像 ID 格式, 先调 /nutr-user-info 搜索 → 拿到 externalUserId
 *   3. 构造 DietRecordsQueryVO: { externalUserId, recordDate, mealType, startDate, endDate }
 *   4. POST /diet-records/page-list
 *   5. 调 formatter/diet.js 精简数据
 *   6. console.log 输出精简文本
 *
 * 目标 API:
 *   POST /diet-records/page-list
 *   请求体 DietRecordsQueryVO: { externalUserId, recordDate, mealType, startDate, endDate }
 *   响应体 CommonRes<PageListVO<DietRecordsRSVO>>
 *     DietRecordsRSVO: { recordDate, mealType, mealTime, totalCalories, dietRecordDetailsRSVOS[] }
 *     DietRecordDetailsRSVO: { foodName, quantity, unit, calories, giFlag }
 */
```

#### src/commands/patient.js — 患者搜索命令

```javascript
/**
 * 文件名：patient.js
 * 描述：患者信息搜索命令
 * 作者：LR
 * 创建日期：2026-04-07
 *
 * 命令格式: bayiyuan-cli patient search '<json>'
 *
 * JSON 参数:
 *   name (必填) — 患者姓名
 *   id (可选) — externalUserId，有则优先使用
 *
 * 目标 API:
 *   GET /nutr-user-info/detail?externalUserId= → UserInfoSetUpRSVO
 *   POST /nutr-user-info/user-current-course-node → CourseNodeRSVO (当前治疗阶段)
 */
```

#### src/commands/weight.js — 体重数据查询

```javascript
/**
 * 文件名：weight.js
 * 描述：患者体重和身体指标查询命令
 * 作者：LR
 * 创建日期：2026-04-07
 *
 * 命令格式: bayiyuan-cli weight query '<json>'
 *
 * JSON 参数:
 *   patient (必填) — 患者姓名或 externalUserId
 *   days (可选, 默认7) — 查询天数
 *   date (可选) — 起始日期 YYYY-MM-DD
 *
 * 目标 API:
 *   POST /user-weight-info/body-record-period → HealthChangeResultRSVO
 *   POST /user-weight-info/latest-body-record → UserWeightInfoRSVO
 *   POST /user-weight-info/body-record-days → List<UserWeightInfoRSVO>
 */
```

#### src/formatter/diet.js — 饮食数据精简

```javascript
/**
 * 文件名：diet.js
 * 描述：将 DietRecordsRSVO[] 转为精简文本摘要，大幅减少 token 消耗
 * 作者：LR
 * 创建日期：2026-04-07
 *
 * 精简规则:
 *   - 去掉系统字段: guid, tenantGuid, externalUserId
 *   - 每餐一行: "早餐(08:00) 380kcal - 食物1(120kcal)、食物2(60kcal)"
 *   - 底部汇总: "日总热量: 1230kcal / 目标1800kcal"
 *   - 无数据时: "未找到该患者的饮食记录"
 *
 * 输出示例 (~200 tokens):
 *   张三 2026-04-07 饮食记录：
 *   早餐(08:00) 380kcal - 全麦面包2片(180kcal)、牛奶200ml(130kcal)、鸡蛋1个(70kcal)
 *   午餐(12:30) 650kcal - 米饭200g(230kcal)、红烧鱼200g(375kcal)、青菜150g(45kcal)
 *   日总热量: 1030kcal / 目标1800kcal (差770kcal)
 */
```

***

## 方案 B：Java 版本

### 项目结构

```
bayiyuan-cli-java/
├── pom.xml                           # Picocli + OkHttp (不启动Spring Context)
├── src/main/java/com/csbr/cli/
│   ├── BayiyuanCli.java             # @Command 主入口
│   ├── commands/
│   │   ├── DietCommand.java          # @Command(name = "diet")
│   │   ├── PatientCommand.java       # @Command(name = "patient")
│   │   └── WeightCommand.java        # @Command(name = "weight")
│   ├── api/
│   │   └── ApiClient.java            # OkHttp 封装
│   ├── formatter/
│   │   ├── DietFormatter.java
│   │   ├── PatientFormatter.java
│   │   └── WeightFormatter.java
│   └── config/
│       └── CliConfig.java            # 环境变量配置
├── skills/                           # 同 Node.js 版本的 SKILL.md（共用）
│   └── (同上3个SKILL.md)
└── src/test/java/com/csbr/cli/
    └── commands/
        ├── DietCommandTest.java
        ├── PatientCommandTest.java
        └── WeightCommandTest.java
```

### 依赖

```xml
<!-- Picocli: 轻量级CLI框架, 不依赖Spring Boot -->
<dependency>
    <groupId>info.picocli</groupId>
    <artifactId>picocli</artifactId>
    <version>4.7.6</version>
</dependency>
<!-- OkHttp: HTTP客户端, 比Spring WebClient轻量 -->
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.12.0</version>
</dependency>
<!-- Jackson: JSON处理 -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
<!-- 可选: 引用现有的 csbr-cloud-entity 复用 VO 类 -->
<dependency>
    <groupId>com.csbr.qingcloud</groupId>
    <artifactId>csbr-cloud-entity</artifactId>
    <version>1.0.5-owm</version>
</dependency>
```

### 核心差异

- 使用 Picocli 注解驱动 `@Command`, `@Option`, `@Parameters`
- **不启动 Spring Context**（避免 JVM 启动慢），纯 Picocli + OkHttp
- 可引用 `csbr-cloud-entity` 复用 VO/DTO 类做反序列化
- 打包为 fat jar: `java -jar bayiyuan-cli.jar diet query '{"patient":"张三"}'`
- 后续可用 GraalVM native-image 编译为原生二进制（\~50ms启动）

***

## SKILL.md 文件（两个方案共用）

### skills/bayiyuan-diet-query/SKILL.md

```markdown
---
name: bayiyuan-diet-query
description: 查询患者饮食记录，包括每餐食物明细和热量。当用户询问某患者
  吃了什么、热量摄入、每日饮食情况时使用。
---
# 患者饮食记录查询

## 命令
bayiyuan-cli diet query '<JSON参数>'

## 参数
| 参数 | 必填 | 类型 | 说明 |
|------|------|------|------|
| patient | 是 | string | 患者姓名或ID |
| days | 否 | number | 查询天数，默认1 |
| date | 否 | string | 起始日期 YYYY-MM-DD，默认今天 |
| meal | 否 | string | 餐次: breakfast/lunch/dinner |

## 示例
bayiyuan-cli diet query '{"patient":"张三"}'
bayiyuan-cli diet query '{"patient":"张三","days":3}'
bayiyuan-cli diet query '{"patient":"张三","date":"2026-04-05","meal":"lunch"}'

## 输出格式
返回精简文本，包含每餐食物、热量和日汇总。无记录时返回提示。
```

### skills/bayiyuan-patient-search/SKILL.md

```markdown
---
name: bayiyuan-patient-search
description: 搜索患者基本信息和健康档案。当用户询问某患者的基本情况、
  治疗阶段、建档信息时使用。
---
# 患者信息搜索

## 命令
bayiyuan-cli patient search '<JSON参数>'

## 参数
| 参数 | 必填 | 类型 | 说明 |
|------|------|------|------|
| name | 是 | string | 患者姓名 |
| id | 否 | string | 患者ID(externalUserId)，有则优先使用 |

## 示例
bayiyuan-cli patient search '{"name":"张三"}'
bayiyuan-cli patient search '{"id":"ext_user_12345"}'
```

### skills/bayiyuan-weight-query/SKILL.md

```markdown
---
name: bayiyuan-weight-query
description: 查询患者体重和身体指标数据变化趋势。当用户询问某患者的体重、
  BMI、身体数据、体重变化时使用。
---
# 患者体重数据查询

## 命令
bayiyuan-cli weight query '<JSON参数>'

## 参数
| 参数 | 必填 | 类型 | 说明 |
|------|------|------|------|
| patient | 是 | string | 患者姓名或ID |
| days | 否 | number | 查询天数，默认7 |
| date | 否 | string | 起始日期 YYYY-MM-DD |

## 示例
bayiyuan-cli weight query '{"patient":"张三"}'
bayiyuan-cli weight query '{"patient":"张三","days":30}'
```

***

## API 路由配置（两种模式）

### 模式1: Gateway 统一入口

```bash
# 环境变量
BAYIYUAN_API_URL=http://gateway-host:port
BAYIYUAN_API_MODE=gateway
BAYIYUAN_API_TOKEN=your-service-account-token

# 请求路径 (需确认 Nacos 中的 gateway routes 前缀)
# 营养服务: ${API_URL}/nutritional-service/diet-records/page-list
# 人事服务: ${API_URL}/personel-service/staff/get-info-list
```

优点: 认证统一由 Gateway 的 Spring Security OAuth2 处理
注意: 需确认 Gateway 的路由规则前缀（在 Nacos 配置中心查看）

### 模式2: Nacos 直连

```bash
# 环境变量
BAYIYUAN_API_URL_NUTRITIONAL=http://192.168.x.x:port
BAYIYUAN_API_URL_PERSONEL=http://192.168.x.x:port
BAYIYUAN_API_MODE=direct

# 请求路径
# 营养服务: ${API_URL_NUTRITIONAL}/diet-records/page-list
# 人事服务: ${API_URL_PERSONEL}/staff/get-info-list
```

优点: 跳过 Gateway 延迟，更快
注意: 认证需单独处理，内部服务可能无需 token

***

## 关键源文件参考

### 营养服务 (ms-owm-nutritional-service)

| 文件                       | 路径                                         | 用途                                                                 |
| ------------------------ | ------------------------------------------ | ------------------------------------------------------------------ |
| DietRecordsController    | `controller/DietRecordsController.java`    | `/diet-records/` (page-list, detail, agent-record)                 |
| DailyNutritionController | `controller/DailyNutritionController.java` | `/daily-nutrition/` (user-daily-nutrition)                         |
| NutrUserInfoController   | `controller/NutrUserInfoController.java`   | `/nutr-user-info/` (detail, user-current-course-node)              |
| UserWeightInfoController | `controller/UserWeightInfoController.java` | `/user-weight-info/` (body-record-period, latest-body-record)      |
| DietRecordsQueryVO       | `domain/vo/DietRecordsQueryVO.java`        | 查询参数: externalUserId, recordDate, mealType, startDate, endDate     |
| DietRecordsRSVO          | `domain/vo/DietRecordsRSVO.java`           | 返回: recordDate, mealType, totalCalories, dietRecordDetailsRSVOS\[] |
| DietRecordDetailsRSVO    | `domain/vo/DietRecordDetailsRSVO.java`     | 明细: foodName, quantity, unit, calories, giFlag                     |
| DailyNutritionRSVO       | `domain/vo/DailyNutritionRSVO.java`        | 每日营养: totalCalories, calorieGoal, calorieDiff, mealCount           |

### 人事服务 (ms-owm-personel-service)

| 文件              | 路径                                | 用途                                              |
| --------------- | --------------------------------- | ----------------------------------------------- |
| StaffController | `controller/StaffController.java` | `/staff/` (get-info-list, getByGuid, page-list) |

### 网关 (ms-owm-gateway-server)

| 文件            | 路径                                        | 用途                                        |
| ------------- | ----------------------------------------- | ----------------------------------------- |
| bootstrap.yml | `resources/develop/bootstrap.yml`         | Nacos (192.168.6.21:8848, namespace: owm) |
| —             | Spring Security OAuth2 + csbr-cloud-idaas | 认证体系                                      |

***

## 实施顺序

### Phase 1: 基础框架（两个版本并行开发，选型验证）

| 步骤 | 内容                                            | 产出         |
| -- | --------------------------------------------- | ---------- |
| 1  | 初始化两个项目 (Node.js + Java)                      | 项目脚手架      |
| 2  | 实现 config + api/client                        | HTTP 客户端可用 |
| 3  | 实现 `bayiyuan-cli diet query` 单个命令 + formatter | 第一个命令可运行   |
| 4  | 对比测试: 冷启动时间、响应速度、开发体验                         | 选型报告       |
| 5  | **选定主方案**，另一个降级为备选                            | 决策         |

### Phase 2: 完成 MVP（3个命令）

| 步骤 | 内容                                 | 产出     |
| -- | ---------------------------------- | ------ |
| 6  | 实现 `patient search` 命令 + formatter | 患者搜索可用 |
| 7  | 实现 `weight query` 命令 + formatter   | 体重查询可用 |
| 8  | 编写 3 个 SKILL.md 文件                 | 技能定义   |
| 9  | 单元测试 (formatter + 命令)              | 测试覆盖   |

### Phase 3: 集成验证

| 步骤 | 内容                                      | 产出        |
| -- | --------------------------------------- | --------- |
| 10 | 部署到与微服务同网络的环境                           | 环境就绪      |
| 11 | 手动 CLI 测试 (正常/异常/边界)                    | 测试报告      |
| 12 | 安装 Skills 到 OpenClaw (`npx skills add`) | Skills 注册 |
| 13 | 飞书端到端测试                                 | 验证闭环      |
| 14 | Token 消耗对比测试 (对比 MCP 方案)                | 性能报告      |

***

## 验证方案

### 手动 CLI 测试

```bash
# 正常查询
bayiyuan-cli diet query '{"patient":"张三"}'
bayiyuan-cli patient search '{"name":"张三"}'
bayiyuan-cli weight query '{"patient":"张三","days":7}'

# 参数校验 — 缺少必填参数
bayiyuan-cli diet query '{}'
# 期望: 错误: 缺少必填参数 patient
#       用法: bayiyuan-cli diet query '{"patient":"张三"}'

# 未找到数据
bayiyuan-cli diet query '{"patient":"不存在的人"}'
# 期望: 未找到患者"不存在的人"的信息

# JSON 格式错误
bayiyuan-cli diet query '{patient:张三}'
# 期望: 错误: JSON 参数格式不正确
#       用法: bayiyuan-cli diet query '{"patient":"张三"}'
```

### 飞书端到端测试

| 用户输入           | 期望触发的命令                                    | 期望回复        |
| -------------- | ------------------------------------------ | ----------- |
| "查一下张三今天的饮食记录" | `diet query '{"patient":"张三"}'`            | 三餐食物和热量摘要   |
| "张三最近一周体重变化"   | `weight query '{"patient":"张三","days":7}'` | 体重趋势和变化幅度   |
| "查一下患者李四的信息"   | `patient search '{"name":"李四"}'`           | 患者基本信息和治疗阶段 |

### 性能指标

| 指标                   | Node.js 版本 | Java 版本                       | 目标值     |
| -------------------- | ---------- | ----------------------------- | ------- |
| 冷启动                  | < 300ms    | < 2s (jar) / < 100ms (native) | < 500ms |
| 单次查询端到端              | < 1s       | < 1.5s                        | < 2s    |
| system prompt tokens | \~60       | 同左                            | < 100   |
| 单次查询返回 tokens        | \~200      | 同左                            | < 300   |

***

## 待讨论问题（2026-04-08）

### 问题1: 姓名→ID 映射缺少明确方案 \[阻塞]

方案中 "如果 patient 不是 ID 格式，先调 `/nutr-user-info/detail` 搜索" 存在问题：`/nutr-user-info/detail` 接口是按 `externalUserId` 查询的，**没有按姓名模糊搜索的能力**。这是核心链路上的阻塞点——用户说"查张三的饮食"，CLI 拿到的是姓名而不是 ID。

**需确认：**

- 人事服务的 `/staff/get-info-list` 是否支持按姓名搜索并返回 externalUserId？
- 或者是否需要新增一个姓名搜索接口？

### 问题2: 认证方案不够具体 \[阻塞]

方案说 Gateway 模式用 `Bearer token`，Direct 模式"认证需单独处理"，但未明确：

- token 从哪来？是固定的服务账号 token，还是需要 OAuth2 客户端凭证流？
- token 过期怎么办？CLI 每次冷启动，没有 refresh 机制
- 内网直连模式下，微服务是否真的不需要认证？

### 问题3: SKILL.md meal 参数枚举与 API 不匹配

SKILL.md 里 meal 参数写的是 `breakfast/lunch/dinner`，但 `DietRecordsQueryVO.mealType` 的实际枚举值是中文：**"早餐/早加餐/午餐/午加餐/晚餐/晚加餐/零食"**。

**方案选项：**

- A: SKILL.md 直接用中文枚举（LLM 更容易匹配用户自然语言）
- B: CLI 内部做英文→中文映射

### 问题4: 重名患者处理

如果搜索"张三"返回多个患者，当前方案未处理。

**方案选项：**

- A: 返回候选列表，让 LLM 追问用户确认（如 "找到3个张三，请确认：1. 张三(内分泌科) 2. 张三(骨科)..."）
- B: 返回前 N 个匹配的摘要信息，由 LLM 自行判断

### 问题5: 错误输出格式缺乏标准化

方案只给了几个错误示例，但没有定义统一的错误输出格式。建议约定标准格式，方便 OpenClaw LLM 稳定解析：

```
[ERROR] <错误码>: <错误描述>
[HINT] <用法提示>
```

### 问题6: "体重打卡"的范围确认

MVP 写的是 "体重打卡"，但实际实现的是 `weight query`（查询）。如果需要支持体重录入（`/user-weight-info/agent-record` 接口已存在），应明确是否包含写入操作。

### 问题7: 是否有必要并行开发两个版本

Phase 1 要求 Node.js + Java 并行开发再选型，但 exec 场景下冷启动速度差异已经很明确（Node.js \~200ms vs Java \~1.5s）。

**方案选项：**

- A: 直接用 Node.js 做主方案，集中资源更快交付 MVP；Java 版本作为文档备选
- B: 维持现方案，两个版本并行开发后对比选型

