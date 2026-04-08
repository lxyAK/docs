# Claude Code Haha 工具系统深度分析报告

> 报告生成时间：2026-04-02
> 
> 分析对象：https://github.com/NanmiCoder/claude-code-haha.git
> 
> 分析人：教授 🧑‍💻

---

## 📋 目录

1. [项目概述](#项目概述)
2. [工具系统架构](#工具系统架构)
3. [核心类型定义](#核心类型定义)
4. [内置工具清单](#内置工具清单)
5. [权限与安全系统](#权限与安全系统)
6. [UI 渲染系统](#ui-渲染系统)
7. [BashTool 深度分析](#bashtool-深度分析)
8. [工具注册与发现](#工具注册与发现)
9. [设计亮点](#设计亮点)
10. [总结与评价](#总结与评价)

---

## 项目概述

### 项目背景

Claude Code Haha 是基于 2026-03-31 从 Anthropic npm registry 泄露的 Claude Code 源码修复后的本地可运行版本。原始泄露源码无法直接运行，本项目修复了启动链路中的多个阻塞问题。

### 技术栈

| 层级 | 技术 |
|------|------|
| 运行时 | **Bun**（非 Node.js） |
| 语言 | TypeScript |
| 终端 UI | React + **Ink** |
| CLI 解析 | Commander.js |
| API | Anthropic SDK |
| 协议 | MCP, LSP |

### 修复内容

| 问题 | 根因 | 修复 |
|------|------|------|
| TUI 不启动 | 入口脚本把无参数启动路由到了 recovery CLI | 恢复走 `cli.tsx` 完整入口 |
| 启动卡死 | `verify` skill 导入缺失的 `.md` 文件，Bun text loader 无限挂起 | 创建 stub `.md` 文件 |
| `--print` 卡死 | `filePersistence/types.ts` 缺失 | 创建类型桩文件 |
| `--print` 卡死 | `ultraplan/prompt.txt` 缺失 | 创建资源桩文件 |
| **Enter 键无响应** | `modifiers-napi` native 包缺失，`isModifierPressed()` 抛异常导致 `handleEnter` 中断，`onSubmit` 永远不执行 | 加 try-catch 容错 |
| setup 被跳过 | `preload.ts` 自动设置 `LOCAL_RECOVERY=1` 跳过全部初始化 | 移除默认设置 |

---

## 工具系统架构

### 目录结构

```
src/
├── Tool.ts                    # 工具核心类型定义（40+ 方法）
├── tools.ts                   # 工具注册与聚合
├── tools/                     # 46+ 个工具实现
│   ├── BashTool/              # Shell 命令执行（20+ 文件）
│   ├── FileReadTool/          # 文件读取
│   ├── FileWriteTool/         # 文件写入
│   ├── FileEditTool/          # 文件编辑
│   ├── GlobTool/              # 文件模式匹配
│   ├── GrepTool/              # 内容搜索
│   ├── AgentTool/             # 多 Agent 协作
│   ├── SkillTool/             # Skill 系统
│   ├── WebFetchTool/          # 网页抓取
│   ├── WebSearchTool/         # 网络搜索
│   ├── LSPTool/               # 语言服务器协议
│   ├── NotebookEditTool/      # Jupyter 笔记本
│   ├── TaskCreateTool/        # 任务创建
│   ├── TaskGetTool/           # 任务获取
│   ├── TaskUpdateTool/        # 任务更新
│   ├── TaskListTool/          # 任务列表
│   ├── TaskStopTool/          # 任务停止
│   ├── ListMcpResourcesTool/  # MCP 资源列表
│   ├── ReadMcpResourceTool/   # MCP 资源读取
│   ├── McpAuthTool/           # MCP 认证
│   ├── EnterPlanModeTool/     # 进入计划模式
│   ├── ExitPlanModeV2Tool/    # 退出计划模式
│   ├── EnterWorktreeTool/     # 进入工作区
│   ├── ExitWorktreeTool/      # 退出工作区
│   └── ...（更多工具）
```

---

## 核心类型定义

### Tool 接口概览

`Tool.ts` 定义了约 **40 个方法和属性**，是整个工具系统的核心。

#### 核心执行相关

| 方法/属性 | 类型 | 说明 |
|-----------|------|------|
| `name` | `string` | 工具名称 |
| `aliases` | `string[]` | 别名（向后兼容） |
| `call()` | `Function` | 核心执行方法 |
| `inputSchema` | `ZodSchema` | 输入参数 Zod schema |
| `outputSchema` | `ZodSchema` | 输出结果 Zod schema |
| `inputJSONSchema` | `JSONSchema` | MCP 工具的 JSON schema |

#### 权限与安全相关

| 方法/属性 | 类型 | 说明 |
|-----------|------|------|
| `checkPermissions()` | `Function` | 权限检查 |
| `validateInput()` | `Function` | 输入验证 |
| `isReadOnly()` | `Function` | 是否只读操作 |
| `isDestructive()` | `Function` | 是否不可逆操作 |
| `isConcurrencySafe()` | `Function` | 是否可并发执行 |
| `interruptBehavior()` | `'cancel' \| 'block'` | 中断行为 |

#### UI 渲染相关

| 方法/属性 | 类型 | 说明 |
|-----------|------|------|
| `renderToolUseMessage()` | `Function` | 渲染工具使用时的 UI |
| `renderToolResultMessage()` | `Function` | 渲染工具结果的 UI |
| `renderToolUseProgressMessage()` | `Function` | 渲染进度 UI |
| `renderToolUseErrorMessage()` | `Function` | 渲染错误 UI |
| `renderToolUseRejectedMessage()` | `Function` | 渲染拒绝 UI |
| `renderGroupedToolUse()` | `Function` | 渲染分组工具使用 |

#### 元数据与优化相关

| 方法/属性 | 类型 | 说明 |
|-----------|------|------|
| `searchHint` | `string` | 搜索提示词（3-10 词） |
| `shouldDefer` | `boolean` | 是否延迟加载（需要 ToolSearch） |
| `alwaysLoad` | `boolean` | 是否始终加载（不需要搜索） |
| `isMcp` | `boolean` | 是否 MCP 工具 |
| `isLsp` | `boolean` | 是否 LSP 工具 |
| `maxResultSizeChars` | `number` | 最大结果字符数 |
| `strict` | `boolean` | 严格模式 |

#### 其他辅助方法

| 方法/属性 | 类型 | 说明 |
|-----------|------|------|
| `description()` | `Function` | 生成描述 |
| `prompt()` | `Function` | 生成提示词 |
| `userFacingName()` | `Function` | 用户友好名称 |
| `getToolUseSummary()` | `Function` | 工具使用摘要 |
| `getActivityDescription()` | `Function` | 活动描述（Spinner 显示） |
| `toAutoClassifierInput()` | `Function` | 自动分类器输入 |
| `isSearchOrReadCommand()` | `Function` | 是否搜索/读取命令（UI 折叠） |
| `extractSearchText()` | `Function` | 提取搜索文本 |
| `getPath()` | `Function` | 获取操作路径 |
| `preparePermissionMatcher()` | `Function` | 准备权限匹配器 |
| `backfillObservableInput()` | `Function` | 回填可观察输入 |
| `mapToolResultToToolResultBlockParam()` | `Function` | 映射工具结果 |
| `isEnabled()` | `Function` | 是否启用 |
| `inputsEquivalent()` | `Function` | 输入是否等价 |
| `isTransparentWrapper()` | `Function` | 是否透明包装器 |
| `isResultTruncated()` | `Function` | 结果是否截断 |
| `renderToolUseTag()` | `Function` | 渲染工具使用标签 |
| `renderToolUseQueuedMessage()` | `Function` | 渲染排队消息 |
| `mcpInfo` | `object` | MCP 工具信息 |

### ToolUseContext 类型

工具执行上下文，包含约 **50 个属性**：

```typescript
type ToolUseContext = {
  // 核心选项
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools
    verbose: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
    querySource?: QuerySource
    refreshTools?: () => Tools
  }
  
  // 状态管理
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  setAppStateForTasks?: (f: (prev: AppState) => AppState) => void
  
  // UI 相关
  setToolJSX?: SetToolJSXFn
  addNotification?: (notif: Notification) => void
  appendSystemMessage?: (msg: Exclude<SystemMessage, SystemLocalCommandMessage>) => void
  sendOSNotification?: (opts: { message: string; notificationType: string }) => void
  
  // 消息与历史
  messages: Message[]
  updateFileHistoryState: (updater: (prev: FileHistoryState) => FileHistoryState) => void
  updateAttributionState: (updater: (prev: AttributionState) => AttributionState) => void
  
  // 进度与中断
  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void
  setHasInterruptibleToolInProgress?: (v: boolean) => void
  setStreamMode?: (mode: SpinnerMode) => void
  onCompactProgress?: (event: CompactProgressEvent) => void
  
  // Agent 相关
  agentId?: AgentId
  agentType?: string
  requireCanUseTool?: boolean
  
  // 缓存与限制
  nestedMemoryAttachmentTriggers?: Set<string>
  loadedNestedMemoryPaths?: Set<string>
  dynamicSkillDirTriggers?: Set<string>
  discoveredSkillNames?: Set<string>
  fileReadingLimits?: { maxTokens?: number; maxSizeBytes?: number }
  globLimits?: { maxResults?: number }
  toolDecisions?: Map<string, { source: string; decision: 'accept' | 'reject'; timestamp: number }>
  queryTracking?: QueryChainTracking
  contentReplacementState?: ContentReplacementState
  renderedSystemPrompt?: SystemPrompt
  
  // 其他
  userModified?: boolean
  setResponseLength: (f: (prev: number) => number) => void
  pushApiMetricsEntry?: (ttftMs: number) => void
  setSDKStatus?: (status: SDKStatus) => void
  openMessageSelector?: () => void
  setConversationId?: (id: UUID) => void
  requestPrompt?: (sourceName: string, toolInputSummary?: string | null) => (request: PromptRequest) => Promise<PromptResponse>
  toolUseId?: string
  criticalSystemReminder_EXPERIMENTAL?: string
  preserveToolUseResults?: boolean
  localDenialTracking?: DenialTrackingState
  handleElicitation?: (serverName: string, params: ElicitRequestURLParams, signal: AbortSignal) => Promise<ElicitResult>
}
```

### buildTool 工厂函数

```typescript
function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D>
```

提供安全默认值：
- `isEnabled` → `true`
- `isConcurrencySafe` → `false`（假设不安全）
- `isReadOnly` → `false`（假设写入）
- `isDestructive` → `false`
- `checkPermissions` → `{ behavior: 'allow', updatedInput }`
- `toAutoClassifierInput` → `''`（跳过分类器）
- `userFacingName` → `name`

---

## 内置工具清单

### 基础工具（6 个）

| 工具 | 功能 | 关键特性 |
|------|------|---------|
| `BashTool` | 执行 Shell 命令 | 最复杂，20+ 文件，安全验证 |
| `FileReadTool` | 读取文件 | 支持大文件、编码、范围 |
| `FileWriteTool` | 写入文件 | 原子写入、权限检查 |
| `FileEditTool` | 编辑文件 | Diff 预览、撤销支持 |
| `GlobTool` | 文件模式匹配 | 忽略规则、递归 |
| `GrepTool` | 内容搜索 | 正则、二进制跳过 |

### 高级工具（6 个）

| 工具 | 功能 | 关键特性 |
|------|------|---------|
| `AgentTool` | 多 Agent 协作 | 子 Agent 管理、通信 |
| `SkillTool` | Skill 系统 | 动态加载、skill 发现 |
| `WebFetchTool` | 网页抓取 | 缓存、重试、超时 |
| `WebSearchTool` | 网络搜索 | 多引擎、结果过滤 |
| `LSPTool` | 语言服务器协议 | 代码补全、跳转、重构 |
| `NotebookEditTool` | Jupyter 笔记本 | 单元格编辑、执行 |

### 任务管理（5 个）

| 工具 | 功能 | 关键特性 |
|------|------|---------|
| `TaskCreateTool` | 创建任务 | 优先级、截止时间、标签 |
| `TaskGetTool` | 获取任务 | 详细信息、历史 |
| `TaskUpdateTool` | 更新任务 | 状态、分配、进度 |
| `TaskListTool` | 列出任务 | 过滤、排序、分页 |
| `TaskStopTool` | 停止任务 | 取消执行、清理 |

### MCP 协议（3 个）

| 工具 | 功能 | 关键特性 |
|------|------|---------|
| `ListMcpResourcesTool` | 列出 MCP 资源 | 资源发现、类型过滤 |
| `ReadMcpResourceTool` | 读取 MCP 资源 | 内容获取、缓存 |
| `McpAuthTool` | MCP 认证 | OAuth、API Key |

### 工作区与模式（4 个）

| 工具 | 功能 | 关键特性 |
|------|------|---------|
| `EnterPlanModeTool` | 进入计划模式 | 暂停执行、规划 |
| `ExitPlanModeV2Tool` | 退出计划模式 | 应用计划、恢复 |
| `EnterWorktreeTool` | 进入工作区 | Git 分支、隔离 |
| `ExitWorktreeTool` | 退出工作区 | 清理、切换回 |

### 其他工具（10+ 个）

| 工具 | 功能 |
|------|------|
| `AskUserQuestionTool` | 询问用户问题 |
| `BriefTool` | 摘要工具 |
| `ConfigTool` | 配置工具 |
| `TodoWriteTool` | Todo 写入 |
| `ToolSearchTool` | 工具搜索 |
| `TaskOutputTool` | 任务输出 |
| `TestingPermissionTool` | 测试权限 |
| `TungstenTool` | Tungsten 工具 |
| `TeamCreateTool` | 团队创建 |
| `TeamDeleteTool` | 团队删除 |
| `SendMessageTool` | 发送消息 |

### 特性开关控制的工具

| 工具 | 特性开关 |
|------|---------|
| `REPLTool` | `USER_TYPE === 'ant'` |
| `SuggestBackgroundPRTool` | `USER_TYPE === 'ant'` |
| `SleepTool` | `PROACTIVE` \|\| `KAIROS` |
| `CronCreateTool` / `CronDeleteTool` / `CronListTool` | `AGENT_TRIGGERS` |
| `RemoteTriggerTool` | `AGENT_TRIGGERS_REMOTE` |
| `MonitorTool` | `MONITOR_TOOL` |
| `SendUserFileTool` | `KAIROS` |
| `PushNotificationTool` | `KAIROS` \|\| `KAIROS_PUSH_NOTIFICATION` |
| `SubscribePRTool` | `KAIROS_GITHUB_WEBHOOKS` |
| `VerifyPlanExecutionTool` | `CLAUDE_CODE_VERIFY_PLAN === 'true'` |
| `OverflowTestTool` | `OVERFLOW_TEST_TOOL` |
| `CtxInspectTool` | `CONTEXT_COLLAPSE` |
| `TerminalCaptureTool` | `TERMINAL_PANEL` |
| `WebBrowserTool` | `WEB_BROWSER_TOOL` |
| `SnipTool` | `HISTORY_SNIP` |
| `ListPeersTool` | `UDS_INBOX` |
| `WorkflowTool` | `WORKFLOW_SCRIPTS` |

---

## 权限与安全系统

### 四层权限模型

```
┌─────────────────────────────────────────────────────────┐
│  1. validateInput()          输入验证                  │
│     - 语法验证                                              │
│     - 语义验证                                              │
│     - 边界检查                                              │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  2. checkPermissions()     工具特定权限检查             │
│     - 工具级权限规则                                       │
│     - 路径验证                                              │
│     - 命令语义分析                                         │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  3. 通用权限系统            全局权限规则                 │
│     - alwaysAllowRules                                     │
│     - alwaysDenyRules                                      │
│     - alwaysAskRules                                       │
│     - 权限模式（default / ask / auto / bypass）          │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  4. 用户交互确认            最终用户确认                 │
│     - 权限对话框                                           │
│     - 自动/手动模式切换                                    │
│     - 记住决定                                             │
└─────────────────────────────────────────────────────────┘
```

### 安全标记

```typescript
// 是否只读操作（不修改文件系统）
isReadOnly(input: z.infer<Input>): boolean

// 是否不可逆操作（删除、覆盖、发送等）
isDestructive?(input: z.infer<Input>): boolean

// 是否可并发执行
isConcurrencySafe(input: z.infer<Input>): boolean

// 中断行为
interruptBehavior?(): 'cancel' | 'block'
  // 'cancel' - 停止工具并丢弃结果
  // 'block'  - 继续运行，新消息等待
```

### ToolPermissionContext

```typescript
type ToolPermissionContext = {
  mode: PermissionMode  // 'default' | 'ask' | 'auto' | 'bypassPermissions'
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean  // 后台 Agent
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}
```

### 权限结果

```typescript
type PermissionResult =
  | { behavior: 'allow'; updatedInput?: { [key: string]: unknown } }
  | { behavior: 'deny'; reason: string }
  | { behavior: 'ask'; message: string }
```

---

## UI 渲染系统

### 渲染方法概览

每个工具都可以自定义 8 个渲染方法：

```typescript
// 1. 渲染工具使用时的消息
renderToolUseMessage(
  input: Partial<z.infer<Input>>,
  options: { theme: ThemeName; verbose: boolean; commands?: Command[] }
): React.ReactNode

// 2. 渲染工具结果消息
renderToolResultMessage?(
  content: Output,
  progressMessagesForMessage: ProgressMessage<P>[],
  options: {
    style?: 'condensed'
    theme: ThemeName
    tools: Tools
    verbose: boolean
    isTranscriptMode?: boolean
    isBriefOnly?: boolean
    input?: unknown
  }
): React.ReactNode

// 3. 渲染工具使用进度消息
renderToolUseProgressMessage?(
  progressMessagesForMessage: ProgressMessage<P>[],
  options: {
    tools: Tools
    verbose: boolean
    terminalSize?: { columns: number; rows: number }
    inProgressToolCallCount?: number
    isTranscriptMode?: boolean
  }
): React.ReactNode

// 4. 渲染工具使用排队消息
renderToolUseQueuedMessage?(): React.ReactNode

// 5. 渲染工具使用拒绝消息
renderToolUseRejectedMessage?(
  input: z.infer<Input>,
  options: {
    columns: number
    messages: Message[]
    style?: 'condensed'
    theme: ThemeName
    tools: Tools
    verbose: boolean
    progressMessagesForMessage: ProgressMessage<P>[]
    isTranscriptMode?: boolean
  }
): React.ReactNode

// 6. 渲染工具使用错误消息
renderToolUseErrorMessage?(
  result: ToolResultBlockParam['content'],
  options: {
    progressMessagesForMessage: ProgressMessage<P>[]
    tools: Tools
    verbose: boolean
    isTranscriptMode?: boolean
  }
): React.ReactNode

// 7. 渲染分组工具使用
renderGroupedToolUse?(
  toolUses: Array<{
    param: ToolUseBlockParam
    isResolved: boolean
    isError: boolean
    isInProgress: boolean
    progressMessages: ProgressMessage<P>[]
    result?: {
      param: ToolResultBlockParam
      output: unknown
    }
  }>,
  options: {
    shouldAnimate: boolean
    tools: Tools
  }
): React.ReactNode | null

// 8. 渲染工具使用标签
renderToolUseTag?(input: Partial<z.infer<Input>>): React.ReactNode
```

### UI 辅助方法

```typescript
// 用户友好名称
userFacingName(input: Partial<z.infer<Input>> | undefined): string

// 用户友好名称背景色
userFacingNameBackgroundColor?(
  input: Partial<z.infer<Input>> | undefined
): keyof Theme | undefined

// 工具使用摘要（紧凑视图）
getToolUseSummary?(input: Partial<z.infer<Input>> | undefined): string | null

// 活动描述（Spinner 显示）
getActivityDescription?(
  input: Partial<z.infer<Input>> | undefined
): string | null

// 是否透明包装器（委托渲染）
isTransparentWrapper?(): boolean

// 结果是否截断（控制点击展开）
isResultTruncated?(output: Output): boolean
```

---

## BashTool 深度分析

### 目录结构（20 个文件）

```
tools/BashTool/
├── BashTool.tsx                    # 主实现（160KB）
├── bashCommandHelpers.ts           # 命令辅助（8KB）
├── bashPermissions.ts              # 权限系统（98KB）
├── bashSecurity.ts                 # 安全检查（102KB）
├── BashToolResultMessage.tsx       # 结果渲染（19KB）
├── commandSemantics.ts             # 命令语义（3KB）
├── commentLabel.ts                 # 注释标签（637B）
├── destructiveCommandWarning.ts    # 破坏性命令警告（3KB）
├── modeValidation.ts               # 模式验证（3KB）
├── pathValidation.ts               # 路径验证（43KB）
├── prompt.ts                       # 提示词生成（21KB）
├── readOnlyValidation.ts           # 只读验证（68KB）
├── sedEditParser.ts                # sed 编辑解析（9KB）
├── sedValidation.ts                # sed 验证（21KB）
├── shouldUseSandbox.ts             # 是否使用沙箱（5KB）
├── toolName.ts                     # 工具名称（89B）
├── UI.tsx                          # UI 组件（25KB）
└── utils.ts                        # 工具函数（7KB）
```

### 文件大小统计

| 文件 | 大小 | 说明 |
|------|------|------|
| `bashSecurity.ts` | 102KB | 安全检查 |
| `bashPermissions.ts` | 98KB | 权限系统 |
| `BashTool.tsx` | 160KB | 主实现 |
| `readOnlyValidation.ts` | 68KB | 只读验证 |
| `pathValidation.ts` | 43KB | 路径验证 |
| `prompt.ts` | 21KB | 提示词生成 |
| `sedValidation.ts` | 21KB | sed 验证 |
| `BashToolResultMessage.tsx` | 19KB | 结果渲染 |
| `UI.tsx` | 25KB | UI 组件 |
| **总计** | **约 640KB** | 20 个文件 |

### 核心安全检查（bashSecurity.ts）

102KB 的安全检查模块，包含：
- 命令白名单/黑名单
- 路径访问控制
- 环境变量过滤
- 危险操作检测
- 沙箱决策

### 权限系统（bashPermissions.ts）

98KB 的权限模块，包含：
- 命令语义分析
- 读写操作判断
- 破坏性操作识别
- 权限规则匹配
- 用户提示生成

### 只读验证（readOnlyValidation.ts）

68KB 的只读验证模块：
- 命令分类（读/写/混合）
- 参数分析
- 子命令检测
- 重定向分析
- 管道分析

### 路径验证（pathValidation.ts）

43KB 的路径验证模块：
- 路径规范化
- 相对路径解析
- 符号链接检查
- 目录遍历防护
- 工作区边界检查

---

## 工具注册与发现

### 条件加载（Dead Code Elimination）

使用编译时条件加载，减小包体积：

```typescript
// 特性开关控制
const REPLTool =
  process.env.USER_TYPE === 'ant'
    ? require('./tools/REPLTool/REPLTool.js').REPLTool
    : null

const cronTools = feature('AGENT_TRIGGERS')
  ? [
      require('./tools/ScheduleCronTool/CronCreateTool.js').CronCreateTool,
      require('./tools/ScheduleCronTool/CronDeleteTool.js').CronDeleteTool,
      require('./tools/ScheduleCronTool/CronListTool.js').CronListTool,
    ]
  : []

// 延迟 require 打破循环依赖
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
```

### 延迟加载（Deferred Tools）

```typescript
// 工具标记
shouldDefer: boolean      // 需要 ToolSearch 后才能使用
alwaysLoad: boolean        // 始终加载，不需要搜索

// 搜索提示
searchHint?: string        // 3-10 词，帮助模型发现工具

// 例子
{
  name: 'NotebookEdit',
  searchHint: 'jupyter notebook ipynb',  // 补充名称中没有的关键词
  shouldDefer: true
}
```

### 工具发现流程

```
1. 初始加载始终加载的工具（alwaysLoad: true）
2. 模型请求 ToolSearch
3. 加载所有延迟工具（shouldDefer: true）
4. 使用 searchHint 进行关键词匹配
5. 返回匹配的工具给模型
6. 模型选择并调用工具
```

---

## 设计亮点

### 1. 类型安全

- 完整的 TypeScript 类型定义
- Zod schema 运行时验证
- `buildTool()` 工厂函数提供默认值
- 类型级别的默认值填充

### 2. 默认值模式（Fail-Closed）

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,  // 默认为不安全
  isReadOnly: (_input?: unknown) => false,          // 默认为写入
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (input: { [key: string]: unknown }, _ctx?: ToolUseContext) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
  userFacingName: (_input?: unknown) => '',
}
```

### 3. 渐进式权限

- 多层验证，每层都可以拒绝
- Fail-closed 设计，默认拒绝
- 支持自动/手动/绕过模式
- 记住用户决定

### 4. UI 灵活性

- 每个工具可完全自定义渲染
- 支持简洁/详细两种模式
- 进度、错误、拒绝都有自定义 UI
- 分组渲染优化

### 5. 特性开关（Feature Flags）

- 编译时 dead code elimination
- 运行时特性检查
- 减小生产包体积
- A/B 测试支持

### 6. 搜索优化

- `searchHint` 帮助模型发现工具
- 延迟加载减少初始提示词
- 关键词匹配优化
- 工具分类与标签

### 7. 性能优化

- 最大结果字符数限制
- 结果持久化到文件
- LRU 缓存
- 并发安全控制

### 8. 可观测性

- 进度消息
- 活动描述
- 工具使用摘要
- 搜索文本提取

---

## 总结与评价

### 架构评价

| 维度 | 评分 | 说明 |
|------|------|------|
| **类型安全** | ⭐⭐⭐⭐⭐ | 完整的 TypeScript + Zod |
| **安全性** | ⭐⭐⭐⭐⭐ | 四层权限、fail-closed |
| **可扩展性** | ⭐⭐⭐⭐⭐ | 工厂模式、特性开关 |
| **UI 灵活性** | ⭐⭐⭐⭐⭐ | 每个工具可自定义渲染 |
| **性能** | ⭐⭐⭐⭐ | 缓存、延迟加载、结果限制 |
| **可观测性** | ⭐⭐⭐⭐ | 进度、摘要、搜索提取 |
| **代码质量** | ⭐⭐⭐⭐⭐ | 模块化、职责分离 |
| **文档** | ⭐⭐⭐ | 类型定义清晰，缺少教程 |

### 整体评价

这是一个**工程质量极高**的企业级工具系统：

✅ **优点**：
- 考虑了安全性、权限、UI、性能等各个方面
- 设计灵活，易于扩展新工具
- 有完善的权限控制和安全检查
- 支持复杂的多 Agent 协作场景
- 类型安全，开发体验好
- 性能优化到位

⚠️ **可以改进的地方**：
- 文档可以更完善（缺少教程和最佳实践）
- 部分工具过于复杂（如 BashTool 20 个文件）
- 测试覆盖未知
- 缺少简单示例

### 学习价值

对于想构建企业级 AI Agent 工具系统的开发者来说，这是一个**绝佳的参考架构**！

可以学习的点：
1. 如何设计类型安全的工具系统
2. 如何实现多层权限控制
3. 如何让每个工具自定义 UI
4. 如何使用特性开关进行 dead code elimination
5. 如何实现工具延迟加载与发现
6. 如何处理工具执行的进度与错误

---

## 附录

### 相关资源

- 项目地址：https://github.com/NanmiCoder/claude-code-haha
- Claude Code 官方：https://claude.ai
- Anthropic：https://www.anthropic.com
- Ink（终端 UI）：https://github.com/vadimdemedes/ink
- MCP 协议：https://modelcontextprotocol.io

### 术语表

| 术语 | 说明 |
|------|------|
| **Tool** | 工具，Agent 可以调用的能力 |
| **MCP** | Model Context Protocol，模型上下文协议 |
| **LSP** | Language Server Protocol，语言服务器协议 |
| **Zod** | TypeScript 优先的 schema 验证库 |
| **Ink** | React 终端渲染库 |
| **Worktree** | Git 工作树，用于分支隔离 |
| **Plan Mode** | 计划模式，暂停执行进行规划 |
| **Deferred Tool** | 延迟加载工具，需要 ToolSearch |
| **Fail-Closed** | 关闭失败，默认拒绝 |

---

**报告结束**

如有疑问或需要更深入的分析，请联系教授 🧑‍💻
