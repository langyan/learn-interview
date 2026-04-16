
以下是 Kafka 核心参数说明：

---

## Broker 配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `broker.id` | -1 | Broker 唯一标识，集群中必须唯一 |
| `listeners` | PLAINTEXT://:9092 | 监听地址和端口 |
| `log.dirs` | /tmp/kafka-logs | 日志数据存储目录，多目录用逗号分隔 |
| `num.partitions` | 1 | 新建 Topic 的默认分区数 |
| `default.replication.factor` | 1 | 默认副本数 |
| `log.retention.hours` | 168 | 消息保留时间（小时），默认 7 天 |
| `log.retention.bytes` | -1 | 每个分区最大存储字节数，-1 表示无限制 |
| `log.segment.bytes` | 1073741824 | 单个日志段文件大小，默认 1GB |
| `zookeeper.connect` | — | ZooKeeper 连接地址（旧版必填） |
| `num.network.threads` | 3 | 处理网络请求的线程数 |
| `num.io.threads` | 8 | 处理 I/O 的线程数 |

---

## Producer 配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `bootstrap.servers` | — | Kafka 集群地址，必填 |
| `acks` | 1 | 消息确认机制：`0`不等待、`1`Leader确认、`all`所有副本确认 |
| `retries` | 2147483647 | 发送失败重试次数 |
| `retry.backoff.ms` | 100 | 重试间隔（毫秒） |
| `batch.size` | 16384 | 批量发送大小（字节），默认 16KB |
| `linger.ms` | 0 | 消息延迟发送等待时间，增大可提升吞吐量 |
| `buffer.memory` | 33554432 | 生产者缓冲区大小，默认 32MB |
| `compression.type` | none | 压缩类型：`none`、`gzip`、`snappy`、`lz4`、`zstd` |
| `max.request.size` | 1048576 | 单次请求最大字节数，默认 1MB |
| `key.serializer` | — | Key 序列化类，必填 |
| `value.serializer` | — | Value 序列化类，必填 |
| `enable.idempotence` | true | 是否开启幂等性（Kafka 3.0+ 默认开启） |
| `transactional.id` | — | 事务 ID，开启事务时必填 |

---

## Consumer 配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `bootstrap.servers` | — | Kafka 集群地址，必填 |
| `group.id` | — | 消费者组 ID，必填 |
| `auto.offset.reset` | latest | 无初始 offset 时策略：`earliest`从头消费、`latest`从最新消费 |
| `enable.auto.commit` | true | 是否自动提交 offset |
| `auto.commit.interval.ms` | 5000 | 自动提交 offset 间隔（毫秒） |
| `max.poll.records` | 500 | 单次 poll 最大消息数 |
| `max.poll.interval.ms` | 300000 | 两次 poll 最大间隔，超时触发 rebalance |
| `session.timeout.ms` | 45000 | 消费者心跳超时时间 |
| `heartbeat.interval.ms` | 3000 | 心跳发送间隔，建议为 session.timeout.ms 的 1/3 |
| `fetch.min.bytes` | 1 | 每次 fetch 最小字节数 |
| `fetch.max.wait.ms` | 500 | fetch 等待时间，与 fetch.min.bytes 配合使用 |
| `key.deserializer` | — | Key 反序列化类，必填 |
| `value.deserializer` | — | Value 反序列化类，必填 |

---

## Topic 配置

| 参数 | 说明 |
|------|------|
| `retention.ms` | 消息保留时间（毫秒），覆盖 Broker 全局配置 |
| `retention.bytes` | 分区最大存储字节数 |
| `cleanup.policy` | 清理策略：`delete`（删除）或 `compact`（压缩去重） |
| `min.insync.replicas` | 最少同步副本数，与 `acks=all` 配合保证强一致性 |
| `replication.factor` | 副本数 |
| `num.partitions` | 分区数 |
| `max.message.bytes` | 单条消息最大字节数 |

---

## 关键调优建议

- **高吞吐量**：增大 `batch.size` + `linger.ms`，开启 `compression.type=lz4`
- **低延迟**：`linger.ms=0`，`acks=1`
- **强一致性**：`acks=all` + `min.insync.replicas=2` + `enable.idempotence=true`
- **防止消息丢失**：`acks=all`，关闭 `enable.auto.commit`，手动提交 offset

---

## 解决 Kafka CommitFailedException 问题


| 参数 | 默认值 | 建议值 | 说明 |
|---|---|---|---|
| `max.poll.interval.ms` | 300000 | 根据业务调整 | 两次 poll 最大间隔 |
| `max.poll.records` | 500 | 50～100 | 单次拉取最大条数 |
| `session.timeout.ms` | 45000 | 45000～60000 | 心跳超时 |
| `heartbeat.interval.ms` | 3000 | 3000 | 心跳发送间隔 |

heartbeat.interval.ms (10s)
        ↓ 每 10s 后台线程发一次心跳
session.timeout.ms (30s)
        ↓ 30s 内没收到心跳 → Broker 认为消费者死亡 → Rebalance
max.poll.interval.ms (30min)
        ↓ 30min 内没有调用 poll() → 消费者主动离组 → Rebalance

✅ 正确关系：heartbeat.interval.ms × 3 ≤ session.timeout.ms ≪ max.poll.interval.ms