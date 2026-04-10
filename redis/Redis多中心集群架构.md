以下是 Redis 多数据中心（Multi-datacenter）架构设计，核心思路是：**每个数据中心部署独立 Redis Cluster，中心间通过异步复制或消息同步实现数据一致性**。---

## 核心设计原则

每个数据中心是一个**完整独立的 Redis Cluster**，对外提供完整服务。中心间不共享节点，通过专门的**跨中心同步层**传播数据变更。

![Redis 多数据中心（Multi-datacenter）架构设计](/assets/img/redis_multi_datacenter_cluster.svg "Redis 多数据中心（Multi-datacenter）架构设计")

---

## 每个中心的 Cluster 配置

最少 3 主 3 从，生产推荐 6 主 6 从：

```bash
# 创建单个数据中心的 cluster（DC1 示例）
redis-cli --cluster create \
  10.1.0.1:7001 10.1.0.2:7002 10.1.0.3:7003 \
  10.1.0.4:7004 10.1.0.5:7005 10.1.0.6:7006 \
  --cluster-replicas 1
```

`redis.conf` 关键配置：

```conf
cluster-enabled yes
cluster-config-file nodes-dc1.conf
cluster-node-timeout 5000
cluster-require-full-coverage no   # 允许部分 slot 不可用时继续服务
cluster-announce-ip 10.1.0.1       # 公告 DC 内部 IP，防止跨中心 gossip 混淆
cluster-announce-port 7001
```

> `cluster-require-full-coverage no` 是多中心场景的关键：单 DC 故障时其余中心不受影响。

---

## 跨中心同步方案（三选一）

### 方案 A：RedisGears（开源，推荐用于简单写扩散）

```python
# 在每个 DC 部署 write-behind 函数，将写操作广播到其他中心
import redis

def sync_to_remote(event):
    remote = redis.RedisCluster(
        host="dc2-proxy.internal", port=6379
    )
    if event['type'] == 'hset':
        remote.hset(event['key'], mapping=event['value'])
    elif event['type'] == 'set':
        remote.set(event['key'], event['value'])

# 注册 KeySpace 事件触发
GB('KeysReader').foreach(sync_to_remote).register('*')
```

### 方案 B：Kafka 中间层（推荐用于高可靠、审计场景）

```
写入 DC1 Cluster
    │
    ▼
DC1 消费者捕获 keyspace 事件
    │
    ▼
发布到 Kafka topic: redis-sync-global
    │
    ├──▶ DC2 消费者写入 DC2 Cluster
    └──▶ DC3 消费者写入 DC3 Cluster
```

```python
from kafka import KafkaProducer, KafkaConsumer
import redis, json

# 生产者：DC1 监听并转发
producer = KafkaProducer(bootstrap_servers='kafka:9092',
                         value_serializer=lambda v: json.dumps(v).encode())

def forward_write(key, value):
    producer.send('redis-sync', {'key': key, 'value': value, 'source': 'dc1'})

# 消费者：DC2/DC3 接收并写入
consumer = KafkaConsumer('redis-sync', bootstrap_servers='kafka:9092')
cluster = redis.RedisCluster(host='localhost', port=7001)

for msg in consumer:
    data = json.loads(msg.value)
    if data['source'] != 'dc2':  # 避免回写自己
        cluster.set(data['key'], data['value'])
```

### 方案 C：Redis Enterprise CRDT（商业，零冲突）

Redis Enterprise 原生支持 Active-Active 地理分布，底层使用 **CRDT（无冲突复制数据类型）**，配置极简：

```bash
# 创建 Active-Active 数据库（RE 管理界面或 REST API）
curl -X POST https://re-dc1:9443/v1/bdbs \
  -d '{"name":"mydb","type":"redis","replication":true,
       "crdt":true,"crdt_sync_dist":true,
       "crdt_causal_consistency":true}'
```

---

## 客户端接入策略

```python
import redis

# 就近接入：客户端通过环境变量或 DNS 选择最近 DC
cluster = redis.RedisCluster(
    host="redis-local.dc1.internal",  # VIP / DNS 解析到本 DC
    port=6379,
    retry_on_timeout=True,
    # 读写分离：读请求分发到 replica
    read_from_replicas=True,
)

# 写操作统一走主节点（防冲突）
cluster.set("user:1001", "alice")

# 读操作自动路由到就近 replica
val = cluster.get("user:1001")
```

---

## 关键注意事项

| 问题 | 说明 |
|---|---|
| **写冲突** | 同一 key 多中心同时写时，最终一致性策略需明确（Last-Write-Wins 或 CRDT） |
| **时钟同步** | 各 DC 服务器需用 NTP/PTP 对齐，否则 LWW 策略不可靠 |
| **网络分区** | 跨中心专线断开时，各 DC 独立服务本地流量，恢复后做增量同步 |
| **slot 映射** | 各 DC 的 Cluster 各自维护独立 slot 映射，跨中心不共享 gossip |
| **监控** | 每个 DC 独立部署 Redis Sentinel 或 Prometheus + redis_exporter 监控 |