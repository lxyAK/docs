# Coze Studio 发版部署指南

## 1. 概述

Coze Studio 是一站式 AI Agent 开发工具，提供从开发到部署的完整 AI Agent 开发环境。本文档详细介绍 Coze Studio 开源版的发版部署流程。

### 技术栈
- **后端**: Golang (>= 1.23.4)
- **前端**: React + TypeScript
- **架构**: 微服务架构 + DDD
- **部署方式**: Docker + Docker Compose

---

## 2. 环境要求

### 最低系统要求
- CPU: 2 核
- 内存: 4 GB
- 磁盘: 至少 20 GB 可用空间

### 软件要求
- Docker (>= 20.10)
- Docker Compose (>= 2.0)
- Git

---

## 3. 部署前准备

### 3.1 获取源码

```bash
# 克隆代码仓库
git clone https://github.com/coze-dev/coze-studio.git
cd coze-studio
```

### 3.2 配置环境变量

复制环境变量模板文件：

```bash
cp docker/.env.example docker/.env
```

根据需要编辑 `docker/.env` 文件，主要配置项如下：

#### 3.2.1 服务器配置
```bash
# 监听地址（公网部署建议改为 0.0.0.0:8888）
export WEB_LISTEN_ADDR="127.0.0.1:8888"
export LISTEN_ADDR=":8888"
export SERVER_HOST="http://localhost:8888"
```

#### 3.2.2 数据库配置（MySQL）
```bash
export MYSQL_ROOT_PASSWORD=root
export MYSQL_DATABASE=opencoze
export MYSQL_USER=coze
export MYSQL_PASSWORD=coze123
export MYSQL_HOST=mysql
export MYSQL_PORT=3306
```

#### 3.2.3 Redis 配置
```bash
export REDIS_ADDR="redis:6379"
export REDIS_PASSWORD=""
```

#### 3.2.4 存储配置（MinIO）
```bash
export STORAGE_TYPE="minio"
export MINIO_ROOT_USER=minioadmin
export MINIO_ROOT_PASSWORD=minioadmin123
export MINIO_ENDPOINT="minio:9000"
```

#### 3.2.5 向量数据库配置（Milvus）
```bash
export VECTOR_STORE_TYPE="milvus"
export MILVUS_ADDR="milvus:19530"
```

#### 3.2.6 嵌入模型配置
支持多种嵌入模型提供商，选择其一配置：

**方式一：火山方舟 (Ark)**
```bash
export EMBEDDING_TYPE="ark"
export ARK_EMBEDDING_BASE_URL="https://ark.cn-beijing.volces.com/api/v3"
export ARK_EMBEDDING_MODEL="ep-xxxxx-xxxxxx"
export ARK_EMBEDDING_API_KEY="your-api-key"
export ARK_EMBEDDING_DIMS="2048"
```

**方式二：OpenAI**
```bash
export EMBEDDING_TYPE="openai"
export OPENAI_EMBEDDING_BASE_URL="https://api.openai.com/v1"
export OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
export OPENAI_EMBEDDING_API_KEY="sk-xxxxx"
export OPENAI_EMBEDDING_DIMS=1024
```

**方式三：Ollama**
```bash
export EMBEDDING_TYPE="ollama"
export OLLAMA_EMBEDDING_BASE_URL="http://localhost:11434"
export OLLAMA_EMBEDDING_MODEL="nomic-embed-text"
export OLLAMA_EMBEDDING_DIMS="768"
```

#### 3.2.7 对话模型配置
```bash
export BUILTIN_CM_TYPE="ark"
export BUILTIN_CM_ARK_API_KEY="your-api-key"
export BUILTIN_CM_ARK_MODEL="ep-xxxxx-xxxxxx"
export BUILTIN_CM_ARK_BASE_URL="https://ark.cn-beijing.volces.com/api/v3"
```

#### 3.2.8 用户注册配置
```bash
# 禁用用户注册（可选）
export DISABLE_USER_REGISTRATION=""
# 允许注册的邮箱白名单（逗号分隔）
export ALLOW_REGISTRATION_EMAIL=""
```

#### 3.2.9 安全配置（生产环境必改）
```bash
# 插件加密密钥（必须修改，长度 16/24/32 字节）
export PLUGIN_AES_AUTH_SECRET='your-32-byte-secret-key-here'
export PLUGIN_AES_STATE_SECRET='your-32-byte-secret-key-here'
export PLUGIN_AES_OAUTH_TOKEN_SECRET='your-32-byte-secret-key-here'
```

---

## 4. 快速部署（Docker Compose）

### 4.1 使用 Makefile 一键部署（推荐）

```bash
# 进入项目目录
cd coze-studio

# 一键启动（macOS/Linux）
make web

# Windows 系统手动执行
cp ./docker/.env.example ./docker/.env
docker compose -f ./docker/docker-compose.yml up -d
```

### 4.2 手动使用 Docker Compose 部署

```bash
# 进入 docker 目录
cd coze-studio/docker

# 启动所有服务
docker compose up -d

# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f
```

### 4.3 部署验证

等待服务启动完成（首次启动可能需要 5-10 分钟），当看到以下提示表示启动成功：
```
Container coze-server Started
```

---

## 5. 初始化配置

### 5.1 注册账号

访问 `http://localhost:8888/sign`，输入用户名和密码完成注册。

### 5.2 配置模型

1. 访问模型管理页面：`http://localhost:8888/admin/#model-management`
2. 点击"新增模型"
3. 填写模型配置信息：
   - 模型协议：ark / openai / ollama 等
   - 模型名称：显示名称
   - 模型 ID：实际调用的模型 ID
   - API Key：对应服务商的 API Key
   - Base URL：API 基础地址

### 5.3 验证部署

访问 `http://localhost:8888/`，确认可以正常使用 Coze Studio。

---

## 6. 常用运维命令

### 6.1 服务管理

```bash
# 启动服务
make web
# 或
docker compose -f docker/docker-compose.yml up -d

# 停止服务
make down_web
# 或
docker compose -f docker/docker-compose.yml down

# 重启服务
make down_web && make web

# 查看服务状态
docker compose -f docker/docker-compose.yml ps
```

### 6.2 日志查看

```bash
# 查看所有服务日志
docker compose -f docker/docker-compose.yml logs -f

# 查看特定服务日志
docker compose -f docker/docker-compose.yml logs -f coze-server
docker compose -f docker/docker-compose.yml logs -f mysql
docker compose -f docker/docker-compose.yml logs -f redis
```

### 6.3 数据管理

```bash
# 清理所有数据（谨慎使用！）
make clean
# 或
docker compose -f docker/docker-compose.yml down -v
rm -rf ./docker/data
```

---

## 7. 生产环境部署建议

### 7.1 安全加固

1. **修改默认密码**
   - MySQL root 密码
   - MySQL 用户密码
   - MinIO 管理员密码
   - 插件加密密钥

2. **网络安全**
   - 使用反向代理（Nginx）
   - 配置 HTTPS
   - 限制端口访问
   - 配置防火墙规则

3. **用户注册控制**
   ```bash
   # 禁用公开注册
   export DISABLE_USER_REGISTRATION="true"
   # 或使用白名单
   export ALLOW_REGISTRATION_EMAIL="user1@example.com,user2@example.com"
   ```

### 7.2 数据持久化

确保以下数据目录已持久化：
- MySQL 数据：`docker/data/mysql`
- MinIO 数据：`docker/data/minio`
- Milvus 数据：`docker/data/milvus`
- Elasticsearch 数据：`docker/data/elasticsearch`

### 7.3 备份策略

定期备份以下内容：
1. MySQL 数据库
2. MinIO 对象存储
3. 环境配置文件

### 7.4 监控与告警

建议配置：
- 容器健康检查
- 资源使用率监控（CPU、内存、磁盘）
- 日志聚合与分析
- 告警通知

---

## 8. 常见问题排查

### 8.1 服务启动失败

**检查端口占用**
```bash
netstat -tulpn | grep -E '8888|3306|6379'
```

**查看详细日志**
```bash
docker compose -f docker/docker-compose.yml logs coze-server
```

### 8.2 数据库连接失败

检查 MySQL 容器状态：
```bash
docker compose -f docker/docker-compose.yml ps mysql
docker compose -f docker/docker-compose.yml logs mysql
```

### 8.3 模型配置问题

确认：
1. API Key 是否正确
2. Base URL 是否可访问
3. 模型 ID 是否正确
4. 网络连接是否正常

### 8.4 更多问题

参考官方 Wiki：https://github.com/coze-dev/coze-studio/wiki/9.-常见问题

---

## 9. 高级部署选项

### 9.1 使用 OceanBase 替代 MySQL

项目支持 OceanBase 作为数据库：

```bash
# 使用 OceanBase 配置
cp docker/.env.debug.example docker/.env
docker compose -f docker/docker-compose-oceanbase.yml up -d
```

### 9.2 Kubernetes 部署

项目提供 Helm Chart 支持：

```bash
cd helm/charts/opencoze
# 查看 values.yaml 配置
# 使用 Helm 部署
helm install coze-studio .
```

### 9.3 开发环境部署

如需本地开发：

```bash
# 启动开发环境
make debug

# 仅启动中间件
make middleware

# 构建前端
make fe

# 构建并启动后端
make server
```

---

## 10. 附录

### 10.1 服务端口说明

| 服务 | 端口 | 说明 |
|------|------|------|
| Coze Server | 8888 | 主服务端口 |
| MySQL | 3306 | 数据库 |
| Redis | 6379 | 缓存 |
| MinIO | 9000 | 对象存储 |
| Milvus | 19530 | 向量数据库 |
| Elasticsearch | 9200 | 搜索引擎 |
| NSQ | 4150, 4151 | 消息队列 |

### 10.2 目录结构

```
coze-studio/
├── backend/           # 后端代码
├── frontend/          # 前端代码
├── docker/            # Docker 相关配置
│   ├── docker-compose.yml
│   ├── .env.example
│   └── volumes/       # 数据卷
├── helm/              # Kubernetes Helm Chart
├── scripts/           # 部署脚本
└── Makefile           # 构建脚本
```

### 10.3 参考资源

- 官方文档：https://github.com/coze-dev/coze-studio
- Wiki：https://github.com/coze-dev/coze-studio/wiki
- 扣子开发平台：https://www.coze.cn/home
- 社区交流群：飞书群 / Discord / Telegram

---

## 11. 版本更新流程

### 11.0 更新前检查清单

在执行版本更新前，请确认以下事项：

- [ ] 已阅读最新的 Release Notes（查看 GitHub Releases）
- [ ] 已确认当前版本与目标版本的兼容性
- [ ] 已安排维护时间窗口（建议低峰期）
- [ ] 已通知相关用户系统将进行维护
- [ ] 已准备好回滚方案
- [ ] 磁盘空间充足（至少需要当前数据量的 2 倍）

---

### 11.1 详细更新步骤

#### 步骤 1：进入项目目录并检查当前状态

```bash
# 进入项目目录
cd /path/to/coze-studio

# 检查当前 git 状态
git status

# 查看当前版本
git log -1 --oneline

# 查看当前分支
git branch
```

#### 步骤 2：查看远程版本信息

```bash
# 获取远程最新信息
git fetch origin

# 查看远程分支最新提交
git log origin/main -1 --oneline

# 查看版本差异（可选，了解变更内容）
git log HEAD..origin/main --oneline
git diff HEAD..origin/main --stat
```

#### 步骤 3：备份配置文件

```bash
# 备份当前环境变量配置
cp docker/.env docker/.env.backup.$(date +%Y%m%d_%H%M%S)

# 备份 docker-compose.yml（如果有自定义修改）
cp docker/docker-compose.yml docker/docker-compose.yml.backup.$(date +%Y%m%d_%H%M%S)

# 确认备份文件已创建
ls -lh docker/*.backup.*
```

#### 步骤 4：备份数据（重要！）

##### 4.1 停止服务以确保数据一致性

```bash
# 停止所有服务
make down_web
# 或
docker compose -f docker/docker-compose.yml down

# 确认所有容器已停止
docker compose -f docker/docker-compose.yml ps
```

##### 4.2 备份 MySQL 数据库

```bash
# 方式一：直接备份数据目录（推荐）
BACKUP_DIR="./backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

# 备份 MySQL 数据
cp -r docker/data/mysql $BACKUP_DIR/mysql

# 方式二：使用 mysqldump 导出 SQL（如果 MySQL 容器还在运行）
# docker compose -f docker/docker-compose.yml exec mysql mysqldump -ucoze -pcoze123 opencoze > $BACKUP_DIR/opencoze.sql
```

##### 4.3 备份 MinIO 对象存储

```bash
# 备份 MinIO 数据
cp -r docker/data/minio $BACKUP_DIR/minio
```

##### 4.4 备份 Milvus 向量数据库

```bash
# 备份 Milvus 数据
cp -r docker/data/milvus $BACKUP_DIR/milvus
```

##### 4.5 备份 Elasticsearch 数据

```bash
# 备份 Elasticsearch 数据
cp -r docker/data/elasticsearch $BACKUP_DIR/elasticsearch
```

##### 4.6 验证备份完整性

```bash
# 查看备份目录大小
du -sh $BACKUP_DIR

# 列出备份内容
ls -lh $BACKUP_DIR

# 记录备份位置
echo "备份已保存到: $(realpath $BACKUP_DIR)"
```

#### 步骤 5：拉取最新代码

```bash
# 确保在 main 分支上
git checkout main

# 拉取最新代码
git pull origin main

# 确认拉取成功
git log -1 --oneline
```

**注意**：如果有本地修改，git pull 可能会失败。处理方式：

```bash
# 方式一：暂存本地修改
git stash
git pull origin main
git stash pop  # 更新完成后恢复修改（需手动解决冲突）

# 方式二：丢弃本地修改（谨慎使用！）
git reset --hard origin/main
git pull origin main
```

#### 步骤 6：检查并合并配置文件变更

```bash
# 查看新的环境变量模板与当前配置的差异
diff docker/.env.example docker/.env.backup.* | head -50

# 或使用 vimdiff 详细对比
# vimdiff docker/.env.example docker/.env

# 复制新的环境变量模板
cp docker/.env.example docker/.env

# 从备份中恢复自定义配置
# 打开 docker/.env，参考 docker/.env.backup.* 填入你的配置
# 重点检查：
# - 数据库密码
# - API Keys
# - 模型配置
# - 存储配置
# - 安全密钥

# 编辑配置文件
vim docker/.env
```

**配置迁移检查清单**：
- [ ] MySQL 配置（MYSQL_ROOT_PASSWORD, MYSQL_PASSWORD）
- [ ] MinIO 配置（MINIO_ROOT_PASSWORD）
- [ ] 嵌入模型配置（EMBEDDING_TYPE, ARK_EMBEDDING_* 等）
- [ ] 对话模型配置（BUILTIN_CM_*）
- [ ] 安全密钥（PLUGIN_AES_*）
- [ ] 用户注册配置（DISABLE_USER_REGISTRATION）
- [ ] 监听地址（WEB_LISTEN_ADDR）

#### 步骤 7：拉取最新 Docker 镜像

```bash
# 拉取最新镜像
docker compose -f docker/docker-compose.yml pull

# 查看镜像版本
docker images | grep coze
```

#### 步骤 8：启动新版本

```bash
# 启动所有服务
make web
# 或
docker compose -f docker/docker-compose.yml up -d

# 查看启动日志（实时跟踪）
docker compose -f docker/docker-compose.yml logs -f

# 按 Ctrl+C 退出日志查看
```

**预期启动时间**：首次启动新版本可能需要 5-15 分钟，取决于：
- 网络速度（拉取镜像）
- 机器性能
- 数据量大小（数据库迁移）

#### 步骤 9：数据库迁移验证

项目使用 Atlas 进行数据库迁移，通常会自动执行。

```bash
# 查看 coze-server 日志，确认迁移是否成功
docker compose -f docker/docker-compose.yml logs coze-server | grep -i "migrate"

# 或搜索 "migration"
docker compose -f docker/docker-compose.yml logs coze-server | grep -i "migration"

# 如果迁移失败，手动执行迁移
make sync_db

# 查看迁移日志
docker compose -f docker/docker-compose.yml logs --tail=100 coze-server
```

#### 步骤 10：验证服务状态

```bash
# 检查所有容器状态
docker compose -f docker/docker-compose.yml ps

# 预期输出：所有容器状态应为 "Up" 或 "Up (healthy)"

# 检查各服务健康状态
echo "=== 服务健康检查 ==="

# 检查 Coze Server
curl -s http://localhost:8888/health || echo "Coze Server 可能未完全启动"

# 检查 MySQL
docker compose -f docker/docker-compose.yml exec mysql mysqladmin -ucoze -pcoze123 ping

# 检查 Redis
docker compose -f docker/docker-compose.yml exec redis redis-cli ping

# 检查 MinIO
curl -s http://localhost:9000/minio/health/live || echo "MinIO 健康检查端点可能不同"
```

#### 步骤 11：功能验证

**11.1 访问 Web 界面**
- 打开浏览器访问：`http://localhost:8888/`
- 确认页面正常加载
- 检查版本号（如有显示）

**11.2 登录验证**
- 使用现有账号登录
- 确认登录功能正常

**11.3 核心功能测试**
- [ ] 创建一个测试智能体
- [ ] 测试对话功能
- [ ] 上传一个测试文件到知识库
- [ ] 创建一个简单工作流并测试运行
- [ ] 测试模型调用

**11.4 数据验证**
- [ ] 确认原有智能体仍在
- [ ] 确认原有知识库数据完整
- [ ] 确认历史对话记录存在

#### 步骤 12：清理旧镜像（可选）

```bash
# 查看当前镜像
docker images | grep coze

# 删除旧版本镜像（谨慎操作！确认新镜像正常运行后再执行）
# docker rmi <旧镜像ID>

# 或清理所有悬空镜像
docker image prune -f
```

---

### 11.2 数据库迁移详解

#### 11.2.1 Atlas 迁移工具说明

Coze Studio 使用 Atlas 作为数据库迁移工具，迁移文件位于：
```
docker/atlas/migrations/
```

#### 11.2.2 自动迁移流程

1. coze-server 启动时自动检查数据库版本
2. 与迁移文件对比，确定需要执行的迁移
3. 自动执行未应用的迁移
4. 记录迁移状态到 `atlas_schema_revisions` 表

#### 11.2.3 手动迁移步骤

如果自动迁移失败，可以手动执行：

```bash
# 确保中间件正在运行
make middleware

# 执行数据库迁移
make sync_db

# 查看迁移日志
docker compose -f docker/docker-compose-debug.yml --profile mysql-setup logs
```

#### 11.2.4 迁移失败处理

**常见问题 1：迁移锁超时**
```bash
# 检查 Atlas 锁表
docker compose -f docker/docker-compose.yml exec mysql mysql -ucoze -pcoze123 opencoze -e "SELECT * FROM atlas_schema_revisions;"

# 如果需要，手动清理锁（谨慎！）
# docker compose -f docker/docker-compose.yml exec mysql mysql -ucoze -pcoze123 opencoze -e "DELETE FROM atlas_schema_revisions WHERE ...;"
```

**常见问题 2：数据冲突**
- 查看详细错误日志
- 根据错误信息手动处理冲突数据
- 重新执行迁移

---

### 11.3 回滚流程

如果更新后出现严重问题，需要回滚到之前版本：

#### 11.3.1 回滚前确认

- [ ] 已确认问题无法快速修复
- [ ] 已备份更新后产生的新数据（如有）
- [ ] 准备好回滚时间窗口

#### 11.3.2 详细回滚步骤

```bash
# 1. 停止新版本服务
make down_web

# 2. 恢复数据（使用更新前的备份）
BACKUP_TO_RESTORE="./backups/20260327_143000"  # 替换为实际备份目录

# 确认备份目录存在
ls -lh $BACKUP_TO_RESTORE

# 清理当前数据目录
rm -rf docker/data/*

# 恢复 MySQL
cp -r $BACKUP_TO_RESTORE/mysql docker/data/

# 恢复 MinIO
cp -r $BACKUP_TO_RESTORE/minio docker/data/

# 恢复 Milvus
cp -r $BACKUP_TO_RESTORE/milvus docker/data/

# 恢复 Elasticsearch
cp -r $BACKUP_TO_RESTORE/elasticsearch docker/data/

# 3. 恢复代码到之前版本
# 查看 git 历史，找到更新前的 commit
git log --oneline -10

# 回滚到指定 commit
git reset --hard <commit-hash>

# 或回滚到上一个版本
git reset --hard HEAD~1

# 4. 恢复配置文件
cp docker/.env.backup.* docker/.env

# 5. 拉取对应版本的 Docker 镜像
# 检查 docker-compose.yml 中的镜像标签
grep "image:" docker/docker-compose.yml

# 拉取旧版本镜像
docker compose -f docker/docker-compose.yml pull

# 6. 启动旧版本
make web

# 7. 验证回滚成功
# 按照 11.1 步骤 10-11 进行验证
```

---

### 11.4 更新后检查清单

更新完成后，请确认以下事项：

- [ ] 所有容器状态正常（Up）
- [ ] 日志中无 ERROR 级别错误
- [ ] 可以正常访问 Web 界面
- [ ] 可以正常登录
- [ ] 原有数据完整（智能体、知识库、对话记录）
- [ ] 模型调用正常
- [ ] 文件上传/下载功能正常
- [ ] 工作流执行正常
- [ ] 已清理临时文件和旧镜像（可选）
- [ ] 已更新文档记录当前版本
- [ ] 已通知用户更新完成

---

### 11.5 常见更新问题及解决方案

#### 问题 1：git pull 冲突

**症状**：
```
error: Your local changes to the following files would be overwritten by merge
```

**解决方案**：
```bash
# 查看哪些文件有冲突
git status

# 方式一：暂存修改
git stash
git pull
git stash pop  # 然后手动解决冲突

# 方式二：备份后重置
cp docker/.env docker/.env.local
git reset --hard origin/main
git pull
# 然后从 docker/.env.local 恢复配置
```

#### 问题 2：容器启动失败，端口被占用

**症状**：
```
Bind for 0.0.0.0:8888 failed: port is already allocated
```

**解决方案**：
```bash
# 查找占用端口的进程
netstat -tulpn | grep 8888
# 或
lsof -i :8888

# 停止旧容器（如果还在运行）
docker ps -a | grep coze
docker stop <container-id>
docker rm <container-id>

# 或者修改 docker/.env 中的端口配置
```

#### 问题 3：数据库迁移失败

**症状**：
- coze-server 日志中出现迁移错误
- 服务无法完全启动

**解决方案**：
```bash
# 1. 查看详细错误日志
docker compose -f docker/docker-compose.yml logs coze-server --tail=200

# 2. 尝试手动迁移
make sync_db

# 3. 如果还是失败，考虑从备份恢复后重试
# 或在 GitHub Issues 中搜索类似问题
```

#### 问题 4：配置文件格式错误

**症状**：
- 容器启动立即退出
- 日志中显示环境变量解析错误

**解决方案**：
```bash
# 检查 .env 文件格式
# 确保没有特殊字符未转义
# 确保没有多余的空格

# 使用备份的 .env 文件对比
diff docker/.env docker/.env.backup.*

# 或者从 .env.example 重新开始，逐步添加配置
```

#### 问题 5：模型配置丢失

**症状**：
- 更新后之前配置的模型不见了
- 无法选择模型

**解决方案**：
```bash
# 1. 检查 .env 中的模型配置是否正确
grep -E "MODEL_|BUILTIN_CM_" docker/.env

# 2. 如果配置正确但模型不显示，通过 Web UI 重新添加
# 访问 http://localhost:8888/admin/#model-management

# 3. 或者从数据库备份中恢复（如果需要保留历史记录）
```

---

### 11.6 大版本升级特别说明

对于跨大版本升级（如 v0.4.x → v0.5.x），建议：

1. **先在测试环境验证**
   - 在测试服务器上执行完整升级流程
   - 验证所有功能正常
   - 记录遇到的问题和解决方案

2. **考虑增量升级**
   - 如果跨度太大，考虑分多次小版本升级
   - 每次升级后充分验证

3. **预留更长的维护窗口**
   - 大版本升级可能需要更长时间
   - 建议预留 2-4 小时

4. **详细阅读 Release Notes**
   - 重点关注 Breaking Changes
   - 注意配置文件变更
   - 了解数据迁移要求

---

**文档版本**: 2.0  
**最后更新**: 2026-03-27  
**维护者**: 教授 🧑‍💻
