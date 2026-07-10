# 🎯 InterviewGuide 源码精读 — Spring AI + RAG 全栈项目学习

> 精读 [Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide)
> 源码，深入理解 Spring AI 集成、RAG 检索增强、异步任务编排与实时语音通信的工程实践。

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
| Day 1 | 环境搭建 + 架构概览 | Docker Compose / DataGrip / pgvector 配置 | 🔄 |
| Day 2 | AI 核心链路 | 多 Provider 管理 / 结构化输出 / Prompt 工程 | 🔄 |
| Day 3 | RAG + 异步 + 语音 | pgvector 检索 / Redis Stream / WebSocket 语音 | 🔄 |
| Day 4 | 工程化实践 | 限流 / 安全 / 架构决策 | ⬜ |
| Day 5 | 查缺补漏 | 全链路走读 + 知识体系梳理 | ⬜ |

详细计划 → [LEARNING_PLAN.md](LEARNING_PLAN.md)

---

## 🧠 核心收获（持续更新）

### ⚙️ Day 1：环境搭建 + 架构概览

| # | 主题 | 笔记 | 关键收获 |
|---|------|------|----------|
| 1 | 本地环境搭建 | [📝](my-learning/notes/01-env-setup.md) | Docker Compose 一键启动 PG+pgvector+Redis+RustFS |

### 🧩 Day 2：AI 核心链路

| # | 主题 | 笔记 | 关键收获 |
|---|------|------|----------|
| 2 | Spring AI 多 Provider 管理 | [📝](my-learning/notes/02-spring-ai-provider.md) | 策略模式 + 动态切换 + 结构化输出 |
| 3 | 统一评估引擎 | [📝](my-learning/notes/03-unified-evaluation.md) | 分批评估 + 二次汇总 + 降级兜底 |

### 🔗 Day 3：RAG + 异步 + 语音

| # | 主题 | 笔记 | 关键收获 |
|---|------|------|----------|
| 4 | RAG 检索增强全链路 | [📝](my-learning/notes/04-rag-pipeline.md) | 文档→向量→检索→生成全链路 |
| 5 | Redis Stream 异步任务 | [📝](my-learning/notes/05-redis-stream-async.md) | 生产者/消费者模板 + 重试 + 死信 |
| 6 | 实时语音通信 | [📝](my-learning/notes/06-voice-interview.md) | WebSocket + VAD + 并发 TTS |

→ 更多笔记见 [my-learning/notes/](my-learning/notes/)

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

在职前端开发，日常工作以 React + TypeScript 为主。正在向 AI 全栈方向扩展能力边界，通过精读 Spring AI + RAG 项目源码，补齐后端架构和 LLM 工程化落地的认知拼图。

*最后更新：2026.07.10*
