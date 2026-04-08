# Coze Studio 仅后端部署指南

## 概述

本文档说明如何只部署 Coze Studio 的后端服务（coze-server），不部署前端 Web 界面。

适用于以下场景：
- 已有自定义前端，只需接入 Coze Studio 后端 API
- 通过 API 直接调用 Coze Studio 功能
- 开发环境中只需要后端服务进行调试

---

## 部署架构

### 必需组件（必须全部启动）

1. **中间件服务**
   - MySQL：数据库
   - Redis：缓存
   - Elasticsearch：搜索引擎
   - MinIO：对象存储
   - Milvus：向量数据库
   - etcd：Milvus 依赖
   - NSQ：消息队列

2. **后端服务**
   - coze-server：核心后端 API 服务

### 可选组件（不需要启动）
- coze-web：前端 Web 界面（Nginx）

---

## 部署步骤

### 方法一：使用 Docker Compose（推荐）

#### 步骤 1：准备配置文件

```bash
# 进入项目目录
cd coze-studio

# 复制环境变量配置
cp docker/.env.example docker/.env

# 编辑配置文件，根据需要修改
vim docker/.env
```

**关键配置项**：
```bash
# 后端监听地址（改为 0.0.0.0 允许外部访问）
export LISTEN_ADDR=":8888"
export WEB_LISTEN_ADDR="0.0.0.0:8888"

# 数据库配置
export MYSQL_ROOT_PASSWORD=your_password
export MYSQL_PASSWORD=your_password

# MinIO 配置
export MINIO_ROOT_PASSWORD=your_password

# 模型配置（必须配置）
export EMBEDDING_TYPE="ark"
export ARK_EMBEDDING_API_KEY="your-api-key"
export ARK_EMBEDDING_MODEL="your-model"
export ARK_EMBEDDING_BASE_URL="your-base-url"

export BUILTIN_CM_TYPE="ark"
export BUILTIN_CM_ARK_API_KEY="your-api-key"
export BUILTIN_CM_ARK_MODEL="your-model"
export BUILTIN_CM_ARK_BASE_URL="your-base-url"
```

#### 步骤 2：创建自定义 docker-compose 文件

创建一个仅包含后端和中间件的 docker-compose 文件：

```bash
cd coze-studio/docker

# 创建自定义配置文件
cat > docker-compose-backend-only.yml << 'EOF'
name: coze-studio-backend
x-env-file: &env_file
  - .env

services:
  mysql:
    image: mysql:8.4.5
    container_name: coze-mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-root}
      MYSQL_DATABASE: ${MYSQL_DATABASE:-opencoze}
      MYSQL_USER: ${MYSQL_USER:-coze}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:-coze123}
    env_file: *env_file
    ports:
      - '3306:3306'  # 暴露 MySQL 端口（可选）
    volumes:
      - ./data/mysql:/var/lib/mysql
      - ./volumes/mysql/schema.sql:/docker-entrypoint-initdb.d/init.sql
      - ./atlas/opencoze_latest_schema.hcl:/opencoze_latest_schema.hcl:ro
    entrypoint:
      - bash
      - -c
      - |
        /usr/local/bin/docker-entrypoint.sh mysqld --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci &
        MYSQL_PID=$$!

        echo 'Waiting for MySQL to start...'
        until mysqladmin ping -h localhost -u root -p$${MYSQL_ROOT_PASSWORD} --silent 2>/dev/null; do
          echo 'MySQL is starting...'
          sleep 2
        done

        echo 'Waiting for workflow_version table to exist...'
        while true; do
          if mysql -h localhost -u root -p$${MYSQL_ROOT_PASSWORD} $${MYSQL_DATABASE} -e "SHOW TABLES LIKE 'workflow_version';" 2>/dev/null | grep -q "workflow_version"; then
            echo 'Found workflow_version table, continuing...'
            break
          else
            echo 'workflow_version table not found, retrying in 2 seconds...'
            sleep 2
          fi
        done

        echo 'MySQL is ready, installing Atlas CLI...'

        if ! command -v atlas >/dev/null 2>&1; then
          echo 'Installing Atlas CLI...'
          curl -sSf https://atlasgo.sh | sh -s -- -y --community
          export PATH=$$PATH:/root/.local/bin
        else
          echo 'Atlas CLI already installed'
        fi

        if [ -f '/opencoze_latest_schema.hcl' ]; then
          echo 'Running Atlas migrations...'
          ATLAS_URL="mysql://$${MYSQL_USER}:$${MYSQL_PASSWORD}@localhost:3306/$${MYSQL_DATABASE}"
          atlas schema apply -u "$ATLAS_URL" --to "file:///opencoze_latest_schema.hcl" --exclude "atlas_schema_revisions,table_*" --auto-approve
          echo 'Atlas migrations completed successfully'
        else
          echo 'No migrations found'
        fi
        wait $$MYSQL_PID
    healthcheck:
      test:
        [
          'CMD',
          'mysqladmin',
          'ping',
          '-h',
          'localhost',
          '-u$${MYSQL_USER}',
          '-p$${MYSQL_PASSWORD}',
        ]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - coze-network

  redis:
    image: bitnamilegacy/redis:8.0
    container_name: coze-redis
    restart: always
    user: root
    privileged: true
    env_file: *env_file
    environment:
      - REDIS_AOF_ENABLED=${REDIS_AOF_ENABLED:-no}
      - REDIS_PORT_NUMBER=${REDIS_PORT_NUMBER:-6379}
      - REDIS_IO_THREADS=${REDIS_IO_THREADS:-4}
      - ALLOW_EMPTY_PASSWORD=${ALLOW_EMPTY_PASSWORD:-yes}
    ports:
      - '6379:6379'  # 暴露 Redis 端口（可选）
    volumes:
      - ./data/bitnami/redis:/bitnami/redis/data:rw,Z
    command: >
      bash -c "
        /opt/bitnami/scripts/redis/setup.sh
        chown -R redis:redis /bitnami/redis/data
        chmod g+s /bitnami/redis/data
        exec /opt/bitnami/scripts/redis/entrypoint.sh /opt/bitnami/scripts/redis/run.sh
      "
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s
      timeout: 10s
      retries: 10
      start_period: 10s
    networks:
      - coze-network

  elasticsearch:
    image: bitnamilegacy/elasticsearch:8.18.0
    container_name: coze-elasticsearch
    restart: always
    user: root
    privileged: true
    env_file: *env_file
    environment:
      - TEST=1
    ports:
      - '9200:9200'  # 暴露 ES 端口（可选）
    volumes:
      - ./data/bitnami/elasticsearch:/bitnami/elasticsearch/data
      - ./volumes/elasticsearch/elasticsearch.yml:/opt/bitnami/elasticsearch/config/my_elasticsearch.yml
      - ./volumes/elasticsearch/analysis-smartcn.zip:/opt/bitnami/elasticsearch/analysis-smartcn.zip:rw,Z
      - ./volumes/elasticsearch/setup_es.sh:/setup_es.sh
      - ./volumes/elasticsearch/es_index_schema:/es_index_schema
    healthcheck:
      test:
        [
          'CMD-SHELL',
          'curl -f http://localhost:9200 && [ -f /tmp/es_plugins_ready ] && [ -f /tmp/es_init_complete ]',
        ]
      interval: 5s
      timeout: 10s
      retries: 10
      start_period: 10s
    networks:
      - coze-network
    command: >
      bash -c "
        /opt/bitnami/scripts/elasticsearch/setup.sh
        chown -R elasticsearch:elasticsearch /bitnami/elasticsearch/data
        chmod g+s /bitnami/elasticsearch/data
        mkdir -p /bitnami/elasticsearch/plugins;
        echo 'Installing smartcn plugin...';
        if [ ! -d /opt/bitnami/elasticsearch/plugins/analysis-smartcn ]; then
          echo 'Copying smartcn plugin...';
          cp /opt/bitnami/elasticsearch/analysis-smartcn.zip /tmp/analysis-smartcn.zip
          elasticsearch-plugin install file:///tmp/analysis-smartcn.zip
          if [[ \"$$?\" != \"0\" ]]; then
            echo 'Plugin installation failed, exiting operation';
            rm -rf /opt/bitnami/elasticsearch/plugins/analysis-smartcn
            exit 1;
          fi;
          rm -f /tmp/analysis-smartcn.zip;
        fi;
        touch /tmp/es_plugins_ready;
        echo 'Plugin installation successful, marker file created';
        (
          echo 'Waiting for Elasticsearch to be ready...'
          until curl -s -f http://localhost:9200/_cat/health >/dev/null 2>&1; do
            echo 'Elasticsearch not ready, waiting...'
            sleep 2
          done
          echo 'Elasticsearch is ready!'
          echo 'Running Elasticsearch initialization...'
          sed 's/\r$$//' /setup_es.sh > /setup_es_fixed.sh
          chmod +x /setup_es_fixed.sh
          /setup_es_fixed.sh --index-dir /es_index_schema
          touch /tmp/es_init_complete
          echo 'Elasticsearch initialization completed successfully!'
        ) &
        exec /opt/bitnami/scripts/elasticsearch/entrypoint.sh /opt/bitnami/scripts/elasticsearch/run.sh
        echo -e \"⏳ Adjusting Elasticsearch disk watermark settings...\"
      "

  minio:
    image: minio/minio:RELEASE.2025-06-13T11-33-47Z-cpuv1
    container_name: coze-minio
    user: root
    privileged: true
    restart: always
    env_file: *env_file
    ports:
      - '9000:9000'  # 暴露 MinIO API 端口
      - '9001:9001'  # 暴露 MinIO Console 端口（可选）
    volumes:
      - ./data/minio:/data
      - ./volumes/minio/default_icon/:/default_icon
      - ./volumes/minio/official_plugin_icon/:/official_plugin_icon
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-minioadmin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:-minioadmin123}
      MINIO_DEFAULT_BUCKETS: ${STORAGE_BUCKET:-opencoze},${MINIO_DEFAULT_BUCKETS:-milvus}
    entrypoint:
      - /bin/sh
      - -c
      - |
        (
          until (/usr/bin/mc alias set localminio http://localhost:9000 $${MINIO_ROOT_USER} $${MINIO_ROOT_PASSWORD}) do
            echo \"Waiting for MinIO to be ready...\"
            sleep 1
          done
          /usr/bin/mc mb --ignore-existing localminio/$${STORAGE_BUCKET}
          /usr/bin/mc cp --recursive /default_icon/ localminio/$${STORAGE_BUCKET}/default_icon/
          /usr/bin/mc cp --recursive /official_plugin_icon/ localminio/$${STORAGE_BUCKET}/official_plugin_icon/
          echo \"MinIO initialization complete.\"
        ) &
        exec minio server /data --console-address \":9001\"
    healthcheck:
      test:
        [
          'CMD-SHELL',
          '/usr/bin/mc alias set health_check http://localhost:9000 ${MINIO_ROOT_USER} ${MINIO_ROOT_PASSWORD} && /usr/bin/mc ready health_check',
        ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    networks:
      - coze-network

  etcd:
    image: bitnamilegacy/etcd:3.5
    container_name: coze-etcd
    user: root
    restart: always
    privileged: true
    env_file: *env_file
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
      - ALLOW_NONE_AUTHENTICATION=yes
    ports:
      - '2379:2379'  # 暴露 etcd 端口（可选）
    volumes:
      - ./data/bitnami/etcd:/bitnami/etcd:rw,Z
      - ./volumes/etcd/etcd.conf.yml:/opt/bitnami/etcd/conf/etcd.conf.yml:ro,Z
    command: >
      bash -c "
        /opt/bitnami/scripts/etcd/setup.sh
        chown -R etcd:etcd /bitnami/etcd
        chmod g+s /bitnami/etcd
        exec /opt/bitnami/scripts/etcd/entrypoint.sh /opt/bitnami/scripts/etcd/run.sh
      "
    healthcheck:
      test: ['CMD', 'etcdctl', 'endpoint', 'health']
      interval: 5s
      timeout: 10s
      retries: 10
      start_period: 10s
    networks:
      - coze-network

  milvus:
    container_name: coze-milvus
    image: milvusdb/milvus:v2.5.10
    user: root
    privileged: true
    restart: always
    env_file: *env_file
    command: >
      bash -c "
        chown -R root:root /var/lib/milvus
        chmod g+s /var/lib/milvus
        exec milvus run standalone
      "
    security_opt:
      - seccomp:unconfined
    environment:
      ETCD_ENDPOINTS: etcd:2379
      MINIO_ADDRESS: minio:9000
      MINIO_BUCKET_NAME: ${MINIO_DEFAULT_BUCKETS:-milvus}
      MINIO_ACCESS_KEY_ID: ${MINIO_ROOT_USER:-minioadmin}
      MINIO_SECRET_ACCESS_KEY: ${MINIO_ROOT_PASSWORD:-minioadmin123}
      MINIO_USE_SSL: false
      LOG_LEVEL: debug
    volumes:
      - ./data/milvus:/var/lib/milvus:rw,Z
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:9091/healthz']
      interval: 5s
      timeout: 10s
      retries: 10
      start_period: 10s
    ports:
      - '19530:19530'  # 暴露 Milvus 端口（可选）
      - '9091:9091'    # 暴露 Milvus 管理端口（可选）
    depends_on:
      etcd:
        condition: service_healthy
      minio:
        condition: service_healthy
    networks:
      - coze-network

  nsqlookupd:
    image: nsqio/nsq:v1.2.1
    container_name: coze-nsqlookupd
    command: /nsqlookupd
    restart: always
    ports:
      - '4160:4160'  # 暴露 NSQ Lookup 端口（可选）
      - '4161:4161'  # 暴露 NSQ Lookup HTTP 端口（可选）
    networks:
      - coze-network
    healthcheck:
      test: ['CMD-SHELL', 'nsqlookupd --version']
      interval: 5s
      timeout: 10s
      retries: 10
      start_period: 10s

  nsqd:
    image: nsqio/nsq:v1.2.1
    container_name: coze-nsqd
    command: /nsqd --lookupd-tcp-address=nsqlookupd:4160 --broadcast-address=nsqd
    restart: always
    ports:
      - '4150:4150'  # 暴露 NSQD 端口（可选）
      - '4151:4151'  # 暴露 NSQD HTTP 端口（可选）
    depends_on:
      nsqlookupd:
        condition: service_healthy
    networks:
      - coze-network
    healthcheck:
      test: ['CMD-SHELL', '/nsqd --version']
      interval: 5s
      timeout: 10s
      retries: 10
      start_period: 10s

  nsqadmin:
    image: nsqio/nsq:v1.2.1
    container_name: coze-nsqadmin
    command: /nsqadmin --lookupd-http-address=nsqlookupd:4161
    restart: always
    ports:
      - '4171:4171'  # 暴露 NSQ Admin 端口（可选）
    depends_on:
      nsqlookupd:
        condition: service_healthy
    networks:
      - coze-network

  coze-server:
    image: cozedev/coze-studio-server:latest
    restart: always
    container_name: coze-server
    env_file: *env_file
    networks:
      - coze-network
    ports:
      - '8888:8888'  # 暴露后端 API 端口
      - '8889:8889'  # 暴露后端管理端口（可选）
    volumes:
      - .env:/app/.env
      - ../backend/conf:/app/resources/conf
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
      elasticsearch:
        condition: service_healthy
      minio:
        condition: service_healthy
      milvus:
        condition: service_healthy
    command: ['/app/opencoze']

networks:
  coze-network:
    driver: bridge
EOF
```

#### 步骤 3：启动后端服务

```bash
# 进入 docker 目录
cd coze-studio/docker

# 使用自定义配置启动
docker compose -f docker-compose-backend-only.yml up -d

# 查看服务状态
docker compose -f docker-compose-backend-only.yml ps

# 查看日志
docker compose -f docker-compose-backend-only.yml logs -f
```

---

### 方法二：修改原始 docker-compose.yml

如果你不想创建新文件，也可以直接修改原始的 `docker/docker-compose.yml`：

#### 步骤 1：注释掉 coze-web 服务

编辑 `docker/docker-compose.yml`，注释掉或删除 `coze-web` 服务部分：

```yaml
# coze-web:
#   image: cozedev/coze-studio-web:latest
#   container_name: coze-web
#   restart: always
#   ports:
#     - "${WEB_LISTEN_ADDR:-8888}:80"
#   volumes:
#     - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
#     - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf:ro
#   depends_on:
#     - coze-server
#   networks:
#     - coze-network
```

#### 步骤 2：暴露 coze-server 端口

取消 `coze-server` 的端口注释：

```yaml
coze-server:
  image: cozedev/coze-studio-server:latest
  restart: always
  container_name: coze-server
  env_file: *env_file
  networks:
    - coze-network
  ports:
    - '8888:8888'  # 取消注释
    # - '8889:8889'
  volumes:
    - .env:/app/.env
    - ../backend/conf:/app/resources/conf
  depends_on:
    mysql:
      condition: service_healthy
    redis:
      condition: service_healthy
    elasticsearch:
      condition: service_healthy
    minio:
      condition: service_healthy
    milvus:
      condition: service_healthy
  command: ['/app/opencoze']
```

#### 步骤 3：（可选）暴露其他中间件端口

如果需要从外部访问中间件，可以取消对应的端口注释：

```yaml
mysql:
  # ...
  ports:
    - '3306:3306'  # 取消注释

redis:
  # ...
  ports:
    - '6379:6379'  # 取消注释

# 其他服务类似...
```

#### 步骤 4：启动服务

```bash
cd coze-studio
make web
# 或
docker compose -f docker/docker-compose.yml up -d
```

---

### 方法三：使用 Makefile（开发环境）

如果你是在开发环境中，可以使用 Makefile 只启动中间件和后端：

```bash
cd coze-studio

# 1. 启动中间件
make middleware

# 2. 构建前端（虽然不用，但 server 可能依赖静态资源）
make fe

# 3. 构建并启动后端
make server
```

---

## 验证部署

### 1. 检查容器状态

```bash
# 查看所有容器状态
docker ps -a | grep coze

# 预期输出：
# coze-server
# coze-mysql
# coze-redis
# coze-elasticsearch
# coze-minio
# coze-milvus
# coze-etcd
# coze-nsqlookupd
# coze-nsqd
# coze-nsqadmin
```

### 2. 检查后端健康状态

```bash
# 健康检查端点
curl http://localhost:8888/health

# 预期返回：{"status":"ok"} 或类似响应
```

### 3. 查看后端日志

```bash
# 查看 coze-server 日志
docker logs -f coze-server

# 确认没有 ERROR 级别日志
```

### 4. 测试 API 端点

参考官方 API 文档：https://github.com/coze-dev/coze-studio/wiki/6.-API-%E5%8F%82%E8%80%83

---

## 端口说明

| 服务 | 端口 | 说明 | 是否必须暴露 |
|------|------|------|-------------|
| coze-server | 8888 | 后端 API | **是** |
| coze-server | 8889 | 后端管理 | 否 |
| MySQL | 3306 | 数据库 | 否（仅内部访问） |
| Redis | 6379 | 缓存 | 否（仅内部访问） |
| Elasticsearch | 9200 | 搜索引擎 | 否（仅内部访问） |
| MinIO | 9000 | 对象存储 API | 否（仅内部访问） |
| MinIO | 9001 | 对象存储 Console | 否 |
| Milvus | 19530 | 向量数据库 | 否（仅内部访问） |
| etcd | 2379 | 协调服务 | 否（仅内部访问） |
| NSQ | 4150/4151/4160/4161/4171 | 消息队列 | 否（仅内部访问） |

---

## 常用运维命令

### 启动/停止/重启

```bash
# 使用自定义 compose 文件
cd coze-studio/docker

# 启动
docker compose -f docker-compose-backend-only.yml up -d

# 停止
docker compose -f docker-compose-backend-only.yml down

# 重启
docker compose -f docker-compose-backend-only.yml restart

# 查看状态
docker compose -f docker-compose-backend-only.yml ps
```

### 查看日志

```bash
# 查看所有服务日志
docker compose -f docker-compose-backend-only.yml logs -f

# 只看后端日志
docker logs -f coze-server

# 查看最近 100 行日志
docker logs --tail=100 coze-server
```

### 数据管理

```bash
# 备份数据（停止服务后执行）
cd coze-studio/docker
tar -czf coze-backup-$(date +%Y%m%d).tar.gz ./data

# 清理数据（谨慎！）
docker compose -f docker-compose-backend-only.yml down -v
rm -rf ./data
```

---

## 初始化配置

### 1. 注册用户（首次部署）

由于没有前端界面，需要通过 API 或直接操作数据库来注册用户：

**方式一：直接操作数据库**
```bash
# 进入 MySQL 容器
docker exec -it coze-mysql mysql -ucoze -pcoze123 opencoze

# 插入用户（示例）
INSERT INTO users (username, email, password_hash, created_at, updated_at) 
VALUES ('admin', 'admin@example.com', 'hashed_password_here', NOW(), NOW());
```

**方式二：临时启动前端注册**
如果不想直接操作数据库，可以临时启动前端注册用户后再停止：
```bash
# 启动完整服务（包括前端）
cd coze-studio
make web

# 浏览器访问 http://localhost:8888/sign 注册用户

# 注册完成后，停止并改用仅后端模式
make down_web
# 然后使用仅后端配置启动
```

### 2. 配置模型

通过 API 或管理员界面配置模型：

**API 方式**：参考官方 API 文档

**临时启动前端方式**：同用户注册，临时启动前端配置模型后再停止。

---

## 注意事项

1. **端口暴露安全**
   - 生产环境中，不要暴露中间件端口（MySQL、Redis 等）到公网
   - 后端 API 端口建议通过反向代理（Nginx）访问，并配置 HTTPS

2. **数据持久化**
   - 确保 `docker/data` 目录已挂载到持久化存储
   - 定期备份数据

3. **配置文件**
   - 生产环境务必修改所有默认密码
   - 妥善保管 `.env` 文件，不要提交到版本控制

4. **API 文档**
   - 详细的 API 使用方法参考官方 Wiki：https://github.com/coze-dev/coze-studio/wiki

---

## 参考资源

- 官方文档：https://github.com/coze-dev/coze-studio
- API 参考：https://github.com/coze-dev/coze-studio/wiki/6.-API-%E5%8F%82%E8%80%83
- 完整部署指南：./coze-studio发版部署指南.md

---

**文档版本**: 1.0  
**最后更新**: 2026-03-27  
**维护者**: 教授 🧑‍💻
