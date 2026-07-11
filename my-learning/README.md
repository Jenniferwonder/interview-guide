# 学习产出索引

> **学习方向**：从前端扩展到 AI 全栈
> **学习周期**：2026.07.10 起
> **原项目**：[Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide)

---

## 交付物路线

完整计划见 [../LEARNING_PLAN.md](../LEARNING_PLAN.md)。

| # | 交付物 | 核心问题 | 产出 |
|---|--------|----------|:--:|
| 0 | 本地环境一键启动 | 怎么把全套基础设施跑起来？ | ✅ [→](notes/01-env-setup.md) |
| 1 | LLM 调用可观测性 | 大模型调用的延迟、成功率、Token 消耗去哪看？ | ⬜ |
| 2 | 简历分析全链路集成测试 | 异步任务链路怎么保证每个环节正确？ | ⬜ |
| 3 | RAG 检索质量评估闭环 | 怎么知道检索到的文档对用户有没有用？ | ⬜ |
| 4 | Prompt A/B 测试框架 | 两个 Prompt 哪个更好？怎么量化？ | ⬜ |
| 5 | 语音面试延迟诊断 | 端到端延迟偏高的瓶颈到底在哪一段？ | ⬜ |
| 6 | SSE 断连自动重连 | 网络闪断导致回答丢失怎么办？ | ⬜ |

---

## 支撑笔记

| 笔记 | 对应交付物 |
|------|-----------|
| [01 环境搭建](notes/01-env-setup.md) | 交付物 0 |
| [02 Spring AI 多 Provider 管理](notes/02-spring-ai-provider.md) | 交付物 1 |
| [03 统一评估引擎](notes/03-unified-evaluation.md) | 交付物 3、4 |
| [04 RAG 检索增强全链路](notes/04-rag-pipeline.md) | 交付物 3、6 |
| [05 Redis Stream 异步任务](notes/05-redis-stream-async.md) | 交付物 2 |
| [06 实时语音通信](notes/06-voice-interview.md) | 交付物 5 |

---

## 代码改动

按交付物组织，见 [code-changes/](code-changes/)。

| 目录 | 对应交付物 | 状态 |
|------|-----------|:--:|
| `code-changes/00-env-setup/` | 环境搭建 + 踩坑记录 | ✅ |
| `code-changes/01-llm-observability/` | LLM 调用可观测性 | ⬜ |
| `code-changes/02-resume-integration-test/` | 简历分析全链路集成测试 | ⬜ |
| `code-changes/03-rag-feedback-loop/` | RAG 检索质量评估闭环 | ⬜ |
| `code-changes/04-prompt-ab-test/` | Prompt A/B 测试框架 | ⬜ |
| `code-changes/05-voice-latency-diagnosis/` | 语音面试延迟诊断 | ⬜ |
| `code-changes/06-sse-reconnect/` | SSE 断连自动重连 | ⬜ |
