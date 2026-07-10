# InterviewGuide 项目学习行动计划 — AI 应用开发工程师面试准备

> **目标岗位**：AI 应用开发工程师
> **项目定位**：基于 Spring AI + RAG + 多模态的全栈智能面试平台
> **预计总耗时**：3-5 天（约 25-35 小时）

---

## 学习路线总览

```
Day 1: 跑起来 + 摸清架构
Day 2: AI 核心链路深挖（LLM 调用 / 结构化输出 / Prompt 工程）
Day 3: RAG + 异步 + 语音（知识库检索增强 / Redis Stream / WebSocket 语音）
Day 4: 工程化能力 + 面试表达（限流 / 安全 / 评估架构 / 项目亮点整理）
Day 5: 模拟面试 + 查缺补漏
```

---

## Day 1：环境搭建 + 架构概览（~6h）

### 1.1 把项目跑起来（2h）

```bash
# 1. 启动基础设施
docker compose -f docker-compose.dev.yml up -d

# 2. 配置 .env（从 .env.example 复制，填入百炼 API Key）
cp .env.example .env

# 3. 启动后端
./gradlew :app:bootRun

# 4. 启动前端
cd frontend && pnpm install && pnpm run dev
```

**验证点**：浏览器打开 `http://localhost:5173`，能看到面试中心页面。

### 1.2 理解项目骨架（1.5h）

按顺序读这些文件，边读边在 IDE 中打开对应目录：

| 顺序 | 文件 / 目录 | 关注点 |
|------|------------|--------|
| 1 | `AGENTS.md` | 技术栈、架构约定、禁令清单 |
| 2 | `app/src/main/java/interview/guide/` | 顶层包结构：`common/` `infrastructure/` `modules/` |
| 3 | `app/src/main/resources/application.yml` | 配置结构、Provider 定义、pgvector 配置 |
| 4 | `.env.example` | 环境变量全景 |
| 5 | `frontend/src/` | 前端路由 → 页面 → API 调用链 |

**核心认知**：
- `common/` = 横切能力（AI 调用、限流、异步、异常、统一响应）
- `infrastructure/` = 技术基础设施（文件、导出、Redis、MapStruct）
- `modules/` = 6 个业务模块，每个自包含 MVC 三层

### 1.3 走通一个完整业务流程（2.5h）

**推荐选择"简历上传 → 分析 → 查看报告"这条链**，因为它覆盖了：

```
前端 UploadPage.tsx
  → POST /api/resume/upload
  → Redis Stream 异步分析（AbstractStreamProducer → AbstractStreamConsumer）
  → LLM 调用（LlmProviderRegistry → StructuredOutputInvoker → Prompt 模板）
  → 结果落库 + 前端轮询进度
```

用 Debug 模式跟踪一遍，记录每个环节的类和方法。

---

## Day 2：AI 核心链路深挖（~7h）

> 这是面试的重点考察区域，需要能讲清楚"怎么做"和"为什么这么做"。

### 2.1 LLM Provider 管理与调用（2h）

**必读源码**：

| 类 | 为什么重要 |
|----|----------|
| `LlmProviderRegistry.java` | 多 Provider 注册中心，理解如何动态获取 ChatClient |
| `ApiPathResolver.java` | OpenAI 兼容 API 路径解析 |
| `StructuredOutputInvoker.java` | 结构化输出的核心封装（重试、JSON 修复） |
| `StructuredOutputProperties.java` | 结构化输出配置 |

**面试能讲清楚**：
1. 如何支持多个 LLM Provider（DashScope/Kimi/DeepSeek/GLM）？
2. Spring AI 的 ChatClient 是怎么集成的？
3. 结构化输出为什么要封装？JSON 解析失败怎么处理？
4. Provider 的 API Key 怎么安全存储？（加密落盘到 `~/.interview-guide/`）

### 2.2 Prompt 工程（1.5h）

**必读文件**：

```
app/src/main/resources/prompts/
├── resume-analysis-system.st      # 简历分析 Prompt
├── interview-question-skill-system.st  # 面试出题 Prompt
├── interview-evaluation-system.st      # 面试评估 Prompt（分批评估）
├── interview-evaluation-summary-system.st  # 二次汇总 Prompt
├── knowledgebase-query-system.st       # RAG 问答 Prompt
├── knowledgebase-query-rewrite.st      # 查询改写 Prompt
└── jd-parse-system.st                  # JD 解析 Prompt
```

**面试能讲清楚**：
1. 为什么用 StringTemplate 而不是字符串拼接？
2. 面试评估为什么要"分批评估 → 二次汇总"？
3. 查询改写（Query Rewrite）在 RAG 中起什么作用？
4. System Prompt 和 User Prompt 分别放什么内容？

### 2.3 Skill 驱动的出题系统（1.5h）

**必读文件**：

```
app/src/main/resources/skills/
├── java-backend/SKILL.md
├── ai-agent-dev/SKILL.md
├── frontend/SKILL.md
├── system-design/SKILL.md
└── ...（共 10 个方向）
```

以及 `InterviewQuestionService` 中加载 SKILL.md 和出题的逻辑。

**面试能讲清楚**：
1. Skill.md 的结构设计：考察范围、难度分布、参考知识库
2. 出题时如何做到"历史题目去重"？
3. 如何根据 JD 内容动态调整出题方向？

### 2.4 统一评估引擎（2h）

**必读源码**：

| 类 | 说明 |
|----|------|
| `UnifiedEvaluationService.java` | 统一评估服务入口 |
| `EvaluationReport.java` | 评估报告结构 |
| `QaRecord.java` | 问答记录模型 |

**面试能讲清楚**：
1. 文字面试和语音面试如何共用同一套评估引擎？
2. 大批量问答如何分批评估？Token 超限怎么处理？
3. 评估结果的结构化输出包含哪些维度？
4. LLM 评估失败时的降级兜底策略？

---

## Day 3：RAG + 异步 + 语音（~7h）

### 3.1 RAG 知识库系统（2.5h）

**必读源码**：

| 类 / 文件 | 关注点 |
|----------|--------|
| `knowledgebase/` 模块下的 Service | 文档上传、分块策略、向量化流程 |
| `knowledgebase-query-rewrite.st` | 查询改写 Prompt |
| `knowledgebase-query-system.st` | RAG 问答 Prompt |
| SSE 流式响应实现 | 如何做打字机效果 |
| pgvector 配置 | 向量维度 1024、COSINE 距离 |

**面试能讲清楚**：
1. 文档向量化的完整流程（上传 → 分块 → Embedding → 存 pgvector）
2. RAG 的检索链路（Query Rewrite → Embedding → 向量检索 → 拼接 Context → LLM 生成）
3. 为什么选 pgvector 而不是 Milvus/Weaviate/Pinecone？
4. 分块策略是什么？Chunk size 和 overlap 怎么定的？
5. SSE 流式响应的实现细节

### 3.2 Redis Stream 异步处理（2h）

**必读源码**：

| 类 | 关注点 |
|----|--------|
| `AsyncTaskStreamConstants.java` | Stream Key、Consumer Group、任务状态常量 |
| `AbstractStreamProducer.java` | 生产者模板 |
| `AbstractStreamConsumer.java` | 消费者模板（确认/重试/死信） |
| `resume/` 模块的 listener 包 | 简历分析消费者具体实现 |

**面试能讲清楚**：
1. 为什么用 Redis Stream 而不是 Kafka/RabbitMQ？
2. 生产者和消费者的模板设计解决了什么问题？
3. 消费失败后重试几次？超过重试次数怎么办？
4. 如何处理"实体已删除但消息还在"的情况？
5. 前端如何实时获取异步任务进度？

### 3.3 语音面试（2.5h）

**必读源码**：

| 类 / 文件 | 关注点 |
|----------|--------|
| `voiceinterview/` 模块 WebSocket Handler | 实时双向通信 |
| DashScope SDK 集成 | ASR 语音识别 / TTS 语音合成 |
| VAD 服务端断句逻辑 | 静音检测、自动断句 |
| 回声防护 + 手动提交 | 避免 AI 语音被误录 |

**面试能讲清楚**：
1. WebSocket 在语音面试中的作用？为什么不用 HTTP？
2. ASR 和 TTS 的数据流是怎样的？
3. "句子级并发 TTS，边生成边合成边播放"是怎么实现的？
4. 服务端 VAD 断句的原理？中间结果和最终结果的区别？
5. 已知的局限性（延迟、回声、音色）和改进方向？

---

## Day 4：工程化能力 + 面试表达（~7h）

### 4.1 限流系统（1.5h）

**必读源码**：

| 类 | 关注点 |
|----|--------|
| `@RateLimit` 注解 | 可重复注解，Global/IP/User 维度 |
| `RateLimitAspect.java` | AOP 切面实现 |
| Redis Lua 脚本 | 滑动窗口算法 |

**面试能讲清楚**：
1. 为什么要做可重复注解？一个接口配多个限流维度的场景？
2. 滑动窗口算法 vs 固定窗口 vs 令牌桶？
3. 为什么用 Redis Lua 脚本而不是 Java 代码？
4. 限流的 Redis Key 命名规范？

### 4.2 安全与配置（1h）

| 类 / 文件 | 关注点 |
|----------|--------|
| `PromptSanitizer.java` | 防止 Prompt 注入 |
| `PromptSecurityConstants.java` | 安全常量 |
| API Key 加密存储 | `~/.interview-guide/` 目录加密落盘 |
| CORS 配置 | 跨域安全 |

**面试能讲清楚**：
1. Prompt 注入攻击是什么？如何防护？
2. Provider API Key 怎么安全存储和传输？
3. 配置分层：`.env` → `application.yml` → `@ConfigurationProperties`

### 4.3 全局架构设计决策（1.5h）

整理项目中的关键架构决策，面试时可以主动展示技术判断力：

| 决策 | 选择 | 理由 |
|------|------|------|
| 向量数据库 | pgvector（非独立向量库） | 精简架构，PG 向量能力够用 |
| 消息队列 | Redis Stream（非 Kafka） | 精简架构，不想引入太多组件 |
| 构建工具 | Gradle（非 Maven） | 个人偏好，更灵活的 DSL |
| 对象存储 | MinIO / RustFS（S3 兼容） | 不绑定云厂商 |
| 语音模型 | 千问3（ASR/TTS/LLM 统一 Key） | 简化配置 |
| 事务隔离 | LLM/S3/HTTP 不在事务内 | 避免长事务和连接占用 |
| 响应格式 | HTTP 200 + Result.error(code, msg) | 统一异常处理，业务错误不抛 HTTP 异常 |

### 4.4 项目亮点总结（3h）

**面试中主动展示的 5 个亮点**（每个准备 3-5 分钟的口述）：

#### 亮点 1：多 Provider + 统一结构化输出
> 设计了 `LlmProviderRegistry` 管理多 LLM Provider，配合 `StructuredOutputInvoker` 实现带重试的结构化输出。新增 Provider 只需配置，不需要改业务代码。这体现了**开闭原则**和**策略模式**。

#### 亮点 2：RAG 全链路（文档 → 向量 → 检索 → 生成）
> 完整实现了 RAG 知识库系统：Tika 文档解析 → 分块策略 → pgvector 向量化 → Query Rewrite → 相似度检索 → SSE 流式生成。关键细节：分块大小、Overlap、TopK 策略、查询改写对检索质量的影响。

#### 亮点 3：Redis Stream 异步任务模板
> 设计了 `AbstractStreamProducer` / `AbstractStreamConsumer` 模板，把创建 Stream、发送消息、消费确认、重试、死信处理封装成可复用的抽象。新增异步任务只需继承模板实现业务逻辑。

#### 亮点 4：文字+语音统一评估引擎
> 文字面试和语音面试共用 `UnifiedEvaluationService`，采用"分批评估 + 结构化输出 + 二次汇总 + 降级兜底"策略，既解决了 Token 限制，又保证了评估质量。

#### 亮点 5：实时语音面试（WebSocket + 并发 TTS）
> WebSocket 全双工通信 + 服务端 VAD 断句 + 句子级并发 TTS"边生成边合成边播放"，首包延迟控制在 200ms。有意识地讨论了当前局限性和改进方向（WebRTC、端到端语音模型）。

---

## Day 5：模拟面试 + 查缺补漏（~4h）

### 5.1 高频面试问题自测

准备以下问题的 2 分钟回答：

**Spring AI 相关**：
1. Spring AI 2.0 和 1.x 有什么变化？
2. `ChatClient` vs `ChatModel` 的区别？
3. Spring AI Agent Utils 是什么？项目中怎么用的？
4. Advisors 的作用？

**RAG 相关**：
1. RAG 的完整流程？
2. 如何评估 RAG 的检索质量？
3. Chunk 大小对检索效果的影响？
4. Query Rewrite 为什么重要？
5. 如何处理检索到的文档和用户问题的相关性？

**LLM 工程化**：
1. Prompt 模板管理的实践？
2. 结构化输出的可靠性如何保证？
3. LLM 调用失败的降级策略？
4. Token 消耗如何优化？
5. 如何防止 Prompt 注入？

**Java 21 特性**：
1. 虚拟线程是什么？项目里怎么用？
2. Record 类型在项目中的使用场景？
3. 虚拟线程和平台线程的区别？

### 5.2 代码走读练习

挑 3 个核心链路，不看代码画出时序图：
1. **简历分析链**：上传 → 异步分析 → LLM 调用 → 报告生成
2. **模拟面试链**：选择 Skill → 出题 → 回答 → 评估 → 报告
3. **RAG 问答链**：提问 → Query Rewrite → 向量检索 → Context → SSE 生成

### 5.3 查缺补漏

- [ ] 能讲清楚 pgvector 的索引类型（IVFFlat vs HNSW）及选择理由？
- [ ] 能讲清楚 Redis Stream 的消费者组和消息确认机制？
- [ ] 能讲清楚 WebSocket 的心跳保活和重连机制？
- [ ] 能讲清楚 Docker Compose 的服务编排？
- [ ] 前端关键页面能说出数据流？

---

## 速查：项目核心文件地图

```
interview-guide/
├── AGENTS.md                          # ← 入口：所有编码规范
├── CLAUDE.md                          # ← Claude Code 配置
├── .env.example                       # ← 环境变量参考
├── docker-compose.dev.yml             # ← 本地基础设施
│
├── app/
│   ├── build.gradle                   # ← 依赖和插件
│   └── src/main/java/interview/guide/
│       ├── App.java                   # ← 启动类
│       ├── common/                    # ← 横切能力
│       │   ├── ai/                    # ★ AI 核心
│       │   │   ├── LlmProviderRegistry.java
│       │   │   ├── StructuredOutputInvoker.java
│       │   │   ├── ApiPathResolver.java
│       │   │   ├── AgentUtilsConfiguration.java
│       │   │   ├── PromptSanitizer.java
│       │   │   └── PromptSecurityConstants.java
│       │   ├── annotation/RateLimit.java    # ★ 限流注解
│       │   ├── aspect/RateLimitAspect.java  # ★ 限流切面
│       │   ├── async/                       # ★ 异步模板
│       │   │   ├── AbstractStreamProducer.java
│       │   │   └── AbstractStreamConsumer.java
│       │   ├── evaluation/                  # ★ 统一评估
│       │   │   ├── UnifiedEvaluationService.java
│       │   │   ├── EvaluationReport.java
│       │   │   └── QaRecord.java
│       │   ├── exception/                   # 异常体系
│       │   ├── config/                      # 配置类
│       │   └── result/Result.java           # 统一响应
│       ├── infrastructure/              # 技术基础设施
│       └── modules/                     # 业务模块
│           ├── resume/                  # 简历管理
│           ├── interview/               # 模拟面试
│           ├── interviewschedule/       # 面试安排
│           ├── knowledgebase/           # 知识库 (RAG)
│           ├── llmprovider/             # 多模型管理
│           └── voiceinterview/          # 语音面试
│
├── frontend/src/                       # ← React 前端
│   ├── pages/                          # 页面组件
│   ├── components/                     # 可复用组件
│   ├── api/                            # API 调用层
│   └── types/                          # TypeScript 类型
│
└── docs/                               # 项目文档
```

---

## 学习技巧

1. **带着问题读代码**：每个模块先想 3 个"为什么这样做"，再去源码中找答案
2. **画图 > 死记**：核心链路的时序图比背类名有效得多
3. **讲出来**：每学完一个模块，用 3 分钟口头总结给空气听
4. **对比学习**：这个项目用 Redis Stream，想想换成 Kafka 会有什么不同
5. **面试视角**：读代码时始终想"面试官会怎么问这个问题"
