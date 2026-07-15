# 面试会话列表投影优化

> 对应接口：`GET /api/interview/sessions`（前端经 Vite 代理为 `http://localhost:5173/api/interview/sessions`）  
> 对应源码：`InterviewSessionRepository`、`InterviewPersistenceService`、`InterviewController`、`SessionListItemDTO`

## 现象

面试记录页加载慢：列表 API 响应时间偏长，数据量不大时也不理想。

## 根因

列表页 DTO（`SessionListItemDTO`）只用轻量字段：`sessionId`、`skillId`、`status`、`overallScore`、`createdAt` 等。

旧路径却是：

```text
Controller → persistenceService.findAll()
          → sessionRepository.findAllByOrderByCreatedAtDesc()  // 整行 Entity
          → SessionListItemDTO.from(entity)
```

`InterviewSessionEntity` 含多个 **TEXT** 大字段（`questionsJson`、`overallFeedback`、`strengthsJson`、`improvementsJson`、`referenceAnswersJson` 等）。`findAll()` 会把这些列全部读入内存，再丢弃不用——I/O 与反序列化开销浪费在列表场景上。

## 优化做法

### 1. JPQL 构造器投影

在 `InterviewSessionRepository` 中只 SELECT 列表需要的列，直接构造 DTO：

```java
@Query("""
    SELECT new interview.guide.modules.interview.model.SessionListItemDTO(
        s.sessionId, s.skillId, s.difficulty, s.resumeId,
        COALESCE(s.totalQuestions, 0), s.status, s.evaluateStatus,
        s.evaluateError, s.overallScore, s.createdAt, s.completedAt
    )
    FROM InterviewSessionEntity s
    ORDER BY s.createdAt DESC
    """)
List<SessionListItemDTO> findAllListItems();
```

### 2. Service / Controller 接线

- `InterviewPersistenceService.findAllListItems()`：`@Transactional(readOnly = true)`，委托投影查询。
- `InterviewController.listSessions()`：改为 `return Result.success(persistenceService.findAllListItems())`，不再 `findAll().stream().map(...)`。

### 3. 排序索引

实体表索引包含 `created_at`（及 resume / skill 组合索引），支撑 `ORDER BY created_at DESC`，避免大表全表排序拖累。

## 验证

重启后端后打开面试记录页，或：

```powershell
Measure-Command { Invoke-RestMethod http://localhost:8082/api/interview/sessions }
```

对比优化前后耗时；必要时在 PostgreSQL 用 `EXPLAIN ANALYZE` 确认未扫描 TEXT 大列。

## 核心要点

- **列表 API 的 SELECT 应由 DTO 字段驱动**，不要先加载完整 Entity 再裁剪。
- JPQL `SELECT new ...DTO(...)` 是 Spring Data JPA 常用的轻量投影方式。
- 大字段（TEXT/JSON）适合详情接口按需加载；列表与详情查询路径应分开。

→ 本会话改动清单见 [session-2026-07-15.md](../code-changes/00-env-setup/session-2026-07-15.md)
