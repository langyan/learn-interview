以下是将您提供的文章内容整理为 Markdown 格式的结果：


# HikariCP 如何在我们的 Spring Boot 微服务中导致 CPU 占用率高达 98%




## 1. CPU 98%


结账服务——我们收入管道的核心——因 504 Gateway 超时错误而出现故障。客户无法完成付款，订单卡住了，支援通道信息爆炸。

我打开 Grafana，立刻看到灾难正在实时展开：

- 所有 pod 的 CPU 使用率都达到了 **98%**
- 请求延迟激增
- 数据库连接量失控地攀升
- 许多会话在事务中处于空闲状态
- 待处理请求每秒都在增加

数据库看起来不像是生产系统，更像是演唱会后体育场外的交通，什么都没动。

我登录了一个 pod，捕获了一个线程转储。同样的栈迹遍布各处：

```java
"http-nio-8080-exec-74" #74 daemon prio=5 os_prio=0 tid=0x00007f9e3c0d8800 nid=0x6b4 waiting on condition
   java.lang.Thread.State: WAITING (parking)
 at sun.misc.Unsafe.park(Native Method)
 - parking to wait for  <0x00000006c2e1f1b8> (a java.util.concurrent.Semaphore$NonfairSync)
 at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
 at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
 at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:997)
 at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1304)
 at java.util.concurrent.Semaphore.acquire(Semaphore.java:312)
 at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:182)
```

超过 **180** 个线程被阻塞，等待来自 HikariCP 的数据库连接。

连接池已经到了极限。每个新请求都会生成另一个等待线程。CPU 使用率不断上升，因为 JVM 忙于在阻塞线程间切换上下文，而不是做实际工作。

那一刻，我意识到一件重要的事：

> 大多数开发者通过 Spring Boot 每天都使用 HikariCP，但真正理解它内部工作原理的人很少——直到生产环境迫使他们去理解。

那晚让我深入 HikariCP 的内部：

- 连接借用的真实运作方式
- `ConcurrentBag` 的实际功能
- 为什么会出现线程阻塞
- 池耗尽如何破坏吞吐量
- 为什么糟糕的事务设计会扼杀性能
- 以及微小的配置错误如何可能毁掉整个系统

这篇文章是我希望在那件事发生前能看到的完整分析。

---

## 2. 什么是 HikariCP？

连接池是一个可重复使用的数据库连接缓存，避免了新建 TCP 连接、执行 TLS 握手和对每个请求进行认证的巨大成本。

**没有池子，会发生以下情况：**

1. API 收到请求 → 打开数据库连接（50-100 毫秒）
2. 运行查询（2 毫秒）
3. 关闭连接（拆除）

对于 1000 个并发请求，你光是连接和断开就花费 50 到 100 秒。

**使用池时**：你“借用”一个已经建立的连接，使用后归还 —— 几乎是免费的。

**HikariCP**（日语意为“光”）是 Spring Boot 2.x 及以后版本的默认连接池。它以以下特点闻名：

- 极其快速（微优化字节码，零开销代理）
- 轻量级（≈130KB jar 包）
- 可靠且经得起生产环境考验

Spring Boot 选择它，是因为它在吞吐量和延迟基准测试中持续优于 C3P0、Apache DBCP2、Tomcat Pool 和 Vibur。

> HikariCP 通过自定义的字节码级代理生成（Javassist）、精心调整的锁定策略以及 `ConcurrentBag` 实现了这一点。

---

## 4. HikariCP 内部运作原理

### 4.1 核心架构

每个 HikariCP 池都包含：

- **ConcurrentBag** —— 一个自定义的无锁集合，包含所有 `PoolEntry` 对象。
- **PoolEntry** —— 包裹实际 `java.sql.Connection` 的对象。状态包括：`IN_USE`、`NOT_IN_USE`、`RESERVED`、`REMOVED`。
- **HouseKeeper** —— 一个每 30 秒运行一次的后台线程，用于驱逐空闲连接、强制执行 `maxLifetime` 并执行 keepalive 查询。

### 4.2 连接生命周期

1. **池启动** —— 创建 `minimumIdle` 个连接（如果 `minimumIdle` 为 0 则是懒加载）。
2. **借用连接** —— 调用者调用 `getConnection()`：
   - HikariCP 首先检查**线程本地切换列表**（快速路径）。如果同一线程最近使用过连接，连接会立即返回，没有任何锁定。
   - 如果没有，它会扫描 `ConcurrentBag` 中的 `NOT_IN_USE` 条目。如果找到，标记为 `IN_USE`。
   - 如果池已满，调用者会在信号量上阻塞，直到连接超时。
3. **返回连接** —— 代理调用 `close()`：
   - 状态被设回 `NOT_IN_USE`，信号量释放等待线程。
4. **空闲连接** —— `HouseKeeper` 移除闲置时间超过 `idleTimeout` 的连接，降至 `minimumIdle`。
5. **最大生命周期** —— 任何比 `maxLifetime` 更早的连接都会被透明地替换。

**核心洞察：** 快速路径（线程本地缓存）是 HikariCP 在温热 JVM 场景下借用连接耗时 < 50 纳秒的原因。当同一线程反复获取/释放连接时，它避免了任何同步块或 CAS 循环。

### 4.3 ConcurrentBag —— 秘密武器

`ConcurrentBag` 是一个仅附加的列表，每个线程都有自己的 `ThreadLocal` 队列。

- 线程借用连接时，不会从单个共享队列中窃取。
- 它首先检查自己的 `ThreadLocal` 列表（快速路径）。
- 然后使用**窃取**方式无锁地扫描其他线程的 `ThreadLocal` 队列，这种方式支持缓存。

这种设计消除了大多数连接池面临的核心瓶颈。

---

## 5. 内部工作深度探索

### 5.1 Spring Boot 中的请求流程

1. HTTP 请求到达 `@RestController`。
2. 控制器调用一个 `@Service` 方法，该方法带有 `@Transactional` 或使用 `JdbcTemplate`。
3. Spring 的事务管理器调用 `DataSource.getConnection()`。
4. HikariCP 的 `getConnection()` 触发了上述借用逻辑。
5. 应用程序执行 SQL，然后在事务提交（或语句关闭）期间，代理将连接归还给池。

### 5.2 排队机制与池耗尽

当所有 `maximumPoolSize` 连接都被使用且有新请求到达时：

1. 请求线程调用 `Semaphore.tryAcquire(timeout)`。
2. 它被停放（park）在 AQS 队列中（线程状态：`WAITING`）。
3. 超时触发 `SQLTransientConnectionException`，消息类似：
   ```
   HikariPool-1 - Connection is not available, request timed out after 30000ms.
   ```

在高峰流量时，故障会滚雪球般蔓延：

- 线程堆积，每个线程占用 ~1 MB 栈空间
- 上下文切换让 CPU 使用率暴涨
- 等待请求对象增加 GC 压力

这正是我们凌晨两点灾难的根源。

---

## 6. Spring Boot 配置示例

### 小项目（开发笔记本）

```properties
spring.datasource.hikari.maximumPoolSize=5
spring.datasource.hikari.connectionTimeout=5000
spring.datasource.hikari.idleTimeout=300000
```

### 中等流量的应用

```yaml
spring:
  datasource:
    hikari:
      maximumPoolSize: 10
      minimumIdle: 10
      connectionTimeout: 3000
      idleTimeout: 600000
      maxLifetime: 1800000
      leakDetectionThreshold: 15000
```

### Kubernetes 中的高流量微服务

```properties
spring.datasource.hikari.maximumPoolSize=20
spring.datasource.hikari.connectionTimeout=2000
spring.datasource.hikari.maxLifetime=1200000
spring.datasource.hikari.keepaliveTime=60000
spring.datasource.hikari.leakDetectionThreshold=10000
```

> 结合 `kubernetes resources.requests.cpu` 和 `limits.cpu` 来保证 JDBC 驱动 I/O 有足够的 CPU。

---

## 7. 真实生产事件：结账服务死掉的那晚

### 背景

电子商务结账服务，Spring Boot 2.7，RDS 上的 PostgreSQL。  
Pod：4 个，每个 2 个 CPU 核心，堆 1GB。

### 症状

- 在一次闪购期间，响应时间从 50ms 上升到了 30s
- 所有 Pod 的 CPU 都达到 95%+
- `hikaricp_connections_active` 指标显示 20（最大池容量）持续满额
- `hikaricp_connections_pending` 达到 200
- 数据库 CPU 占了 80%

### 调查经过

- **Grafana**：4 个 Pod 共 80 个连接全部激活，等待队列增长
- **线程转储**：`HikariPool.getConnection()` 上有 180 个线程阻塞
- **PostgreSQL**：
  ```sql
  select count(*) from pg_stat_activity
  ```
  显示事务会话中有 80 个空闲连接
- **代码审查**：一位开发者引入了一个调用外部支付 API 的方法，并加了 `@Transactional`。该事务在等待慢速 HTTP 调用（有时需要 20 秒）时保持数据库连接开启 —— 这是**设计中的连接泄漏**。

### 根本原因

长期持有的事务阻塞了连接池。每个请求线程都会抓取一个连接并保持空闲，等待支付服务。池子在几秒钟内就耗尽了。

### 修复

1. 移除了同步 API 调用中的 `@Transactional`，只包裹了数据库写入部分。
2. 设置 `spring.datasource.hikari.leakDetectionThreshold=5000` 以捕捉未来的泄漏。
3. 调低 `connectionTimeout=2000`。
4. 增加了关于支付 API 的断路器。
5. 由于遗留依赖，临时将池大小增加到 30，随后计划拆分服务。

部署完成后，CPU 降到 12%，待处理连接数降到 0，结账恢复正常。

---

## 8. HikariCP CPU 问题

HikariCP 高 CPU 并不常见，但发生时通常是以下几种情况之一：

1. **连接池耗尽** → 数百个停放线程 → 上下文切换风暴
2. **过于激进的 keepalive/验证查询** —— 如果 `keepaliveTime` 非常低（例如 5 秒），数据库很慢，`HouseKeeper` 会一直 ping，从池中窃取 CPU
3. **`maxLifetime` 过短且不断创建/销毁循环**，导致数据库连接设置过于饱和，TCP 需要内核 CPU
4. **JDBC 驱动存在漏洞** —— 驱动本身在高并发情况下可能会锁死
5. **高流失率下数千个短生命周期连接代理的 GC 压力**
6. **过度日志** —— 如果不加限制，HikariCP 可能会刷屏“连接不可用”等警告，日志记录成本高昂

### Linux CPU 故障排除命令

```bash
top -H -p <java-pid>   # 查找最繁忙的线程
# 将线程 ID 转换为十六进制，然后获取堆栈
jstack <pid> > threaddump.txt
```

---

## 9. 连接泄漏问题

**连接泄漏**是指你获得了一个连接但从未关闭它（或归还给池）。随着时间的推移，池子会逐渐干涸。

### 常见泄漏原因

```java
// 错误示例 1：连接从未关闭
Connection conn = dataSource.getConnection();
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("...");
// 没有 close()

// 错误示例 2：流未关闭
jdbcTemplate.queryForStream(sql, ...)
// Stream 未关闭，一直持有连接

// 错误示例 3：@Transactional 放在长时间的非数据库操作上
@Transactional
public void processPayment() {
    orderRepo.save(order);          // 2ms
    paymentGateway.charge(card);    // 20 秒！
}
```

### 如何拯救你：`leakDetectionThreshold`

```properties
spring.datasource.hikari.leakDetectionThreshold=10000
```

任何超过 10 秒的连接会输出日志：

```
[HikariPool-1 housekeeper] WARN  c.z.h.p.ProxyLeakTask - Connection leak detection triggered for conn...
```

现在你已经确切知道是哪种方法在持有连接了。

**最佳实践**：始终使用 try-with-resources 或 Spring 的 `JdbcTemplate` / `TransactionTemplate`，它们会自动关闭连接。

---

## 10. 初学者常见错误

- ❌ 设置巨大的 `maximumPoolSize` —— 认为“越大越好”，这会让数据库过载，浪费内存
- ❌ 没有关闭连接（前面已提到）
- ❌ 滥用 `@Transactional` —— 在整个 REST 控制器方法上使用
- ❌ 在同一连接池上运行长时间查询，导致快速查询被饿死（如需区分，可以为 OLAP 和 OLTP 分开池）
- ❌ 忽略索引 —— 慢查询会保持连接更久
- ❌ 在不了解 `maxLifetime` 与数据库超时关系的情况下，复制粘贴 Stack Overflow 的配置
- ❌ 高流量服务使用默认配置 —— 30 秒连接超时是导致停产的配方
- ❌ 没有监控 —— 盲目飞行

---

## 13. 监控 HikariCP

Spring Boot Actuator + Micrometer 暴露所有关键指标。

在 `application.properties` 中启用：

```properties
management.endpoints.web.exposure.include=health,metrics,prometheus
```

### Prometheus 告警规则示例

```yaml
- alert: HikariPoolExhausted
  expr: rate(hikaricp_connections_timeout_total[1m]) > 0
  for: 1m
  annotations:
    summary: "HikariCP pool is timing out"
```

**Grafana 仪表盘**：建议展示 `active` + `idle` + `pending` 的堆叠图，并以 `maximumPoolSize` 作为阈值线。

---

## 14. 线程转储分析

当出现问题时，线程转储能揭示真相。

### 池耗尽的症状

```java
"http-nio-8080-exec-97" #97 daemon prio=5 os_prio=0 tid=0x... nid=0x9a waiting on condition
   java.lang.Thread.State: WAITING (parking)
 at sun.misc.Unsafe.park(Native Method)
 - parking to wait for  <0x00000006c2e1f1b8> (a java.util.concurrent.Semaphore$NonfairSync)
 at java.util.concurrent.locks.LockSupport.park(...)
 at java.util.concurrent.Semaphore.acquire(...)
 at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:182)
```

成百上千个这样的栈迹意味着你已经没有连接了。

### 连接泄漏的表现

```java
"http-nio-8080-exec-45" #45 ... TIMED_WAITING (parking)
   ...
   at com.zaxxer.hikari.pool.ProxyConnection.close(ProxyConnection.java:...)
   - locked <0x000000...> (a com.zaxxer.hikari.pool.ProxyConnection)
```

如果线程卡在关闭连接中，JDBC 驱动可能正在等待 TCP 套接字。

### 收集线程转储的命令

```bash
jstack <pid> > dump.txt
# 或者连续采集 5 次，间隔 2 秒：
for i in {1..5}; do jstack <pid> > dump_$i.txt; sleep 2; done
```

> 在堆栈跟踪中寻找卡在 `HikariPool.getConnection()` 上的线程，看看它们在等待什么。

