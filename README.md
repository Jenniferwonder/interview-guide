# 🎯 interview-guide 源码精读 — 从前端到 AI 全栈

> Forked from [Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide)
> —— 一个集成了简历分析、模拟面试、RAG 知识库和实时语音的智能面试平台。
>
> 以**业务链路为线索**追踪 Spring AI + RAG + 异步任务的全栈工程实践，
> 通过有深度的**工程级代码改动**来验证每条链路的技术设计。

[![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1-green?logo=springboot)](https://spring.io/projects/spring-boot)
[![Spring AI](https://img.shields.io/badge/Spring%20AI-2.0-blue)](https://spring.io/projects/spring-ai)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-336791?logo=postgresql)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-Stream-DC382D?logo=redis)](https://redis.io/)
[![React](https://img.shields.io/badge/React-18.3-61DAFB?logo=react)](https://react.dev/)

---

## 📖 我在做什么

这是我对 [Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide) 的**学习型 fork**。

我将项目拆解为 **7 个工程级交付物**，每个都超越"加个配置就能跑"的层面，而是**发现真实问题 → 加指标/日志 → 定位根因 → 写代码改 → 验证效果 → 输出数据**的完整工程闭环。

输出形式：
- 🛠️ **代码改动**：可追溯的 git diff，每个交付物有完整的实现 + 测试 + 效果数据
- 📝 **支撑笔记**：每个改动对应的源码分析链路和设计思路
- 🐛 **踩坑记录**：环境搭建中的实际问题诊断与修复过程

---

## 🗺️ 交付物路线

> 按项目的**三大核心功能 + 两大技术亮点**组织，每个交付物是一个完整的"发现问题 → 改动 → 验证"闭环。

| # | 交付物 | 核心问题 | 工程深度 | 状态 |
|---|--------|----------|----------|:--:|
| 0 | 本地环境一键启动 | 怎么把全套基础设施跑起来？ | Docker Compose 编排 + 代理问题排查 → [📝](my-learning/notes/01-env-setup.md) | ✅ |
| 1 | LLM 调用可观测性 | 大模型调用延迟、成功率、Token 消耗去哪看？ | Micrometer 指标埋点 + Grafana 查询模板 → [📝](my-learning/notes/02-spring-ai-provider.md) | ⬜ |
| 2 | 简历分析全链路集成测试 | 异步任务链路怎么保证每个环节正确？ | Mock S3 + Mock LLM + 验证 Stream 状态流转 → [📝](my-learning/notes/05-redis-stream-async.md) | ⬜ |
| 3 | RAG 检索质量评估闭环 | 怎么知道检索到的文档对用户有没有用？ | 新增评分接口 + 落库 + 用于 Prompt 迭代 → [📝](my-learning/notes/04-rag-pipeline.md) | ⬜ |
| 4 | Prompt A/B 测试框架 | 两个 Prompt 哪个更好？怎么量化？ | 并行调用 + 结构化评分 + 自动选出更优版本 → [📝](my-learning/notes/03-unified-evaluation.md) | ⬜ |
| 5 | 语音面试延迟诊断 | 端到端延迟偏高的瓶颈到底在哪一段？ | ASR/TTS/LLM 三段分别埋点 + 瓶颈定位报告 → [📝](my-learning/notes/06-voice-interview.md) | ⬜ |
| 6 | SSE 断连自动重连 | 网络闪断导致回答丢失怎么办？ | Exponential Backoff + Last-Event-ID + 断点续传 → [📝](my-learning/notes/04-rag-pipeline.md) | ⬜ |

→ 详细追踪计划见 [LEARNING_PLAN.md](LEARNING_PLAN.md)

---

## 🧠 各交付物涉及的源码范围

### 交付物 1：LLM 调用可观测性

在 `LlmProviderRegistry` 和 `StructuredOutputInvoker` 中嵌入 Micrometer 指标，覆盖：Token 消耗分布（按 Provider 和模型）、调用延迟（P50/P95/P99）、结构化输出成功率与重试次数分布、Provider 连接失败率。最终输出一组 Grafana dashboard 查询语句。

- 改动范围：`common/ai/LlmProviderRegistry.java` · `common/ai/StructuredOutputInvoker.java`
- 从"能接到 LLM"到"能监控 LLM"

### 交付物 2：简历分析全链路集成测试

不测单个 Service，而是 Mock 掉 S3 和 LLM 这两个外部依赖后，驱动"上传 → 保存 → 发送 Stream → 消费者拉取 → 调用 LLM → 落库 → 状态变更"的完整链路的集成测试，验证异步任务 `PENDING → PROCESSING → COMPLETED` 和 `PENDING → PROCESSING → FAILED` 两条状态流转路径。

- 改动范围：`app/src/test/` 新增集成测试 · 可能需要抽取 Mock 友好的接口
- 从"测一个类"到"测一条链路"

### 交付物 3：RAG 检索质量评估闭环

在 `RagChatController` 中添加一个"这条回答是否有用"的评分接口（👍/👎 + 可选文字反馈），落库到 `rag_chat_feedback` 表。评分数据用于后续分析：哪些 Query Rewrite 效果好、哪些知识库文档检索到但未被采纳、TopK 和阈值是否需要调整。

- 改动范围：`modules/knowledgebase/RagChatController.java` · 新增 Entity/Repository · 前端 `KnowledgeBaseQueryPage.tsx` 添加评分按钮
- 从"能检索"到"能迭代"

### 交付物 4：Prompt A/B 测试框架

实现一个通用的 Prompt 对比工具——同时向同一个 LLM 发送两版 Prompt 获取结果，通过另一个 LLM 调用对两个结果进行结构化评分（准确性/完整性/相关性），自动选出更优版本。可以用于简历分析、面试评估、RAG 问答等任意 Prompt 场景。

- 改动范围：`common/ai/` 新增 `PromptAbTestService.java` · 配套的评分 Prompt 模板
- 从"改一次 Prompt 凭感觉看效果"到"用数据选出最优 Prompt"

### 交付物 5：语音面试延迟诊断

利用项目中已有的 Micrometer 基础，在 `QwenAsrService`、`QwenTtsService`、`DashscopeLlmService` 三个关键节点埋点，记录每段的起始到结束耗时。通过 `/actuator/metrics` 端点拉取数据，输出延迟分布表格和瓶颈定位结论，包括**改善建议**（如调整 TTS 并发数、ASR 批量大小等参数）。

- 改动范围：`modules/voiceinterview/QwenAsrService.java` · `QwenTtsService.java` · `DashscopeLlmService.java`
- 从"知道延迟偏高"到"知道每一段各自占多少毫秒"

### 交付物 6：SSE 断连自动重连 + 断点续传

当前知识库问答使用 SSE 流式输出，但前端无重连逻辑——网络闪断后用户只能刷新页面重试。利用 SSE 协议的 `Last-Event-ID` 机制和后端的消息 ID 标记，实现：断开后 Exponential Backoff 自动重连（1s → 2s → 4s → 8s），从断点继续推送后续内容，已渲染的部分不丢失。

- 改动范围：`frontend/src/api/ragChat.ts` · `frontend/src/hooks/` 新增 `useSSE.ts` · 后端 `RagChatController.java` 支持 `Last-Event-ID`
- **前端视角的独特加分项**——大部分后端开发者不会关注这里的用户体验

---

## 📂 目录说明

```
├── README.md              👈 学习产出首页
├── LEARNING_PLAN.md       每个交付物的详细追踪计划和改动指南
├── my-learning/
│   ├── notes/             支撑笔记（按交付物引用）
│   ├── code-changes/      代码改动（按交付物组织，git diff 可追溯）
│   └── demos/             独立可运行示例（可选）
│
├── app/                   原项目后端源码（Spring Boot）
├── frontend/              原项目前端源码（React）
└── ...                    其余为原项目文件
```

> 💡 **本分支（`my-learning`）**专用于学习产出展示。
> 原项目代码保留在 `master` 分支，定期从上游同步。

---

## 🔄 同步策略

```bash
git checkout master && git pull upstream master          # 从原仓库拉取最新代码
git checkout my-learning && git merge master             # 将上游更新合并到学习分支
```

---

## 🙋 关于我

在职前端开发，日常工作以 React + TypeScript 为主。正在向 AI 全栈方向扩展能力边界——以这个 Spring AI + RAG 全栈项目为载体，通过有深度的工程改动（可观测性、集成测试、A/B 测试、延迟诊断、SSE 重连）来构建从前端到后端、从业务代码到基础设施的完整认知。

*最后更新：2026.07.11*
