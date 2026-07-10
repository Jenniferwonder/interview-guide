# 我的 InterviewGuide 学习产出

> **学习方向**：从前端扩展到 AI 全栈
> **学习周期**：2026.07.10 - 2026.07.15
> **原项目**：[Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide)

---

## 学习路线

完整学习计划见 [../LEARNING_PLAN.md](../LEARNING_PLAN.md)。

| 天数 | 主题 | 笔记 |
|------|------|------|
| Day 1 | 环境搭建 + 架构概览 | — |
| Day 2 | AI 核心链路（多 Provider / 结构化输出 / Prompt 工程） | [→](notes/02-ai-core.md) |
| Day 3 | RAG + 异步 + 语音 | [→](notes/03-rag-async-voice.md) |
| Day 4 | 工程化实践 | [→](notes/04-engineering.md) |
| Day 5 | 查缺补漏 + 知识体系梳理 | — |

---

## 核心收获

### 1. Spring AI 多 Provider 管理
- `LlmProviderRegistry` 策略模式，新增 Provider 零代码改动
- `StructuredOutputInvoker` 封装重试 + JSON 修复，保证 LLM 输出可靠性
- API Key 加密落盘，运行时动态切换 Provider

### 2. RAG 检索增强全链路
- 文档解析（Tika）→ 分块策略 → pgvector 向量化（维度 1024，COSINE 距离）
- Query Rewrite → 向量检索 → TopK + 相似度阈值 → Context 拼接 → SSE 流式生成
- 选型理由：pgvector 精简架构，避免引入独立向量数据库

### 3. Redis Stream 异步任务模板
- `AbstractStreamProducer<T>` / `AbstractStreamConsumer<T>` 模板化
- 消费失败最多重试 3 次，超过标记 FAILED
- 消费前校验实体存在性，已删除直接 ACK 丢弃

### 4. 统一面试评估引擎
- 文字面试和语音面试共用 `UnifiedEvaluationService`
- 分批评估 → 结构化输出 → 二次汇总 → 降级兜底
- 解决 LLM Token 限制 + 保证评估质量

### 5. 实时语音面试
- WebSocket 全双工通信 + 服务端 VAD 断句 + 实时字幕
- 句子级并发 TTS，"边生成边合成边播放"，首包延迟 200ms
- 已知局限和改进方向（WebRTC、端到端语音模型）

---

## 架构图

| 图 | 说明 |
|----|------|
| [RAG 检索链路](diagrams/) | 文档上传 → 向量化 → 查询改写 → 检索 → 生成 |
| [异步任务流程](diagrams/) | Producer → Stream → Consumer → 重试/死信 |
| [统一评估引擎](diagrams/) | 分批评估 → 结构化输出 → 二次汇总 |

---

## 代码改动

如有代码实践/改动，见 [code-changes/](code-changes/)。

---

## 同步状态

```bash
# 拉上游最新代码
git pull upstream master

# 推到自己的 fork
git push origin master
```

最后同步上游时间：2026.07.10
