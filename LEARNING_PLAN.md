# 学习行动计划 — 工程级代码交付

> **学习方向**：从前端扩展到 AI 全栈
> **组织方式**：7 个工程级交付物，每个是完整的"发现问题 → 改动 → 验证"闭环

---

## 七个交付物总览

```
交付物                   核心问题
──────────────────────────────────────────────────────────────────────
0  环境搭建              "怎么把全套基础设施跑起来？"
1  LLM 调用可观测性      "大模型调用延迟、成功率、Token 消耗去哪看？"
2  简历分析全链路测试    "异步任务链路怎么保证每个环节正确？"
3  RAG 检索质量评估闭环  "怎么知道检索到的文档对用户有没有用？"
4  Prompt A/B 测试框架   "两个 Prompt 哪个更好？怎么量化？"
5  语音面试延迟诊断      "端到端延迟偏高的瓶颈到底在哪一段？"
6  SSE 断连自动重连      "网络闪断导致回答丢失怎么办？"
```

---

## 交付物 0：本地环境一键启动

### 链路

```
Docker Compose → PG+pgvector + Redis + RustFS → .env 配置 → 前后端启动
```

### 必读文件

| 文件 | 关注点 |
|------|--------|
| `docker-compose.dev.yml` | 3 个服务的编排方式、健康检查、数据卷 |
| `.env.example` | 所有环境变量全景 |
| `application.yml` | Spring Boot 如何通过 `${VAR:default}` 读取配置 |
| `docker/postgres/init.sql` | 数据库初始化脚本 |

### 产出

→ `my-learning/code-changes/00-env-setup/` + `my-learning/notes/01-env-setup.md`

### 踩坑记录

| 问题 | 根因 | 解决 |
|------|------|------|
| RustFS S3 返回 500 | 系统 HTTP_PROXY 拦截了 localhost 请求 | `NO_PROXY=localhost,127.0.0.1` |
| Java 版本不匹配 | 系统默认 Java 8，项目需要 Java 21 | 切换到 Scoop 安装的 `temurin21-jdk` |
| Java 21 未生效 | `JAVA_HOME` 被系统变量覆盖 | 使用绝对路径 `$JAVA_HOME/bin/java` |

---

## 交付物 1：LLM 调用可观测性

### 核心问题

> "项目接了多个 LLM Provider，但谁也不知道每个 Provider 的调用延迟有多少、Token 消耗多少钱、失败率多少、结构化输出重试了多少次。"

### 工程方案

在 `LlmProviderRegistry` 中嵌入 Micrometer 指标，利用项目已有的 Micrometer 依赖，不引入新库：

| 指标 | 来源 | 用途 |
|------|------|------|
| `llm.calls.count{provider,model,result}` | `LlmProviderRegistry` 包装层 | 各 Provider 调用次数和成功率 |
| `llm.calls.duration{provider,model}` | 同上 | P50/P95/P99 延迟分布 |
| `llm.tokens.used{provider,model,type}` | `ChatClient` 返回的 Usage 元数据 | Token 消耗统计 |
| `llm.structured.retries{provider}` | `StructuredOutputInvoker` | 结构化输出重试次数 |

### 改动范围

| 文件 | 改动 |
|------|------|
| `common/ai/LlmProviderRegistry.java` | 在调用路径中包装 Timer + Counter |
| `common/ai/StructuredOutputInvoker.java` | 在重试循环中增加 retries 计数器 |
| `application.yml` | 暴露 `/actuator/metrics` 端点 |

### 预期产出

1. 触发几次 LLM 调用 → 访问 `/actuator/metrics/llm.calls.duration` 拉数据
2. 输出一组 Grafana 查询语句（如果后续接 Prometheus + Grafana 可直接使用）
3. 笔记：`my-learning/notes/02-spring-ai-provider.md` 补充可观测性部分

---

## 交付物 2：简历分析全链路集成测试

### 核心问题

> "简历分析依赖 S3 上传和 LLM 调用两个外部服务，现有测试只测单个 Service，没有任何测试能验证'上传 → Stream → 消费 → LLM → 落库'的完整异步链路。"

### 工程方案

用 `@SpringBootTest` + `@TestConfiguration` 启动完整 Spring 上下文（含 Redis Stream），Mock S3 和 LLM 两个外部依赖，驱动完整异步链路验证：

```
测试流程:
1. POST /api/resume/upload → 断言 200 + taskId
2. 查询 Redis Stream 断言消息已入队
3. Consumer 拉取消息 → Mock Tika 返回固定文本 → Mock LLM 返回固定分析结果
4. 断言 ResumeEntity: PENDING → PROCESSING → COMPLETED
5. 重复上传 → 断言去重逻辑生效

异常场景:
6. Mock LLM 返回非 JSON → 断言 3 次重试后状态为 FAILED
```

### 改动范围

| 文件 | 改动 |
|------|------|
| `app/build.gradle` | 确认 `spring-boot-starter-test` 依赖 |
| `app/src/test/` 新增 `ResumeAnalysisIntegrationTest.java` | 核心测试 |
| `app/src/test/` 新增 `TestConfig.java` | Mock 配置（S3Client + ChatClient） |

### 预期产出

1. `./gradlew :app:test --tests ResumeAnalysisIntegrationTest` 全部通过
2. 笔记：`my-learning/notes/05-redis-stream-async.md` 补充集成测试部分

---

## 交付物 3：RAG 检索质量评估闭环

### 核心问题

> "RAG 问答用 Query Rewrite + pgvector 检索后拼到 LLM，但没人知道检索到的文档片段到底对回答有没有帮助。"

### 工程方案

在 RAG 回答完成后，前端添加 👍/👎 反馈按钮。点击后 POST 到新增评分接口，落库到 `rag_chat_feedback` 表。积累数据后可按知识库、查询类型分析检索质量，反向优化分块策略和 TopK 参数。

### 改动范围

| 文件 | 改动 |
|------|------|
| `modules/knowledgebase/model/` 新增 `RagChatFeedbackEntity.java` | 评分实体 |
| `modules/knowledgebase/repository/` 新增 `RagChatFeedbackRepository.java` | JPA Repository |
| `modules/knowledgebase/RagChatController.java` | 新增 `POST /api/rag-chat/{messageId}/feedback` |
| `frontend/src/pages/KnowledgeBaseQueryPage.tsx` | 每条回答下方添加 👍👎 按钮 |
| `frontend/src/api/ragChat.ts` | 新增 `submitFeedback()` API |

### 数据模型

```sql
rag_chat_feedback:
  id, message_id, session_id, knowledge_base_id,
  rating, comment, query_rewritten, top_k_doc_ids, created_at
```

### 预期产出

1. 前端截图：RAG 回答下方出现评分按钮
2. DB 中有评分记录
3. 笔记：`my-learning/notes/04-rag-pipeline.md` 补充质量评估部分

---

## 交付物 4：Prompt A/B 测试框架

### 核心问题

> "改了一版 Prompt，但不知道新版本比旧版本好还是坏。凭感觉判断不靠谱。"

### 工程方案

实现 `PromptAbTestService`：传入同一个输入、两个版本的 Prompt 模板，并行调用 LLM 获取结果，再用另一个 LLM 调用进行结构化评分（准确性/完整性/相关性三维度），自动选出优胜者。

### 改动范围

| 文件 | 改动 |
|------|------|
| `common/ai/` 新增 `PromptAbTestService.java` | A/B 测试核心逻辑 |
| `resources/prompts/prompt-ab-evaluation.st` | 评分 Prompt 模板 |
| `common/ai/` 新增 `PromptAbTestRequest.java` / `PromptAbTestResult.java` | record 定义 |

### 核心方法

```java
public PromptAbTestResult compare(
    String input,
    String promptTemplateA, String promptTemplateB,
    String abEvaluationPrompt
) {
    // 1. 并行调用两版 Prompt
    var fA = CompletableFuture.supplyAsync(() -> chatClient.call(templateA, input));
    var fB = CompletableFuture.supplyAsync(() -> chatClient.call(templateB, input));
    var outA = fA.join(); var outB = fB.join();

    // 2. LLM 三维度评分
    var scores = structuredOutputInvoker.invoke(abEvaluationPrompt,
        Map.of("input", input, "outputA", outA, "outputB", outB), AbScores.class);

    // 3. 返回对比结论
    return new PromptAbTestResult(outA, outB, scores,
        scores.total(A) > scores.total(B) ? "A" : "B");
}
```

### 预期产出

1. 选一个已有 Prompt（如简历分析），写对照版本，跑 A/B 测试
2. 输出：A 版得分 / B 版得分 / 优胜者
3. 笔记：`my-learning/notes/03-unified-evaluation.md` 补充 A/B 测试部分

---

## 交付物 5：语音面试延迟诊断

### 核心问题

> "项目已知端到端延迟偏高，但不知道是哪一段慢——ASR 慢？LLM 生成慢？TTS 合成慢？没有分阶段数据，没法精准优化。"

### 工程方案

在语音面试的三个关键节点利用项目已有的 Micrometer 加 `@Timed` 埋点：

| 埋点位置 | 指标名 | 测量内容 |
|----------|--------|----------|
| `QwenAsrService.transcribe()` | `voice.asr.duration` | 单句语音识别耗时 |
| `DashscopeLlmService.generate()` | `voice.llm.duration` | LLM 生成回答耗时 |
| `QwenTtsService.synthesize()` | `voice.tts.duration` | 单句 TTS 合成耗时 |
| `VoiceInterviewWebSocketHandler` | `voice.e2e.duration` | 用户说完 → 开始播放端到端延迟 |

### 改动范围

| 文件 | 改动 |
|------|------|
| `modules/voiceinterview/QwenAsrService.java` | 加 `@Timed("voice.asr")` |
| `modules/voiceinterview/QwenTtsService.java` | 加 `@Timed("voice.tts")` |
| `modules/voiceinterview/DashscopeLlmService.java` | 加 `@Timed("voice.llm")` |
| `modules/voiceinterview/VoiceInterviewWebSocketHandler.java` | 加端到端 Timer |

### 预期产出

1. 完成语音面试 → 访问 `/actuator/metrics` 拉取各段延迟数据
2. 输出延迟分布：ASR / LLM / TTS 各段的 avg / p95 / p99
3. 瓶颈结论 + 改善建议
4. 笔记：`my-learning/notes/06-voice-interview.md` 补充延迟诊断部分

---

## 交付物 6：SSE 断连自动重连 + 断点续传

### 核心问题

> "知识库问答采用 SSE 流式输出，但网络闪断后 `EventSource` 直接中断，前端没有任何恢复逻辑，用户只能刷新页面重来，已回答内容全部丢失。"

### 工程方案

**后端**：RagChat SSE 端点支持 `Last-Event-ID` 请求头，识别断点后从对应位置继续推送。

**前端**：抽取通用 `useSSE` Hook，实现：
- `EventSource` 原生 `Last-Event-ID` 自动续传
- Exponential Backoff 重连：1s → 2s → 4s → 8s → 16s（上限）
- 已渲染消息保持显示，重连成功后追加新内容
- 超最大重试次数后弹提示

### 改动范围

| 文件 | 改动 |
|------|------|
| `modules/knowledgebase/RagChatController.java` | SSE 支持 `Last-Event-ID` 断点续推 |
| `frontend/src/hooks/` 新增 `useSSE.ts` | 通用 SSE Hook（含重连 + 续传） |
| `frontend/src/api/ragChat.ts` | 从内联 `EventSource` 切换到 `useSSE` |
| `frontend/src/pages/KnowledgeBaseQueryPage.tsx` | 重连状态提示 UI |

### 预期产出

1. 断网 → 前端显示"重连中…" → 恢复 → 自动续传，之前内容不丢失
2. Chrome DevTools Network 截图：`Last-Event-ID` 衔接
3. 笔记：`my-learning/notes/04-rag-pipeline.md` 补充 SSE 可靠性部分

---

## 笔记索引

```
交付物 0 → my-learning/code-changes/00-env-setup/ + my-learning/notes/01-env-setup.md ✅
交付物 1 → my-learning/notes/02-spring-ai-provider.md + my-learning/code-changes/01-llm-observability/
交付物 2 → my-learning/notes/05-redis-stream-async.md  + my-learning/code-changes/02-resume-integration-test/
交付物 3 → my-learning/notes/04-rag-pipeline.md        + my-learning/code-changes/03-rag-feedback-loop/
交付物 4 → my-learning/notes/03-unified-evaluation.md  + my-learning/code-changes/04-prompt-ab-test/
交付物 5 → my-learning/notes/06-voice-interview.md     + my-learning/code-changes/05-voice-latency-diagnosis/
交付物 6 → my-learning/notes/04-rag-pipeline.md        + my-learning/code-changes/06-sse-reconnect/
```

---

## 推荐执行顺序

```
前端优势（先做）:              通用能力（第二波）:           深层积累（第三波）:
──────────────────────     ──────────────────────      ──────────────────────
⑥ SSE 断连自动重连         ① LLM 调用可观测性          ② 简历分析全链路集成测试
（前端 Hook + 后端小改）    （Micrometer 不引入新库）    （Spring Test + Mock 体系）

                           ③ RAG 检索质量评估闭环       ④ Prompt A/B 测试框架
                           （全栈改动，前后端联动）      （策略模式 + 抽象设计）

                           ⑤ 语音面试延迟诊断
                           （埋点 + 数据分析）
```
