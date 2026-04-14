# Hermes Agent Profile 多实例隔离机制深度分析

> 基于源码 `D:\CSBR\opensource\hermes-agent-main` 的深度分析
> 分析日期：2026-04-14

## 一、概述

Profile 是 Hermes Agent 的**多实例隔离机制**，允许在同一台机器上运行多个完全独立的 Agent 实例。每个 Profile 拥有独立的配置、记忆、技能、会话历史、定时任务和 Gateway 进程，实现真正的数据隔离。

### 设计定位

Profile 不是"多 Agent 并发协作"机制，而是"多身份切换"机制：

- **同一时刻只有一个 Profile 在运行**（单进程模型）
- 切换 Profile 通过 `HERMES_HOME` 环境变量重定向实现
- 设计目标是：工作 Agent、开发 Agent、个人 Agent 使用不同的模型/工具/人格，互不干扰

### 核心源码

| 文件 | 职责 | 行数 |
|------|------|------|
| `hermes_cli/profiles.py` | Profile CRUD、隔离、导出/导入 | 1095 行 |
| `hermes_cli/main.py:83-138` | `_apply_profile_override()` 启动时 Profile 解析 | 55 行 |
| `hermes_cli/main.py:5850-5905` | Profile 子命令 argparse 定义 | 55 行 |
| `hermes_cli/main.py:4182-4400` | Profile 子命令处理器 | 218 行 |
| `hermes_cli/gateway.py:380-432` | Gateway 的 Profile 感知逻辑 | 52 行 |
| `hermes_constants.py:11-137` | `get_hermes_home()`、`get_subprocess_home()` 路径解析 | 127 行 |

---

## 二、核心原理：HERMES_HOME 重定向

### 2.1 隔离的根本机制

Hermes 的所有模块（配置、记忆、技能、会话、日志等）都通过 `get_hermes_home()` 解析根路径：

```python
# hermes_constants.py:11-17
def get_hermes_home() -> Path:
    """Return the active Hermes home directory.
    Reads HERMES_HOME env var, falls back to ~/.hermes.
    """
    return Path(os.getenv("HERMES_HOME", Path.home() / ".hermes"))
```

Profile 的隔离本质上就是**在进程启动时将 `HERMES_HOME` 切换到不同的目录**。之后所有模块的路径解析自然指向各自的独立存储。

### 2.2 启动时解析流程

`_apply_profile_override()` 在所有 Hermes 模块加载之前执行（`main.py:138`），确保后续所有 import 的模块都使用正确的 `HERMES_HOME`：

```
用户执行: hermes --profile work-agent chat

解析流程:
  ① 扫描 sys.argv，找到 --profile work-agent
  ② 调用 resolve_profile_env("work-agent")
     → 返回 "~/.hermes/profiles/work-agent"
  ③ 设置 os.environ["HERMES_HOME"] = "~/.hermes/profiles/work-agent"
  ④ 从 sys.argv 中移除 --profile 参数
  ⑤ 继续正常的 argparse 解析
  ⑥ 后续所有模块调用 get_hermes_home() 都指向新路径
```

### 2.3 三级回退机制

Profile 解析有三个优先级层级：

| 优先级 | 来源 | 触发条件 |
|--------|------|----------|
| 1（最高） | `--profile <name>` / `-p <name>` CLI 参数 | 用户显式指定 |
| 2 | `~/.hermes/active_profile` 文件 | "粘性默认"（`hermes profile use` 设置） |
| 3（默认） | `~/.hermes` 目录 | 无任何配置时使用默认 Profile |

```python
# main.py:89-111 伪代码
if "--profile" in argv or "-p" in argv:
    profile_name = 解析参数值
elif exists("~/.hermes/active_profile"):
    profile_name = 读取文件内容
else:
    profile_name = None  # 使用默认

if profile_name:
    os.environ["HERMES_HOME"] = resolve_profile_env(profile_name)
```

---

## 三、存储结构

### 3.1 目录布局

```
~/.hermes/                              ← 默认 Profile 根目录
├── config.yaml                         ← 默认 Agent 的配置
├── .env                                ← 默认 Agent 的环境变量
├── SOUL.md                             ← 默认 Agent 的人格
├── memories/                           ← 默认 Agent 的记忆
├── sessions/                           ← 默认 Agent 的会话历史
├── skills/                             ← 默认 Agent 的技能库
├── logs/                               ← 默认 Agent 的日志
├── cron/                               ← 默认 Agent 的定时任务
├── active_profile                      ← 当前粘性默认 Profile 名
│
└── profiles/                           ← 所有命名 Profile
    ├── work-agent/
    │   ├── config.yaml                 ← work-agent 的独立配置
    │   ├── .env                        ← work-agent 的独立密钥
    │   ├── SOUL.md                     ← work-agent 的独立人格
    │   ├── memories/                   ← work-agent 的独立记忆
    │   ├── sessions/
    │   ├── skills/
    │   ├── skins/
    │   ├── logs/
    │   ├── plans/
    │   ├── workspace/
    │   ├── cron/
    │   └── home/                       ← 子进程 HOME（git/ssh 隔离）
    │
    └── dev-agent/
        ├── config.yaml
        ├── .env
        ├── ...
```

**关键区别**：
- `default` Profile 就是 `~/.hermes/` 本身（向后兼容，零迁移）
- 命名 Profile 存放在 `~/.hermes/profiles/<name>/` 下
- 不能创建名为 `default` 的 Profile（保留字）

### 3.2 每个 Profile 的 9 个独立目录

```python
# profiles.py:36-50
_PROFILE_DIRS = [
    "memories",      # 持久化记忆（MEMORY.md, USER.md）
    "sessions",      # 会话历史
    "skills",        # 安装的技能（SKILL.md 文件）
    "skins",         # UI 皮肤
    "logs",          # 运行日志
    "plans",         # 计划文件
    "workspace",     # 工作区
    "cron",          # 定时任务配置
    "home",          # 子进程 HOME（隔离 git/ssh/gh 配置）
]
```

### 3.3 子进程 HOME 隔离

`home/` 目录是一个精心设计的隔离层：

```python
# hermes_constants.py:114-137
def get_subprocess_home() -> str | None:
    """Return a per-profile HOME directory for subprocesses.
    Provides:
    - Docker persistence: tool configs land inside persistent volume
    - Profile isolation: each profile gets its own git identity, SSH keys, gh tokens
    """
    hermes_home = os.getenv("HERMES_HOME")
    profile_home = os.path.join(hermes_home, "home")
    if os.path.isdir(profile_home):
        return profile_home
    return None
```

当终端工具、浏览器等子进程启动时，如果 `{HERMES_HOME}/home/` 存在，Hermes 会将 `HOME` 环境变量设为该路径。这意味着：

- 不同 Profile 的 `git config` 完全独立（不同用户名/邮箱）
- SSH 密钥不互相暴露
- `gh`（GitHub CLI）认证令牌隔离
- npm/pip 等工具的配置文件隔离

---

## 四、Profile 生命周期管理

### 4.1 创建 Profile

```bash
# 全新空白 Profile
hermes profile create work-agent

# 基于当前 Profile 克隆配置
hermes profile create work-agent --clone

# 从指定 Profile 克隆
hermes profile create work-agent --clone-from dev-agent

# 完整复制（包括会话历史、记忆等所有数据）
hermes profile create work-agent --clone-all
```

创建流程源码（`profiles.py:381-472`）：

```
① 校验名称（小写字母/数字/连字符，最长 64 字符）
② 检查是否已存在
③ 解析克隆源
④ 创建目录结构（9 个子目录）
⑤ 复制配置文件（--clone 时）
   - config.yaml, .env, SOUL.md
   - memories/MEMORY.md, memories/USER.md
⑥ 写入默认 SOUL.md（如果不存在）
⑦ 播种内置技能（子进程方式，设置 HERMES_HOME）
⑧ 创建 wrapper 别名脚本（~/.local/bin/<name>）
```

### 4.2 克隆的三种模式

| 模式 | 参数 | 复制内容 | 用途 |
|------|------|----------|------|
| 空白创建 | 无参数 | 仅目录结构 + 默认技能 | 从零开始配置 |
| 配置克隆 | `--clone` | config.yaml, .env, SOUL.md, MEMORY.md, USER.md | 复制基础配置 |
| 全量克隆 | `--clone-all` | 整个目录（排除运行时文件） | 完整复制一个 Agent |

全量克隆时会剥离的运行时文件（`_CLONE_ALL_STRIP`）：

```python
# profiles.py:68-72
_CLONE_ALL_STRIP = [
    "gateway.pid",          # Gateway 进程 PID
    "gateway_state.json",   # Gateway 运行状态
    "processes.json",       # 后台进程列表
]
```

### 4.3 名称校验规则

```python
# profiles.py:33
_PROFILE_ID_RE = re.compile(r"^[a-z0-9][a-z0-9_-]{0,63}$")
```

- 必须以小写字母或数字开头
- 只允许小写字母、数字、连字符、下划线
- 最长 64 字符
- 禁止使用的保留名：`hermes`, `default`, `test`, `tmp`, `root`, `sudo`
- 禁止使用 Hermes 子命令名：`chat`, `model`, `gateway`, `setup` 等

### 4.4 删除 Profile

```bash
hermes profile delete work-agent
hermes profile delete work-agent -y   # 跳过确认
```

删除流程（`profiles.py:508-597`）：

```
① 禁止删除 default Profile
② 显示将删除的内容（模型、技能数、Gateway 状态）
③ 要求输入 Profile 名确认（-y 跳过）
④ 禁用并移除 systemd/launchd 服务
⑤ 停止运行中的 Gateway 进程（SIGTERM → 等待 10s → SIGKILL）
⑥ 移除 wrapper 别名脚本
⑦ 删除 Profile 目录（shutil.rmtree）
⑧ 如果当前活跃 Profile 是被删除的，重置为 default
```

### 4.5 重命名 Profile

```bash
hermes profile rename work-agent company-agent
```

重命名流程（`profiles.py:940-987`）：

```
① 停止 Gateway（如果在运行）
② 目录重命名
③ 更新 wrapper 脚本（删旧建新）
④ 更新 active_profile（如果指向旧名）
```

### 4.6 粘性默认（Sticky Default）

```bash
hermes profile use work-agent   # 设置为默认
hermes profile use default      # 恢复到默认
```

实现方式（`profiles.py:701-722`）：

```python
def set_active_profile(name: str) -> None:
    path = _get_active_profile_path()  # ~/.hermes/active_profile
    if name == "default":
        path.unlink(missing_ok=True)   # 删除文件 = 使用默认
    else:
        tmp = path.with_suffix(".tmp")
        tmp.write_text(name + "\n")
        tmp.replace(path)              # 原子写入
```

---

## 五、使用方式

### 5.1 两种使用模式

| 模式 | 命令 | 说明 |
|------|------|------|
| 临时单次 | `hermes -p work-agent chat` | 仅本次生效，不改变默认 |
| 长期默认 | `hermes profile use work-agent` | 写入 `active_profile`，后续自动生效 |

`-p` 是 `--profile` 的短形式，也支持 `--profile=work-agent` 等号语法。

### 5.2 Wrapper 别名

创建 Profile 时自动生成 shell wrapper 脚本：

```bash
# ~/.local/bin/work-agent
#!/bin/sh
exec hermes -p work-agent "$@"
```

之后可以直接用 Profile 名作为命令：

```bash
work-agent chat             # 等同于 hermes -p work-agent chat
work-agent setup            # 配置 work-agent
work-agent gateway start    # 启动 work-agent 的 Gateway
```

别名创建时的冲突检查（`profiles.py:188-218`）：

```
① 检查保留名（hermes, default, sudo 等）
② 检查 Hermes 子命令（chat, model, gateway 等）
③ 检查 PATH 中是否已有同名命令
④ 允许覆盖已有的 Hermes wrapper（包含 "hermes -p" 的脚本）
```

### 5.3 Shell 自动补全

Hermes 提供 bash 和 zsh 的 Profile 名补全（`profiles.py:994-1072`）：

```bash
# 安装补全
eval "$(hermes completion bash)"
# 或
eval "$(hermes completion zsh)"
```

补全支持：
- `-p` / `--profile` 后自动列出所有 Profile 名
- `hermes profile use/delete/show` 后自动补全 Profile 名
- `hermes profile` 后补全子命令

---

## 六、导出与导入

### 6.1 导出

```bash
hermes profile export work-agent                    # 输出 work-agent.tar.gz
hermes profile export work-agent -o /tmp/backup.gz  # 指定输出路径
```

导出逻辑（`profiles.py:780-820`）：

**命名 Profile**：排除凭证文件后打包

```python
_CREDENTIAL_FILES = {"auth.json", ".env"}
shutil.copytree(profile_dir, staged,
                ignore=lambda d, contents: _CREDENTIAL_FILES & set(contents))
```

**默认 Profile**（`~/.hermes` 本身）：排除大量基础设施文件

```python
# profiles.py:78-100
_DEFAULT_EXPORT_EXCLUDE_ROOT = frozenset({
    "hermes-agent",         # 源码仓库（多 GB）
    ".worktrees",           # git worktrees
    "profiles",             # 其他 Profile（避免递归导出）
    "bin",                  # 安装的二进制
    "node_modules",         # npm 包
    "state.db",             # SQLite 数据库
    "auth.json",            # API 密钥
    ".env",                 # 环境变量
    "logs",                 # 日志
    "sandboxes",            # 沙箱
    # ... 等
})
```

### 6.2 导入

```bash
hermes profile import ./work-agent.tar.gz               # 自动推断名称
hermes profile import ./work-agent.tar.gz --name agent2  # 指定名称
```

安全措施（`profiles.py:823-933`）：

```
① 路径安全检查：拒绝绝对路径、.. 上溯、符号链接
② 类型安全检查：只允许常规文件和目录
③ 不允许导入为 "default"（会覆盖 ~/.hermes）
④ 名称冲突检查
⑤ 临时目录提取后重命名到目标位置
```

---

## 七、Gateway 的 Profile 感知

### 7.1 每个 Profile 独立的 Gateway 进程

不同 Profile 可以同时运行各自的 Gateway，互不冲突。系统通过以下机制区分：

**进程查找**（`gateway.py:160-256`）：

```python
def find_gateway_pids(exclude_pids=None, all_profiles=False):
    # 查找命令行中包含 "hermes_cli.main --profile" 的进程
    # 匹配当前 Profile 的 Gateway：
    #   --profile work-agent 的进程只匹配 work-agent
    #   无 --profile 的进程只匹配 default
```

**服务名后缀**（`gateway.py:380-404`）：

```python
def _profile_suffix():
    # default Profile → 服务名 "hermes-gateway"
    # work-agent Profile → 服务名 "hermes-gateway-work-agent"
    # 自定义路径 → 服务名 "hermes-gateway-<8位hash>"
```

**systemd/launchd 服务定义**中自动注入 `--profile` 参数：

```python
def _profile_arg(hermes_home=None):
    # 如果 HERMES_HOME 指向 ~/.hermes/profiles/<name>/
    # 返回 "--profile <name>" 加入服务命令
    # 否则返回空字符串
```

### 7.2 多 Gateway 并行运行

理论上可以同时运行：

```
hermes gateway start                        # default Profile 的 Gateway
hermes -p work-agent gateway start          # work-agent 的 Gateway
hermes -p personal gateway start            # personal 的 Gateway
```

每个 Gateway 使用各自的 `gateway.pid`、`gateway_state.json`、配置和认证，完全隔离。

---

## 八、配置文件隔离详解

### 8.1 独立配置文件

每个 Profile 独立拥有以下配置：

| 文件 | 内容 | 隔离效果 |
|------|------|----------|
| `config.yaml` | 模型、provider、工具开关、委托配置、安全规则 | 不同 Agent 可以用不同模型和 API 密钥 |
| `.env` | API 密钥、环境变量 | 凭证完全隔离 |
| `SOUL.md` | Agent 人格、行为准则 | 不同身份和风格 |
| `memories/MEMORY.md` | Agent 的持久记忆 | 不同知识库 |
| `memories/USER.md` | 用户画像 | 不同用户理解 |

### 8.2 克隆时复制的文件

```python
# profiles.py:53-65
_CLONE_CONFIG_FILES = [
    "config.yaml",      # 模型/provider/工具配置
    ".env",              # API 密钥
    "SOUL.md",           # Agent 人格
]
_CLONE_SUBDIR_FILES = [
    "memories/MEMORY.md",    # Agent 记忆
    "memories/USER.md",      # 用户画像
]
```

### 8.3 技能隔离

每个 Profile 有独立的 `skills/` 目录。创建新 Profile 时通过子进程播种内置技能：

```python
# profiles.py:475-505
def seed_profile_skills(profile_dir, quiet=False):
    result = subprocess.run(
        [sys.executable, "-c",
         "from tools.skills_sync import sync_skills; r = sync_skills(quiet=True); ..."],
        env={**os.environ, "HERMES_HOME": str(profile_dir)},  # 注入 Profile 路径
        cwd=str(project_root),
        timeout=60,
    )
```

关键点：通过子进程 + 环境变量注入，确保技能播种使用 Profile 自己的路径。

---

## 九、安全设计

### 9.1 名称安全

- 正则校验防止路径注入：`^[a-z0-9][a-z0-9_-]{0,63}$`
- 保留名阻止滥用：`default`, `hermes`, `root`, `sudo`
- 子命令冲突检查：避免覆盖 `hermes chat` 等命令
- PATH 冲突检查：不允许覆盖系统命令

### 9.2 导出安全

- 默认 Profile 导出时排除 `auth.json`、`.env`（凭证不打包）
- 命名 Profile 导出时排除 `auth.json`、`.env`
- Tar 提取时防止路径穿越（拒绝 `..`、绝对路径、符号链接）

### 9.3 删除安全

- 禁止删除 `default` Profile
- 必须输入 Profile 名确认（或 `-y` 跳过）
- 删除前自动停止 Gateway 进程和服务

### 9.4 子进程隔离

每个 Profile 的子进程使用独立的 `HOME` 目录（`{HERMES_HOME}/home/`），防止：
- Git 凭证泄露
- SSH 密钥暴露
- GitHub CLI token 污染
- npm/pip 配置冲突

---

## 十、与其他系统对比

### 10.1 与 Docker 容器隔离对比

| 维度 | Hermes Profile | Docker 容器 |
|------|---------------|-------------|
| 隔离级别 | 文件系统路径隔离 | 进程/网络/文件系统完全隔离 |
| 开销 | 接近零（仅目录切换） | 需要容器运行时 |
| 并发 | 单时刻单 Profile | 可同时运行多个容器 |
| 切换速度 | 毫秒级 | 秒级（容器启动） |
| 适用场景 | 同机器多身份 | 多环境/多租户 |

### 10.2 与 Git Worktree 对比

Profile 的设计灵感类似 Git Worktree：

| 维度 | Hermes Profile | Git Worktree |
|------|---------------|-------------|
| 共享内容 | 源码/二进制 | .git 对象库 |
| 独立内容 | 配置/记忆/会话 | 工作区/索引 |
| 切换方式 | 环境变量 | 切换目录 |
| 并发使用 | 单进程 | 可同时修改 |

---

## 十一、限制与注意事项

### 11.1 当前限制

1. **不支持并发运行多个 Profile**：同一进程只能使用一个 Profile。要同时运行两个 Profile 的 Gateway，需要分别启动两个 `hermes gateway start` 进程。

2. **切换需要重启会话**：`hermes profile use` 只影响下次启动，当前会话不受影响。用户被告知需要 `/new` 或退出重进。

3. **克隆不会复制会话历史**：`--clone` 只复制配置和记忆文件，不复制 `sessions/` 目录。只有 `--clone-all` 才会完整复制。

4. **默认 Profile 没有独立目录**：`default` 就是 `~/.hermes/` 本身，不能被删除或重命名。导出 default 时需要特殊处理（排除基础设施文件）。

5. **Wrapper 依赖 PATH**：别名脚本放在 `~/.local/bin/`，需要用户确保该目录在 PATH 中。

### 11.2 适用场景

| 场景 | 配置方式 |
|------|----------|
| 工作/个人分离 | 两个 Profile，不同 SOUL.md、不同工具集 |
| 开发/生产环境 | 两个 Profile，不同 API 密钥、不同模型 |
| 测试新功能 | 临时 Profile，搞坏了直接删除 |
| 多语言 Agent | 不同 SOUL.md 设置不同语言风格 |
| 多用户共享机器 | 每个 Agent 独立记忆和配置 |

### 11.3 不适用的场景

- **需要同时运行多个 Agent**：Profile 是切换机制，不是并发机制
- **Agent 间需要通信**：Profile 之间完全隔离，没有消息通道
- **动态创建/销毁 Agent**：Profile 创建需要 shell 命令，不支持编程接口

---

## 十二、源码关键路径索引

| 功能 | 文件 | 关键行/函数 |
|------|------|------------|
| 启动时 Profile 解析 | `hermes_cli/main.py` | `_apply_profile_override()` 第 83 行 |
| Profile CRUD | `hermes_cli/profiles.py` | `create_profile()` 第 381 行 |
| Profile 删除 | `hermes_cli/profiles.py` | `delete_profile()` 第 508 行 |
| Profile 重命名 | `hermes_cli/profiles.py` | `rename_profile()` 第 940 行 |
| 粘性默认 | `hermes_cli/profiles.py` | `set_active_profile()` 第 701 行 |
| 导出 | `hermes_cli/profiles.py` | `export_profile()` 第 780 行 |
| 导入 | `hermes_cli/profiles.py` | `import_profile()` 第 875 行 |
| Wrapper 脚本 | `hermes_cli/profiles.py` | `create_wrapper_script()` 第 227 行 |
| HERMES_HOME 解析 | `hermes_constants.py` | `get_hermes_home()` 第 11 行 |
| 子进程 HOME | `hermes_constants.py` | `get_subprocess_home()` 第 114 行 |
| Gateway Profile 感知 | `hermes_cli/gateway.py` | `_profile_suffix()` 第 380 行 |
| Profile 子命令定义 | `hermes_cli/main.py` | 第 5850-5905 行 |
| Profile 子命令处理 | `hermes_cli/main.py` | 第 4182-4400 行 |
| Shell 自动补全 | `hermes_cli/profiles.py` | `generate_bash_completion()` 第 994 行 |
