# 🎯 InterviewGuide 源码学习 — AI 应用开发工程师

> 通过精读 [Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide)
> 源码，掌握 Spring AI + RAG + 多模态全栈 AI 应用开发的核心能力。

[![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1-green?logo=springboot)](https://spring.io/projects/spring-boot)
[![Spring AI](https://img.shields.io/badge/Spring%20AI-2.0-blue)](https://spring.io/projects/spring-ai)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-336791?logo=postgresql)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-Stream-DC382D?logo=redis)](https://redis.io/)
[![React](https://img.shields.io/badge/React-18.3-61DAFB?logo=react)](https://react.dev/)

---

## 📖 我在做什么

这是一个**代码精读项目**。我 fork 了 InterviewGuide（一个基于大语言模型的智能面试平台），逐模块分析其架构设计、AI 集成方式、工程化实践，并输出：

- 📝 **学习笔记**：每个核心模块的设计思路和关键代码分析
- 📊 **架构图**：核心链路的时序图和数据流图
- 🛠️ **代码验证**：动手验证关键设计（如新增 Provider、修改 Prompt 模板）

---

## 🗺️ 学习路线

| 阶段 | 主题 | 产出 | 状态 |
|------|------|------|------|
| Day 1 | 环境搭建 + 架构概览 | 跑通全链路，理解模块划分 | ⬜ |
| Day 2 | AI 核心链路 | 多 Provider 管理 / 结构化输出 / Prompt 工程 | ⬜ |
| Day 3 | RAG + 异步 + 语音 | pgvector 检索 / Redis Stream / WebSocket 语音 | ⬜ |
| Day 4 | 工程化实践 | 限流 / 安全 / 架构决策 / 面试表达 | ⬜ |
| Day 5 | 模拟面试 | 高频题自测 / 核心链路走读 | ⬜ |

详细计划 → [LEARNING_PLAN.md](LEARNING_PLAN.md)

---

## 🧠 核心收获（持续更新）

### Spring AI 多 Provider 管理
`LlmProviderRegistry` 策略模式设计，支持 DashScope / Kimi / DeepSeek / GLM 等 Provider
动态切换，新增 Provider 仅需配置。配合 `StructuredOutputInvoker` 保证 LLM 输出可靠性。

### RAG 检索增强全链路
Tika 文档解析 → 分块策略 → pgvector 向量化 → Query Rewrite → 相似度检索 →
SSE 流式生成。关键决策：用 pgvector 而非独立向量库，精简架构。

### Redis Stream 异步任务模板
`AbstractStreamProducer` / `AbstractStreamConsumer` 模板化解耦，
3 次重试 + 死信机制，消费前校验实体存在性。

### 统一面试评估引擎
文字/语音共用 `UnifiedEvaluationService`，"分批评估 → 结构化输出 → 二次汇总 → 降级兜底"
解决 Token 限制并保证评估质量。

### 实时语音面试
WebSocket + 服务端 VAD + 句子级并发 TTS，首包延迟 200ms。
有意识地讨论局限性和改进方向（WebRTC、端到端语音模型）。

---

## 📂 目录说明

```
├── README.md              👈 你正在看的页面（学习产出首页）
├── LEARNING_PLAN.md       详细 5 天学习计划
├── my-learning/           学习产出
│   ├── notes/             学习笔记（按模块）
│   ├── diagrams/          架构图 / 时序图
│   └── code-changes/      代码验证 / 改动
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
# 从原仓库拉取最新代码（不影响学习产出分支）
git checkout master && git pull upstream master

# 将上游更新合并到学习分支
git checkout my-learning && git merge master
```

---

## 🙋 关于我

AI 应用开发工程师求职中。通过本项目深度学习 LLM 工程化落地的完整链路。

*最后更新：2026.07.10*
