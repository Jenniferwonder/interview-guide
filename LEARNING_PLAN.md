# InterviewGuide 学习行动计划 — 业务链路驱动 + 代码交付

> **学习方向**：从前端扩展到 AI 全栈
> **组织方式**：按项目的**三大核心功能 + 两大技术亮点**，每个交付物追踪一条完整业务链路
> **预计总耗时**：3-5 天

---

## 六个交付物总览

```
交付物                   核心问题                             链路长度
─────────────────────────────────────────────────────────────────────
0  环境搭建              "项目怎么跑起来？"                   基础设施层
1  LLM 集成层            "大模型怎么接？输出怎么保证可靠？"   配置 → 调用 → 解析
2  智能简历分析          "AI 怎么异步分析文件？"              上传 → 存储 → 分析 → 报告
3  模拟面试系统          "AI 怎么出题、追问、评估？"          Skill → 出题 → 评估 → 导出
4  RAG 知识库问答        "怎么让 LLM 基于私有知识回答？"     文档 → 向量 → 检索 → 生成
5  异步引擎 + 工程化     "系统怎么扛住并发？怎么安全？"      横切面全部模块
```

---

## 交付物 0：本地环境一键启动

### 链路

```
Docker Compose → PG+pgvector + Redis + RustFS → .env 配置 → 前后端启动
```

### 必读源码

| 文件 | 关注点 |
|------|--------|
| `docker-compose.dev.yml` | 3 个服务的编排方式、健康检查、数据卷 |
| `.env.example` | 所有环境变量全景 |
| `application.yml` | Spring Boot 如何通过 `${VAR:default}` 读取配置 |
| `docker/postgres/init.sql` | 数据库初始化脚本 |

### 代码改动

**已完成**：创建 `.env`、配置 RustFS 凭证、修复代理拦截 S3 500 的问题。

→ 记录在 `my-learning/code-changes/00-env-setup/`

### 踩坑记录

| 问题 | 根因 | 解决 |
|------|------|------|
| RustFS S3 返回 500 | 系统 HTTP_PROXY 拦截了 localhost 请求 | `NO_PROXY=localhost,127.0.0.1` |
| Java 版本不匹配 | 系统默认 Java 8，项目需要 Java 21 | 切换到 Scoop 安装的 `temurin21-jdk` |
| Java 21 未生效 | `JAVA_HOME` 被系统变量覆盖 | 使用绝对路径 `$JAVA_HOME/bin/java` |

---

## 交付物 1：多 LLM Provider + 结构化输出

### 链路

```
LlmProviderRegistry.getChatClient(provider)
  → ChatClient.prompt().user().call()
  → StructuredOutputInvoker（JSON 修复 + 重试）
  → Prompt 模板 (StringTemplate) → LLM 返回 → Java 对象
```

### 必读源码

| 文件 | 关注点 |
|------|--------|
| `common/ai/LlmProviderRegistry.java` | Provider 注册与 ChatClient 缓存 |
| `common/ai/StructuredOutputInvoker.java` | 结构化输出的重试与 JSON 修复 |
| `common/ai/ApiPathResolver.java` | OpenAI 兼容 API 路径解析 |
| `modules/llmprovider/` | Provider 运行时管理与连通性测试 |
| `resources/prompts/` | 14 个 StringTemplate Prompt 模板 |

### 代码改动目标

**新增一个 LLM Provider**：在 `application.yml` / `.env` 中添加自定义 OpenAI 兼容 Provider 配置（如硅基流动 / Groq），验证运行时 `LlmProviderRegistry` 是否能自动发现并切换。

改动范围：
1. `.env` 中新增 `PROVIDER_MYPROVIDER_API_KEY` + `PROVIDER_MYPROVIDER_MODEL`
2. `application.yml` 中新增 Provider 配置块（base-url、api-key）
3. 通过 Admin UI 验证连通性

---

## 交付物 2：智能简历分析（上传 → 解析 → AI 评分 → 报告导出）

### 链路

```
前端 UploadPage.tsx
  → POST /api/resume/upload
  → FileStorageService.uploadResume() → S3 存储
  → AnalyzeStreamProducer.send() → XADD Redis Stream
  → AnalyzeStreamConsumer.processMessage()
    → Tika 文本提取 → 文本清洗
    → LlmProviderRegistry.getChatClient() → resume-analysis-*.st Prompt
    → StructuredOutputInvoker → ResumeAnalysisDTO
  → ResumePersistenceService 落库
  → 前端轮询 /api/resume/{id}/status → AnalysisPanel 展示
  → iText 8 PDF 报告导出
```

### 必读源码

| 文件 | 关注点 |
|------|--------|
| `modules/resume/ResumeController.java` | REST API 设计 |
| `modules/resume/ResumeUploadService.java` | 业务编排（验证 → 存储 → 分析） |
| `modules/resume/AnalyzeStreamProducer.java` | 生产者实现：如何发送异步任务 |
| `modules/resume/AnalyzeStreamConsumer.java` | 消费者实现：如何消费并执行业务 |
| `infrastructure/file/FileStorageService.java` | S3 上传/下载/删除封装 |
| `infrastructure/file/DocumentParseService.java` | Tika 多格式解析 |
| `infrastructure/file/FileHashService.java` | SHA-256 文件去重 |
| `resources/prompts/resume-analysis-system.st` | System Prompt 设计 |
| `resources/prompts/resume-analysis-user.st` | User Prompt 设计 |

### 代码改动目标

**追踪并记录一次完整的简历分析请求**——不是改代码，而是画一张带类名和方法名的完整时序图，标注每个环节的耗时特征（同步 vs 异步）。

---

## 交付物 3：模拟面试系统（Skill 出题 → 多轮对话 → 评估 → 导出）

### 链路

```
前端 InterviewPage.tsx
  → GET /api/interview/skills → 10 个方向的 SKILL.md 列表
  → POST /api/interview/session → 创建会话 + 选择方向
  → POST /api/interview/question → InterviewQuestionService
    → 加载 SKILL.md → LLM 出题（历史去重 + JD 适配）
  → 用户回答 → POST /api/interview/answer
  → Redis Stream → AnswerEvaluationService
    → interview-evaluation-*.st Prompt → 结构化输出
    → 分批评估 → 二次汇总 → 降级兜底
  → iText 8 PDF 导出
```

### 必读源码

| 文件 | 关注点 |
|------|--------|
| `modules/interview/InterviewController.java` | 面试会话、出题、回答 API |
| `modules/interview/InterviewQuestionService.java` | Skill 加载 + 出题逻辑 |
| `resources/skills/*/SKILL.md` | 10 个面试方向的考察范围定义 |
| `modules/interview/AnswerEvaluationService.java` | 评估编排 |
| `common/evaluation/UnifiedEvaluationService.java` | 统一评估引擎（文字+语音共用） |
| `resources/prompts/interview-question-skill-*.st` | 出题 Prompt |
| `resources/prompts/interview-evaluation-*.st` | 评估 + 汇总 Prompt |

### 代码改动目标

**新增一个面试 Skill**：在 `resources/skills/` 下创建新方向的 `SKILL.md`（如 Go 后端 / C++ 后端 / DevOps），验证出题内容是否与新定义的方向一致。

改动范围：
1. `resources/skills/go-backend/SKILL.md`：定义考察范围、难度分布、参考知识库
2. 必要时在 `resources/skills/_shared/references/` 补充参考知识
3. 启动后端 → 在前端选择新方向 → 验证出题质量

---

## 交付物 4：RAG 知识库问答（上传 → 向量化 → 检索 → 流式生成）

### 链路

```
前端 KnowledgeBaseUploadPage.tsx
  → POST /api/knowledge-base/upload → S3 存储文档
  → VectorizeStreamProducer → XADD Redis Stream
  → VectorizeStreamConsumer.processMessage()
    → Tika 文本提取 → 分块 (Chunking)
    → Embedding (DashScope text-embedding-v4, 维度 1024)
    → pgvector INSERT
  ↓
前端 KnowledgeBaseQueryPage.tsx
  → POST /api/rag-chat/query (SSE)
  → Query Rewrite (knowledgebase-query-rewrite.st)
  → Embedding → pgvector 向量检索 (COSINE 距离)
  → TopK 过滤 + 相似度阈值
  → Context 拼接 → LLM 流式生成
  → SSE (SseEmitter) → React Virtuoso 虚拟列表
```

### 必读源码

| 文件 | 关注点 |
|------|--------|
| `modules/knowledgebase/KnowledgeBaseController.java` | 知识库管理 API |
| `modules/knowledgebase/RagChatController.java` | RAG 问答 SSE 端点 |
| `modules/knowledgebase/KnowledgeBaseVectorService.java` | 向量化逻辑（分块 + Embedding + 存储） |
| `modules/knowledgebase/KnowledgeBaseQueryService.java` | RAG 检索 + 生成 |
| `modules/knowledgebase/RagChatSessionService.java` | 多轮对话会话管理 |
| `resources/prompts/knowledgebase-query-rewrite.st` | Query Rewrite Prompt |
| `resources/prompts/knowledgebase-query-*.st` | RAG 问答 System/User Prompt |
| pgvector 配置 (`application.yml`) | 向量维度 1024、COSINE 距离 |

### 代码改动目标

**修改 Query Rewrite Prompt 并观察检索效果变化**：改 `knowledgebase-query-rewrite.st`，上传一份技术文档作为知识库，对比修改前后对同一问题的检索结果差异。

改动范围：
1. 修改 `knowledgebase-query-rewrite.st`
2. 上传一份测试文档 → 提问 → 记录检索到的文档片段
3. 对比对照组 → 评估 Prompt 改动对检索命中率的影响

---

## 交付物 5：异步任务引擎 + 工程化基础设施

### 链路

**异步任务引擎**：
```
业务触发 → Producer.send() → XADD Stream
  → Consumer 轮询 (XREADGROUP) → processMessage()
  → 成功 → XACK
  → 失败 → retry < 3 → 重新投递   /   retry >= 3 → FAILED
  → 消费前校验：实体已删除 → 直接 ACK 丢弃
```

**工程化基础设施**：
```
@RateLimit → RateLimitAspect → Redis Lua 滑动窗口
PromptSanitizer → 输入校验 → 防止注入
配置分层：.env → application.yml → @ConfigurationProperties
全局限流 / API Key 加密 / CORS / 内容安全
```

### 必读源码

| 文件 | 关注点 |
|------|--------|
| `common/async/AbstractStreamProducer.java` | 生产者模板 |
| `common/async/AbstractStreamConsumer.java` | 消费者模板（确认/重试/死信） |
| `common/annotation/RateLimit.java` | 可重复限流注解 |
| `common/aspect/RateLimitAspect.java` | Redis Lua 滑动窗口切面 |
| `common/ai/PromptSanitizer.java` | Prompt 注入防护 |
| `common/config/` 下的配置类 | `@ConfigurationProperties` 实践 |
| `common/exception/` | `BusinessException` + `GlobalExceptionHandler` |

### 代码改动目标

**为 `StructuredOutputInvoker` 编写单元测试**：覆盖正常解析、JSON 格式异常强制修复、超过最大重试次数后降级返回三种场景。

改动范围：
1. `app/src/test/` 下新增测试类
2. Mock `ChatClient` 返回不同质量的 JSON
3. 验证重试计数器和降级逻辑

---

## 笔记索引

```
交付物 0 → my-learning/code-changes/00-env-setup/ + my-learning/notes/01-env-setup.md
交付物 1 → my-learning/notes/02-spring-ai-provider.md + my-learning/notes/03-unified-evaluation.md
交付物 2 → my-learning/notes/05-redis-stream-async.md
交付物 3 → my-learning/notes/03-unified-evaluation.md
交付物 4 → my-learning/notes/04-rag-pipeline.md
交付物 5 → my-learning/notes/05-redis-stream-async.md
```
