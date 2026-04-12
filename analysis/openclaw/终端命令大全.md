# OpenClaw 终端命令大全

## 基础命令

### openclaw --help
显示OpenClaw主帮助信息

### openclaw --version
显示OpenClaw版本号

### openclaw --dev
使用开发模式（隔离状态在 ~/.openclaw-dev，默认网关端口19001）

### openclaw --profile &lt;name&gt;
使用命名配置文件（隔离状态在 ~/.openclaw-&lt;name&gt;）

### openclaw --log-level &lt;level&gt;
设置全局日志级别（silent|fatal|error|warn|info|debug|trace）

---

## 核心命令

### 1. gateway - 网关管理
```bash
openclaw gateway --help
```

**子命令：**
- `openclaw gateway run` - 前台运行WebSocket网关
- `openclaw gateway start` - 启动网关服务
- `openclaw gateway stop` - 停止网关服务
- `openclaw gateway restart` - 重启网关服务
- `openclaw gateway status` - 显示网关服务状态
- `openclaw gateway install` - 安装网关服务
- `openclaw gateway uninstall` - 卸载网关服务
- `openclaw gateway probe` - 探测网关可达性
- `openclaw gateway health` - 获取网关健康状态
- `openclaw gateway discover` - 通过Bonjour发现网关
- `openclaw gateway call` - 直接调用网关RPC方法
- `openclaw gateway usage-cost` - 获取使用成本汇总

**选项：**
- `--port &lt;port&gt;` - 指定网关端口
- `--bind &lt;mode&gt;` - 绑定模式（loopback|lan|tailnet|auto|custom）
- `--auth &lt;mode&gt;` - 认证模式（none|token|password|trusted-proxy）
- `--token &lt;token&gt;` - 共享令牌
- `--password &lt;password&gt;` - 密码认证
- `--tailscale &lt;mode&gt;` - Tailscale暴露模式（off|serve|funnel）
- `--force` - 强制杀死现有监听进程
- `--verbose` - 详细日志输出

---

### 2. agent - 代理运行
```bash
openclaw agent --help
```
运行一次代理回合

---

### 3. agents - 代理管理
```bash
openclaw agents --help
```
管理隔离代理（工作区、认证、路由）

---

### 4. approvals - 执行审批
```bash
openclaw approvals --help
```
管理执行审批（网关或节点主机）

---

### 5. browser - 浏览器管理
```bash
openclaw browser --help
```
管理OpenClaw专用浏览器（Chrome/Chromium）

---

### 6. channels - 聊天通道管理
```bash
openclaw channels --help
```
管理连接的聊天通道（Telegram、Discord等）

**常用操作：**
- `openclaw channels login --verbose` - 登录个人WhatsApp Web并显示QR码

---

### 7. config - 配置管理
```bash
openclaw config --help
```
非交互式配置助手（get/set/unset/file/validate）

**子命令：**
- `openclaw config get` - 获取配置值
- `openclaw config set` - 设置配置值
- `openclaw config unset` - 取消设置配置值
- `openclaw config file` - 配置文件操作
- `openclaw config validate` - 验证配置

---

### 8. configure - 交互式配置向导
```bash
openclaw configure
```
交互式设置向导，用于配置凭证、通道、网关和代理默认值

---

### 9. cron - 定时任务管理
```bash
openclaw cron --help
```
通过网关调度器管理定时任务

---

### 10. dashboard - 控制面板
```bash
openclaw dashboard
```
使用当前令牌打开控制面板UI

---

### 11. devices - 设备管理
```bash
openclaw devices --help
```
设备配对和令牌管理

---

### 12. directory - 目录查询
```bash
openclaw directory --help
```
查询支持的聊天通道的联系人和群组ID

---

### 13. dns - DNS助手
```bash
openclaw dns --help
```
广域发现的DNS助手（Tailscale + CoreDNS）

---

### 14. docs - 文档搜索
```bash
openclaw docs
```
搜索在线OpenClaw文档

---

### 15. doctor - 健康检查
```bash
openclaw doctor
```
网关和通道的健康检查 + 快速修复

---

### 16. health - 健康状态
```bash
openclaw health
```
从运行中的网关获取健康状态

---

### 17. logs - 日志查看
```bash
openclaw logs
```
通过RPC跟踪网关文件日志

---

### 18. memory - 内存管理
```bash
openclaw memory --help
```
搜索和重新索引内存文件

---

### 19. message - 消息管理
```bash
openclaw message --help
```
发送、读取和管理消息

**常用操作：**
- `openclaw message send --target &lt;target&gt; --message "Hi"` - 发送消息
- `openclaw message send --channel telegram --target @mychat --message "Hi"` - 通过Telegram机器人发送

---

### 20. models - 模型管理
```bash
openclaw models --help
```
发现、扫描和配置模型

---

### 21. node - 节点主机
```bash
openclaw node --help
```
运行和管理无头节点主机服务

---

### 22. nodes - 节点管理
```bash
openclaw nodes --help
```
管理网关拥有的节点配对和节点命令

---

### 23. onboard - 入职向导
```bash
openclaw onboard
```
网关、工作区和技能的交互式入职向导

---

### 24. pairing - 配对管理
```bash
openclaw pairing --help
```
安全DM配对（批准入站请求）

---

### 25. plugins - 插件管理
```bash
openclaw plugins --help
```
管理OpenClaw插件和扩展

---

### 26. qr - 二维码生成
```bash
openclaw qr
```
生成iOS配对QR/设置代码

---

### 27. reset - 重置配置
```bash
openclaw reset
```
重置本地配置/状态（保持CLI安装）

---

### 28. sandbox - 沙箱管理
```bash
openclaw sandbox --help
```
管理用于代理隔离的沙箱容器

---

### 29. secrets - 密钥管理
```bash
openclaw secrets --help
```
密钥运行时重新加载控制

---

### 30. security - 安全工具
```bash
openclaw security --help
```
安全工具和本地配置审计

---

### 31. sessions - 会话管理
```bash
openclaw sessions --help
```
列出存储的对话会话

---

### 32. setup - 初始化设置
```bash
openclaw setup
```
初始化本地配置和代理工作区

---

### 33. skills - 技能管理
```bash
openclaw skills --help
```
列出和检查可用的技能

---

### 34. status - 状态显示
```bash
openclaw status
```
显示通道健康状态和最近会话接收者

---

### 35. system - 系统管理
```bash
openclaw system --help
```
系统事件、心跳和状态

---

### 36. tui - 终端UI
```bash
openclaw tui
```
打开连接到网关的终端UI

---

### 37. uninstall - 卸载
```bash
openclaw uninstall
```
卸载网关服务 + 本地数据（CLI保留）

---

### 38. update - 更新管理
```bash
openclaw update --help
```
更新OpenClaw并检查更新通道状态

---

### 39. webhooks - Webhook管理
```bash
openclaw webhooks --help
```
Webhook助手和集成

---

### 40. acp - 代理控制协议
```bash
openclaw acp --help
```
代理控制协议工具

---

### 41. clawbot - 旧版命令别名
```bash
openclaw clawbot --help
```
旧版clawbot命令别名

---

### 42. completion - 补全脚本
```bash
openclaw completion
```
生成shell补全脚本

---

### 43. daemon - 网关服务（旧版别名）
```bash
openclaw daemon --help
```
网关服务（旧版别名）

---

### 44. help - 帮助
```bash
openclaw help
```
显示命令帮助

---

## 常用命令速查

### 日常使用
```bash
# 查看状态
openclaw status

# 打开控制面板
openclaw dashboard

# 查看日志
openclaw logs

# 健康检查
openclaw doctor

# 搜索文档
openclaw docs
```

### 网关管理
```bash
# 启动网关
openclaw gateway start

# 停止网关
openclaw gateway stop

# 重启网关
openclaw gateway restart

# 查看网关状态
openclaw gateway status

# 前台运行网关
openclaw gateway run
```

### 配置管理
```bash
# 交互式配置
openclaw configure

# 入职向导
openclaw onboard

# 初始化设置
openclaw setup
```

### 消息发送
```bash
# 发送消息
openclaw message send --target &lt;target&gt; --message "Hello"

# 通过特定通道发送
openclaw message send --channel telegram --target @mychat --message "Hello"
```

### 插件和技能
```bash
# 管理插件
openclaw plugins --help

# 查看技能
openclaw skills --help
```

### 安全和审计
```bash
# 安全工具
openclaw security --help

# 健康检查
openclaw doctor
```

---

## 获取更多帮助

任何命令都可以使用 `--help` 获取详细帮助：
```bash
openclaw &lt;command&gt; --help
```

例如：
```bash
openclaw gateway --help
openclaw message --help
openclaw config --help
```

---

## 官方文档

在线文档：https://docs.openclaw.ai/cli
