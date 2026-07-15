# JPA ddl-auto 与数据丢失

> 对应配置：`app/src/main/resources/application.yml` → `spring.jpa.hibernate.ddl-auto`  
> 对应现象：重启后端后，之前的面试记录全部消失

## 现象

本地 `bootRun` 重启后，前端面试记录页为空，`GET /api/interview/sessions` 返回空列表。PostgreSQL 容器与数据卷仍在，并非 Docker `down -v` 清库。

## 根因

`spring.jpa.hibernate.ddl-auto` 曾设为 **`create`**：

- 每次应用启动时，Hibernate **先删表再建表**；
- `interview_sessions` 及关联答案等业务数据随之清空；
- Docker 卷还在，但表内容已被 schema 重建擦掉。

配置文件中的注释也写明了意图：

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update #首次启动用 create，表创建成功后，改回 update
```

首次建表可用 `create`，建好后必须改回 `update`（或更严格的模式），否则每次重启都会丢数据。

## ddl-auto 对照

| 值 | 行为 | 适用场景 |
|----|------|----------|
| `create` | 启动时删表再建 | 仅一次性初始化 / 可丢数据的实验 |
| `create-drop` | 启动建表，**关闭应用时再删表** | 集成测试 |
| `update` | 按实体增量改表结构，**尽量保留数据** | 本地开发（当前推荐） |
| `validate` | 只校验实体与库表是否一致，不改库 | 接近生产的环境 |
| `none` | 完全不碰 schema | 生产（配合 Flyway/Liquibase） |

注意：`update` 不会做复杂迁移（如删列、改类型），大改表结构仍需手工 SQL 或迁移工具。

## 修复与现状

1. 将 `ddl-auto` 设为 `update`（本仓库当前已是 `update`）。
2. 生产环境：使用 Flyway / Liquibase 管理 schema，并将 `ddl-auto` 设为 `validate` 或 `none`（见 `AGENTS.md`：生产不能依赖自动建表）。

## 验证

```powershell
# 1. 确认配置
Select-String -Path app\src\main\resources\application.yml -Pattern 'ddl-auto'

# 2. 新建一条面试会话后重启后端
# 3. 再请求列表，记录应仍在
curl http://localhost:8082/api/interview/sessions
```

在 PostgreSQL 中也可核对：

```powershell
docker exec -it interview-postgres psql -U postgres -d interview_guide -c "SELECT session_id, created_at FROM interview_sessions ORDER BY created_at DESC LIMIT 5;"
```

## 核心要点

- **重启丢业务数据，优先查 `ddl-auto`，不要先怪 Docker 卷。**
- `create` / `create-drop` 适合“可丢库”场景；日常开发用 `update`。
- 生产应用迁移工具 + `validate`/`none`，避免 Hibernate 改生产表结构。

→ 相关踩坑摘要见 [code-changes/00-env-setup](../code-changes/00-env-setup/README.md)
