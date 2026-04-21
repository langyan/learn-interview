## 在 Redis 中高效查找特定前缀的大量 Key

**核心原则：绝对不能用 `KEYS` 命令**，它会阻塞整个 Redis 服务。

---

### 方案一：`SCAN` 命令（生产环境首选）

```bash
SCAN cursor MATCH prefix:* COUNT 100
```

`SCAN` 是渐进式遍历，不会阻塞服务器。

**Java 示例（Jedis）：**

```java
Set<String> matchedKeys = new HashSet<>();
String cursor = "0";

do {
    ScanParams params = new ScanParams()
        .match("flight:*")
        .count(200);  // 每次扫描的槽数量，不是精确返回数
    
    ScanResult<String> result = jedis.scan(cursor, params);
    matchedKeys.addAll(result.getResult());
    cursor = result.getCursor();
    
} while (!"0".equals(cursor));
```

**Spring Data Redis 示例：**

```java
Set<String> keys = new HashSet<>();
ScanOptions options = ScanOptions.scanOptions()
    .match("flight:*")
    .count(200)
    .build();

try (Cursor<byte[]> cursor = redisTemplate.executeWithStickyConnection(
        conn -> conn.scan(options))) {
    while (cursor.hasNext()) {
        keys.add(new String(cursor.next()));
    }
}
```

---

### 方案二：`SCAN` + Pipeline 批量处理（性能优化）

扫到 key 后，不要逐个操作，攒成批次再用 pipeline 处理：

```java
List<String> batch = new ArrayList<>();

do {
    ScanResult<String> result = jedis.scan(cursor, params);
    batch.addAll(result.getResult());
    
    // 攒够 500 个再批量处理
    if (batch.size() >= 500) {
        processBatch(pipeline, batch);
        batch.clear();
    }
    cursor = result.getCursor();
} while (!"0".equals(cursor));

// 处理剩余
if (!batch.isEmpty()) processBatch(pipeline, batch);
```

---

### 方案三：Redis Cluster 环境（多节点）

集群模式下 `SCAN` 只扫当前节点，需要对每个节点单独扫：

```java
// Jedis Cluster 需要遍历所有主节点
JedisCluster cluster = ...;
Map<String, JedisPool> clusterNodes = cluster.getClusterNodes();

for (Map.Entry<String, JedisPool> entry : clusterNodes.entrySet()) {
    try (Jedis jedis = entry.getValue().getResource()) {
        // 只处理主节点
        if (jedis.info("replication").contains("role:master")) {
            scanNode(jedis, "flight:*", allKeys);
        }
    }
}
```

---

### 方案四：用专属数据结构维护索引（根本解法）

如果这种查询很频繁，应该**维护一个 Set 作为索引**：

```java
// 写入时同时维护索引
pipeline.set("flight:MU5735:status", value);
pipeline.sadd("idx:flight:keys", "flight:MU5735:status");  // 维护索引
pipeline.sync();

// 查询时直接
Set<String> keys = jedis.smembers("idx:flight:keys");
// 或分批: SSCAN idx:flight:keys
```

---

### 各方案对比

| 方案 | 适用场景 | 性能 | 复杂度 |
|------|---------|------|--------|
| `SCAN` | 通用，一次性查询 | 中（遍历全量） | 低 |
| `SCAN` + Pipeline | 查到后需批量操作 | 高 | 中 |
| 多节点 SCAN | Redis Cluster | 中 | 高 |
| Set 索引 | 高频查询 | 极高 | 中 |

---

### 关键参数调优

`COUNT` 参数不是返回数量的上限，而是**每次扫描的哈希槽数量**。对于一亿个 key：
- 值太小（如 10）：网络往返次数太多，总耗时长
- 值太大（如 10000）：单次扫描时间长，有轻微阻塞风险
- **推荐 200~500**，在你的 `c3i-flight` 场景下可以根据 Redis 负载动态调整