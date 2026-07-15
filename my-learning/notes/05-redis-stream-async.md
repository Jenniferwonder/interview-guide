# Redis Stream 异步任务模板

> 对应源码：`common/async/AbstractStreamProducer.java`、`AbstractStreamConsumer.java`、`AsyncTaskStreamConstants.java`、各模块 `listener/` 包

## 设计目标

将耗时操作（简历分析、面试评估、文档向量化）从请求线程中剥离，通过 Redis Stream 解耦生产与消费。

## 核心模板类

### AbstractStreamProducer\<T\>

```java
// 使用方式：继承并调用 send(t)
public abstract class AbstractStreamProducer<T> {
    // 1. 创建 Stream（如不存在）
    // 2. 序列化消息为 JSON
    // 3. XADD 发送到 Stream
    // 4. 可选：设置消息 TTL
}
```

### AbstractStreamConsumer\<T\>

```java
public abstract class AbstractStreamConsumer<T> {
    // 1. 创建消费者组
    // 2. XREADGROUP 拉取 Pending 消息
    // 3. 反序列化 → 调用 processMessage()
    // 4. 成功 → XACK
    // 5. 失败 → 重试（最多 3 次）→ 标记 FAILED
}
```

## 任务生命周期

```
PENDING → 消费者拉取 → PROCESSING
                         ├── 成功 → XACK → 删除
                         └── 失败
                              ├── retry < 3 → 重新 PENDING
                              └── retry >= 3 → FAILED
```

消费前先校验实体是否存在：
- 实体已删除 → ACK 丢弃（避免死循环）
- 实体存在 → 正常处理

## Stream 定义

所有 Stream Key、消费者组、任务状态常量集中在 `AsyncTaskStreamConstants`：

```
resume:analyze      → 简历分析
interview:evaluate  → 面试评估
voice:evaluate      → 语音面试评估
knowledge:vectorize → 文档向量化
```

## 关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 消息队列 | Redis Stream | 已有 Redis，不想引入 Kafka/RabbitMQ |
| 消费者组 | 按业务模块分组 | 同组内负载均衡，不同组独立消费 |
| 重试次数 | 3 次 | 超过即标记 FAILED，人工介入 |
| 序列化 | JSON | 简单可读，调试方便 |

## Pending 回收与 XAUTOCLAIM

源码：`infrastructure/redis/RedisService.reclaimPendingMessages()` → Redisson `RStream.autoClaim(...)`。

### 作用

消费者组里若有消息被取出后长时间未 ACK（进程崩溃、超时），会进入 Pending 列表。`XAUTOCLAIM` 按空闲时间把这些消息认领到当前消费者，避免任务永久卡在 Pending。

### 版本要求

- `XAUTOCLAIM` 自 **Redis 6.2** 引入。
- 项目 Docker 镜像为 `redis:7`，本身支持。
- 若启动报 `ERR unknown command 'XAUTOCLAIM'`，通常不是 Redisson 4.0 的问题，而是 **应用连到了更旧的 Redis 实例**（例如本机 Windows 老 Redis 占用了 `localhost:6379`）。

### 验证（宿主机，勿只信 docker exec）

```powershell
redis-cli -h 127.0.0.1 -p 6379 info server | findstr redis_version
redis-cli -h 127.0.0.1 -p 6379 command info xautoclaim
```

`docker exec interview-redis redis-cli INFO` 只能证明容器内版本；bootRun 在宿主机时连的是 `localhost:6379` 上**实际响应**的那个进程。

环境侧诊断与处理见 [01 环境搭建 §问题 3](01-env-setup.md)。

## 核心要点

- **Redis Stream vs Kafka**：Stream 部署零成本，适合中小规模；Kafka 持久化更强、吞吐更高，适合大规模事件流
- **消费者组机制**：同组内消息只被一个消费者处理，天然负载均衡；不同组独立消费同一 Stream，互不干扰
- **消息可靠性**：ACK 确认 + Pending 消息恢复（`XAUTOCLAIM`）+ 最多 3 次重试 + FAILED 死信状态，形成完整兜底链路
- **为什么不是 `@Async`**：`@Async` 依赖 JVM 内存队列，重启丢消息、无持久化；Stream 消息落盘 Redis，重启可恢复
- **连对 Redis**：Stream 高级命令对版本敏感；本地多 Redis 并存时，以宿主机 `redis-cli` 验证为准
