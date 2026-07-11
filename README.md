# 🎯 interview-guide 源码精读 — 从前端到 AI 全栈

> Forked from [Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide)
> —— 一个集成了简历分析、模拟面试、RAG 知识库和实时语音的智能面试平台。
>
> 以**业务链路为线索**追踪 Spring AI + RAG + 异步任务的全栈工程实践，
> 通过实际的**代码改动**来验证每条链路的技术设计。

[![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1-green?logo=springboot)](https://spring.io/projects/spring-boot)
[![Spring AI](https://img.shields.io/badge/Spring%20AI-2.0-blue)](https://spring.io/projects/spring-ai)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-336791?logo=postgresql)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-Stream-DC382D?logo=redis)](https://redis.io/)
[![React](https://img.shields.io/badge/React-18.3-61DAFB?logo=react)](https://react.dev/)

---

## 📖 我在做什么

这是我对 [Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide) 的**学习型 fork**。

我将项目拆解为 **6 个交付物**——环境搭建、LLM 集成层、简历分析链路、面试系统链路、RAG 知识库链路、异步引擎+工程化基础设施——每个交付物追踪一条从前端页面到后端数据库的**完整业务链路**，并通过实际改代码（新增 Provider / 新增 Skill / 修改 Prompt / 写测试）来验证理解，输出：

- 🛠️ **代码改动**：可追溯的 git diff，证明"我改过且能跑通"
- 📝 **支撑笔记**：每个改动对应的源码分析和设计思路
- 🐛 **踩坑记录**：环境搭建中的实际问题诊断与修复过程

---

## 🗺️ 学习路线

> 按项目的**三大核心功能 + 两大技术亮点**组织，每个交付物追踪一条完整的从前端到后端的业务链路。

| # | 交付物 | 链路范围 | 覆盖技术 | 状态 |
|---|--------|----------|----------|:--:|
| 0 | 本地环境一键启动 | 基础设施 | Docker Compose · PG+pgvector · Redis · RustFS → [📝](my-learning/notes/01-env-setup.md) | ✅ |
| 1 | 多 LLM Provider + 结构化输出 | LLM 集成层 | `LlmProviderRegistry` · `StructuredOutputInvoker` · API Key 加密 · Prompt 模板 → [📝](my-learning/notes/02-spring-ai-provider.md) [📝](my-learning/notes/03-unified-evaluation.md) | 🔄 |
| 2 | 智能简历分析 | 上传 → 解析 → AI 评分 → 报告导出 | Tika · Redis Stream 异步 · S3 存储 · iText 8 PDF → [📝](my-learning/notes/05-redis-stream-async.md) | ⬜ |
| 3 | 模拟面试系统 | Skill 出题 → 多轮对话 → 评估 → PDF 导出 | Skill 驱动 · Prompt 工程 · 统一评估引擎 · iText 8 → [📝](my-learning/notes/03-unified-evaluation.md) | ⬜ |
| 4 | RAG 知识库问答 | 文档上传 → 向量化 → 检索 → 流式生成 | pgvector · Query Rewrite · COSINE 检索 · SSE → [📝](my-learning/notes/04-rag-pipeline.md) | ⬜ |
| 5 | 异步任务引擎 + 工程化基础设施 | 系统横切面 | Redis Stream 模板 · `@RateLimit` · `PromptSanitizer` · 配置分层 → [📝](my-learning/notes/05-redis-stream-async.md) | ⬜ |

→ 详细说明见 [LEARNING_PLAN.md](LEARNING_PLAN.md)

---

## 🧠 核心收获

### 交付物 1：多 LLM Provider + 结构化输出

追踪 `LlmProviderRegistry` → `StructuredOutputInvoker` → Prompt 模板的完整调用链，理解"新增一个 LLM Provider 不需要改业务代码"的底层实现：

- 运行时 Provider 发现与 ChatClient 缓存机制
- 结构化输出的重试策略：LLM 返回非 JSON → 错误注入 Prompt → 让 LLM 自己修正
- API Key 加密落盘至 `~/.interview-guide/`，运行时动态加载

### 交付物 2：智能简历分析（全链路）

从 `UploadPage.tsx` 出发，追踪一次简历上传到分析报告生成的完整过程：

```
UploadPage → ResumeController → S3 上传 → Redis Stream (XADD)
  → AnalyzeConsumer (XREADGROUP) → Tika 文本提取
  → LlmProviderRegistry.getChatClient() → Prompt 模板 → LLM 分析
  → 结果落库 → 前端轮询 → AnalysisPanel 展示 → iText 8 PDF 导出
```

关键认知：异步任务**不在事务内调用 LLM**，事务只覆盖 DB 状态更新，LLM 调用在事务外通过 Redis Stream 解耦。

### 交付物 3：模拟面试系统

从 Skill 选择到评估报告生成的完整链路：

```
SKILL.md 加载 → InterviewQuestionService 出题（历史去重 + JD 适配）
  → 多轮对话 → UnifiedEvaluationService（分批评估 + 二次汇总）
  → StructuredOutputInvoker（JSON Schema 约束输出）
  → iText 8 PDF 导出
```

关键认知：文字面试和语音面试共用同一套评估引擎，差异仅在于输入来源——ASR 转写 vs 文本输入。

### 交付物 4：RAG 知识库问答

从文档上传到流式回答的全链路，覆盖 pgvector 向量检索的核心实践：

```
文档上传 → Tika 解析 → 文本分块 → Embedding (DashScope text-embedding-v4)
  → pgvector 存储 (1024 维 · COSINE 距离)
  → 用户提问 → Query Rewrite → Embedding → 向量检索 → TopK+阈值过滤
  → Context 拼接 → LLM 生成 → SSE 流式输出 → React Virtuoso 虚拟列表渲染
```

关键认知：向量存储为什么选 pgvector 而不是独立向量库——一套 PG 同时处理关系数据和向量数据，精简运维。

### 交付物 5：异步任务引擎 + 工程化基础设施

系统横切面的工程化能力，覆盖两个方向：

**异步任务引擎**：
- `AbstractStreamProducer<T>` / `AbstractStreamConsumer<T>` 模板化设计
- 消费失败最多重试 3 次，超过标记 FAILED
- 消费前校验实体存在性，已删除直接 ACK 丢弃
- Pending 消息定时恢复机制

**工程化基础设施**：
- 可重复 `@RateLimit` 注解（Global / IP / User 维度），基于 Redis Lua 滑动窗口
- `PromptSanitizer` 防止 Prompt 注入攻击
- 配置分层：`.env` → `application.yml` → `@ConfigurationProperties`
- 全局限流 + API Key 加密 + CORS + 内容安全校验

---

## 📂 目录说明

```
├── README.md              👈 学习产出首页
├── LEARNING_PLAN.md       各交付物的源码追踪计划和改动指南
├── my-learning/
│   ├── notes/             支撑笔记（按交付物引用）
│   ├── code-changes/      代码改动（git diff 可追溯）
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

在职前端开发，日常工作以 React + TypeScript 为主。正在向 AI 全栈方向扩展能力边界——通过精读 Spring AI + RAG 项目源码，追踪从前端页面到后端数据库的完整业务链路，补齐后端架构和 LLM 工程化落地的认知拼图。

*最后更新：2026.07.11*
