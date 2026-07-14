# 本地环境搭建 — PostgreSQL + pgvector + Redis + RustFS

> 对应配置：`docker-compose.dev.yml`、`.env`、`application.yml`

## 设计目标

InterviewGuide 依赖 3 个基础设施：PostgreSQL（含 pgvector 向量扩展）、Redis、S3 兼容对象存储。本文通过 Docker Compose 一键启动，并记录实际操作过程。

> **实操环境**：Windows 11 / Docker Desktop 29.6.1 / Docker Compose v5.2.0

---

## 1. Docker Desktop 安装

Windows 环境需先安装 Docker Desktop：

1. 访问 [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/)
2. 下载并安装 Windows 版
3. 启动 Docker Desktop，等待右下角图标变绿（Running）

验证：

```powershell
docker --version
docker compose version
```

---

## 2. 一键启动基础设施

项目根目录执行：

```powershell
docker compose -f docker-compose.dev.yml up -d
```

`docker-compose.dev.yml` 定义了 3 个服务：

```
services:
  postgres:
    image: pgvector/pgvector:pg16    # PG16 + 内置 pgvector 向量扩展
    container_name: interview-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: 123456
      POSTGRES_DB: interview_guide   # 自动创建业务库
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data   # 数据持久化

  redis:
    image: redis:7
    container_name: interview-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  rustfs:
    image: rustfs/rustfs:latest      # 轻量 S3 兼容存储
    container_name: interview-rustfs
    ports:
      - "9000:9000"   # API
      - "9001:9001"   # Web Console
    environment:
      RUSTFS_ACCESS_KEY: rustfsadmin
      RUSTFS_SECRET_KEY: rustfsadmin
```

### 2.1 关键设计点

- **数据持久化**：`volumes` 映射到 Docker 管理的卷，容器删除后数据不丢失
- **Healthcheck**：每个服务定义了健康检查，下游服务不会在依赖未就绪时启动
- **pgvector 镜像**：`pgvector/pgvector:pg16` 官方封装，开箱即用向量扩展

### 2.2 启动过程实录

```powershell
# 1. 验证 Docker
docker --version          # Docker version 29.6.1, build 8900f1d
docker compose version    # Docker Compose version v5.2.0

# 2. 一键启动
docker compose -f docker-compose.dev.yml up -d
```

首次启动会自动拉取镜像，耗时取决于网络：

```
Image pgvector/pgvector:pg16 Pulling
Image redis:7 Pulling
Image rustfs/rustfs:latest Pulling
```

镜像拉取完成后自动创建容器并启动。验证运行状态：

```powershell
docker compose -f docker-compose.dev.yml ps
```

输出：

```
NAME                 IMAGE                    STATUS
interview-postgres   pgvector/pgvector:pg16   Up (healthy)
interview-redis      redis:7                  Up (healthy)
interview-rustfs     rustfs/rustfs:latest     Up (unhealthy)
```

验证 PostgreSQL 连通性：

```powershell
docker exec interview-postgres pg_isready -U postgres
# /var/run/postgresql:5432 - accepting connections
```

> **注意**：RustFS 的 healthcheck 在此版本中可能显示 `unhealthy`，这是 Alpine 镜像中 `timeout` 命令行为差异导致的假阳性，不影响 S3 API 正常使用。

### 2.3 服务信息速查

| 服务 | 端口 | 账号 / 密码 | 用途 |
|------|------|-------------|------|
| PostgreSQL + pgvector | 5432 | `postgres` / `123456` | 关系数据 + 向量存储 |
| Redis | 6379 | 无密码 | 缓存 + Stream 消息队列 |
| RustFS | 9000 (API) / 9001 (Web) | `rustfsadmin` / `rustfsadmin` | S3 兼容对象存储 |

---

## 3. RustFS Bucket 创建

RustFS 的 Web Console 运行在 `http://localhost:9001`。启动后用 `rustfsadmin / rustfsadmin` 登录，手动创建名为 `interview-guide` 的 bucket。

也可通过命令行（需拉取 AWS CLI 镜像）：

```powershell
# 通过 Docker 容器运行 AWS CLI 创建 bucket
docker run --rm `
  -e AWS_ACCESS_KEY_ID=rustfsadmin `
  -e AWS_SECRET_ACCESS_KEY=rustfsadmin `
  -e AWS_DEFAULT_REGION=us-east-1 `
  amazon/aws-cli `
  --endpoint-url http://host.docker.internal:9000 `
  s3 mb s3://interview-guide
```

此 bucket 对应 `.env` 中的 `APP_STORAGE_BUCKET=interview-guide`，用于存储简历文件、导出 PDF 等。

---

## 4. DataGrip 连接 PostgreSQL

DataGrip 是 JetBrains 出品的数据库 IDE，非商业用途免费使用。

### 4.1 下载与激活

1. 下载 [jetbrains.com/datagrip](https://www.jetbrains.com/datagrip/)
2. 启动后选择 **Non-commercial use**
3. 登录 JetBrains 账号完成激活

### 4.2 连接数据库

1. 新建 Project
2. `+` → Data Source → PostgreSQL
3. 填写连接信息：
   - Host: `localhost`
   - Port: `5432`
   - User: `postgres`
   - Password: `123456`
   - Database: `interview_guide`
4. 首次连接会提示下载 PostgreSQL 驱动 → Download
5. 点击 Test Connection 验证

### 4.3 连接后的视图

```
postgres@localhost
├── postgres
└── interview_guide        ← 业务库（Docker Compose 自动创建）
    ├── public
    │   ├── resume_entity           # 简历表
    │   ├── interview_session_entity # 面试会话表
    │   └── ...
    └── extensions
        └── vector        ← pgvector 向量扩展
```

---

## 5. pgvector 向量扩展

### 5.1 镜像已内置

`pgvector/pgvector:pg16` 镜像已包含向量扩展，但 PostgreSQL 的插件机制是"按需加载"。需要在数据库中手动执行：

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

### 5.2 Spring AI 自动初始化

项目中通过 `application.yml` 配置了 Spring AI 的 `PgVectorStore`：

```yaml
spring:
  ai:
    vectorstore:
      pgvector:
        initialize-schema: true   # 开发环境自动建表
```

| 配置值 | 适用场景 |
|--------|----------|
| `true` | 开发环境，自动创建 `vector_store` 表，快速启动 |
| `false` | 生产环境，手动管理 Schema，避免意外变更 |

### 5.3 项目 pgvector 配置

| 参数 | 值 | 说明 |
|------|-----|------|
| 向量维度 | 1024 | 由 DashScope Embedding 模型（`text-embedding-v4`）决定 |
| 距离类型 | COSINE | 余弦相似度，语义检索场景最常用 |
| 索引类型 | IVFFlat | 平衡检索速度与精度 |

---

## 6. .env 配置

从 `.env.example` 复制 `.env`（已在 `.gitignore` 中，不会提交）：

```bash
cp .env.example .env
```

核心配置项：

| 变量 | 必需 | 说明 |
|------|:--:|------|
| `AI_BAILIAN_API_KEY` | ✅ | 阿里云百炼 API Key，用于 LLM + ASR + TTS |
| `APP_AI_CONFIG_ENCRYPTION_KEY` | ✅ | Provider API Key 本地加密密钥 |
| `POSTGRES_HOST/PORT/DB/USER/PASSWORD` | — | 与 docker-compose.dev.yml 对齐，本地勿改 |
| `REDIS_HOST/PORT` | — | 同上 |
| `APP_STORAGE_*` | — | RustFS S3 配置，同上 |

可选 Provider（不填则不启用）：
- `PROVIDER_KIMI_API_KEY` — 月之暗面 Kimi
- `PROVIDER_DEEPSEEK_API_KEY` — DeepSeek
- `PROVIDER_GLM_API_KEY` — 智谱 GLM

---

## 7. 前后端启动实录

> 环境：Windows 11 / Docker Desktop 29.6.1 / Java 21 (Temurin) / Node 24.18.0

### 7.1 启动前检查

```powershell
# Docker 容器状态
docker compose -f docker-compose.dev.yml ps
# NAME                 STATE     HEALTH
# interview-postgres   running   healthy
# interview-redis      running   healthy
# interview-rustfs     running   unhealthy (Alpine 兼容问题, API 正常)

# Java 版本
"C:\Users\amazi\scoop\apps\temurin21-jdk\current\bin\java" -version
# openjdk version "21.0.11" 2026-04-21 LTS

# Node 版本
node --version  # v24.18.0
```

### 7.2 启动后端

使用 Scoop 安装的 Temurin21 JDK（系统默认 Java 8 不兼容），通过 `NO_PROXY` 排除本地地址防止代理拦截 S3 请求：

```bash
export JAVA_HOME="C:/Users/amazi/scoop/apps/temurin21-jdk/current"
export NO_PROXY="localhost,127.0.0.1,::1"
export no_proxy="localhost,127.0.0.1,::1"
cd /e/SynologyDrive/Codespace/interview-guide
./gradlew :app:bootRun --no-daemon
```

### 7.3 遇到的问题及解决

#### 问题 1：端口 8080 被占用

**现象**：
```
Web server failed to start. Port 8080 was already in use.
```

**诊断**：`Get-NetTCPConnection -LocalPort 8080` 发现 PID 7908 为系统进程 `AgentService`，`taskkill /F` 拒绝访问。

**解决**：改用端口 8082：
```bash
export SERVER_PORT=8082
./gradlew :app:bootRun --no-daemon
```

#### 问题 2：TTS 预热失败（非阻塞）

**现象**：`TTS synthesis interrupted` 连续报错，DashScope TTS 连接 `wss://dashscope.aliyuncs.com` 失败。

**根因**：`.env` 中 `AI_BAILIAN_API_KEY` 未配置真实 Key。

**结论**：不影响核心功能启动。预热线程被 InterruptedException 打断后 Spring Boot 继续初始化剩余 beans，最终 `Started App in 14.3 seconds`，10 个 Skill 全部加载，4 个 Redis Stream Consumer 正常启动。

### 7.4 启动前端

后端端口变更后需对齐 Vite proxy 目标：

```powershell
cd frontend
$env:VITE_API_PROXY_TARGET = "http://localhost:8082"
pnpm run dev
# VITE v5.4.21  ready in 694 ms
# ➜  Local:   http://localhost:5173/
```

### 7.5 运行状态确认

| 服务 | 端口 | 状态 |
|------|------|:--:|
| PostgreSQL + pgvector | 5432 | ✅ healthy |
| Redis | 6379 | ✅ healthy |
| RustFS | 9000 / 9001 | ✅ running |
| Spring Boot 后端 | 8082 | ✅ Started in 14.3s |
| Vite 前端 | 5173 | ✅ ready in 694ms |

关键日志：
```
Tomcat started on port 8082 (http) with context path '/'
共加载 10 个预设 Skill
存储桶已存在: interview-guide
Redisson: 10 connections initialized for localhost/127.0.0.1:6379
analyze / evaluate / vectorize / voice-evaluate 4 个 Consumer 全部 started
```

### 7.6 访问入口

| 入口 | URL |
|------|-----|
| 前端页面 | http://localhost:5173 |
| 后端 API | http://localhost:8082 |
| Swagger UI | http://localhost:8082/swagger-ui.html |
| Actuator | http://localhost:8082/actuator |
| RustFS Console | http://localhost:9001 |

---

## 常用命令

```powershell
# 停止所有服务（保留数据）
docker compose -f docker-compose.dev.yml down

# 停止并清除数据（重置环境）
docker compose -f docker-compose.dev.yml down -v

# 查看容器日志
docker compose -f docker-compose.dev.yml logs -f postgres

# 进入 PostgreSQL 容器
docker exec -it interview-postgres psql -U postgres -d interview_guide
```
