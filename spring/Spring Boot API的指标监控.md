
```markdown
# 800毫秒到60毫秒：暴露并修复了我们慢速Spring Boot API的5个指标



你不知道的是：在真实负载下，200个并发用户，这个接口竟然需要14秒。数据库每个请求执行87次查询。连接池每天下午2点到4点都用尽。而所有能告诉你这些的指标都**没有被收集**。

快速API不是偶然产生的。知道自己什么时候慢也同样重要。

---

## 指标1：延迟——以及为什么你的平均值在欺骗你

大多数开发者在考虑响应时间时，会看平均值。**平均值几乎总是谎言。**

```java
// ❌ 错误 —— 单次请求的耗时无法告诉你真实负载下的情况
long start = System.currentTimeMillis();
OrderResponse response = orderService.processOrder(request);
long duration = System.currentTimeMillis() - start;
logger.info("Request took: {}ms", duration);
```

如果你100个请求中有95个在200毫秒内完成，但有5个耗时12秒，那么你的**平均**时间看起来是800毫秒——"可接受"。与此同时，每20个用户中就有1个盯着加载图标看12秒。他们不在乎平均值。

**高级开发者会衡量什么：**

```java
// ✅ 正确 —— 使用 Micrometer Timer 记录百分位数延迟
@Service
public class OrderService {

    private final Timer orderTimer;

    public OrderService(MeterRegistry registry) {
        this.orderTimer = Timer.builder("orders.latency")
            .description("Order processing latency")
            .publishPercentiles(0.5, 0.95, 0.99)  // p50, p95, p99
            .publishPercentileHistogram()
            .tag("service", "orders")
            .register(registry);
    }

    public OrderResponse processOrder(OrderRequest request) {
        return orderTimer.record(() -> doProcessOrder(request));
    }
}
```

现在Prometheus追踪了三个真正重要的数字：

| 指标 | 它告诉你的 |
|------|-----------|
| p50（中位数） | 一半的用户体验比这快 |
| p95 | 95%的用户体验 |
| p99 | 你最差的1%用户——通常是p50的5–10倍 |

**经验法则：** 看p99，在p95告警，庆祝p50。如果你的p95低于500ms，大多数用户都很满意。如果你的p99超过3秒，说明有人正在受苦——只是你还没听说。

> **规则：** 绝不要用平均值来衡量API性能。平均值掩盖了最差的用户体验。追踪p95和p99。这才是你真正的性能问题所在。

---

## 指标2：N+1查询问题——87次查询，其中1次就够了

这是Java API最常变慢、最隐蔽且**不被测量**的原因。

你拿了一个20个订单的列表。对于每个订单，Hibernate静默地发送一个独立的查询来加载客户。这意味着订单查询1次，客户查询20次——总共21次。加上订单项，你就有了87次查询。代码看起来很干净。结果看起来是正确的。数据库正在执行86次不必要的往返。

```java
// ❌ 错误 —— 看起来无害，静默导致N+1查询
public List<OrderResponse> getOrders() {
    List<Order> orders = orderRepository.findAll(); // 1次查询
    return orders.stream()
        .map(order -> new OrderResponse(
            order,
            order.getCustomer().getName(),  // N次查询 —— 每个订单一次
            order.getItems()                // N次查询
        ))
        .toList();
}
```

**高级开发者会做什么：**

```java
// ✅ 正确 —— 使用 JOIN FETCH 一次查询获取所有数据
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o " +
           "JOIN FETCH o.customer " +
           "JOIN FETCH o.items " +
           "WHERE o.status = :status")
    List<Order> findAllWithDetails(@Param("status") OrderStatus status);
}
```

**但你如何比用户先发现N+1问题？** 衡量每个请求的查询数量：

```yaml
# application.yml —— 在开发环境中记录慢查询和查询计数
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        session:
          events:
            log:
              LOG_QUERIES_SLOWER_THAN_MS: 100  # 记录超过100ms的查询
logging:
  level:
    org.hibernate.stat: DEBUG  # 显示每个Session的查询数量
```

在生产环境，利用Micrometer内置的数据源指标跟踪查询耗时：

```yaml
management:
  metrics:
    enable:
      jdbc: true       # 自动跟踪查询执行时间
      hikaricp: true   # 跟踪连接池使用情况
```

你的Grafana仪表盘现在会显示每个端点的平均查询执行时间。当单个端点查询时间从5ms跳到800ms时，就是你的N+1问题暴露的时候。

> **规则：** 如果端点接触到一个实体列表，**假设存在N+1查询问题**，直到你测量并证明没有。在开发中启用Hibernate统计，在生产中启用查询指标。

---

## 指标3：连接池耗尽——无人察觉的缓慢死亡

你的API早上9点不慢。下午2点还行。然后到下午3点它**完全没有响应**。然后下午5点又恢复正常。

这几乎总是数据库连接池的问题。默认情况下，HikariCP只给你**10个连接**。正常负载下够用。但在下午高峰时段，所有连接都被慢查询占满。新的请求排队等待空闲连接。用户盯着加载图标。最终，队列超时，你开始看到错误。

```yaml
# ❌ 错误 —— 默认池大小，对发生的情况一无所知
spring:
  datasource:
    url: jdbc:postgresql://localhost/mydb
    username: user
    password: pass
    # 没有连接池配置。没有指标。完全盲目。
```

**高级开发者会做什么：**

```yaml
# ✅ 正确 —— 显式池配置 + 暴露指标
spring:
  datasource:
    hikari:
      maximum-pool-size: 20         # 数据库最大连接数
      minimum-idle: 5               # 始终保持5个热连接
      connection-timeout: 3000      # 等待3秒后快速失败 —— 不要无限排队
      idle-timeout: 600000          # 10分钟后释放空闲连接
      max-lifetime: 1800000         # 每30分钟回收连接
      pool-name: MainPool

management:
  metrics:
    enable:
      hikaricp: true                # 向Prometheus暴露连接池指标
```

现在Prometheus自动收集这些指标：

| 指标 | 含义 |
|------|------|
| `hikaricp_connections_active` | 当前正在使用的连接数 |
| `hikaricp_connections_idle` | 可用的空闲连接 |
| `hikaricp_connections_pending` | **等待连接的请求数 ← 重点关注** |
| `hikaricp_connections_timeout` | **放弃等待的请求数 ← 在此告警** |

当 `hikaricp_connections_pending > 0` 持续2分钟时，这就是你的早期警告。等到超时发生，用户已经开始受苦了。

> **规则：** 如果你还不知道当前的活跃连接数，那你就是在瞎猜。在首次生产部署前启用HikariCP指标，而不是在下午3点第一次宕机之后。

---

## 指标4：线程池耗尽——当你的应用完全停止响应时

连接池会满。请求排队。每个排队请求占用一个线程。**线程池会满**。新请求甚至无法开始。应用看起来完全宕机——但实际上是线程池满了。

```yaml
# ❌ 错误 —— 默认Tomcat配置，无可见性
server:
  port: 8080
  # 没有线程池配置。不知道当前有多少活跃线程。
```

**高级开发者会做什么：**

```yaml
# ✅ 正确 —— 显式线程池配置 + Actuator指标
server:
  tomcat:
    threads:
      max: 200          # 最大工作线程数
      min-spare: 20     # 始终保持20个就绪线程
    connection-timeout: 5000  # 等待超过5秒的连接将被拒绝
    accept-count: 100   # 拒绝前最多排队100个请求

management:
  endpoints:
    web:
      exposure:
        include: metrics, threaddump
```

随时通过 `/actuator/threaddump` 检查线程健康——你会看到每个线程、它的状态以及在等待什么。

在Prometheus中关注：

```
tomcat_threads_busy_threads    — 正在处理请求的线程数
tomcat_threads_config_max      — 你配置的最大线程数
```

当 `busy_threads` 持续接近 `config_max` 时，你就处在边缘了。你要么增加池子，要么缩短响应时间——或者两者都做。

```java
// ✅ 额外技巧 —— 在宕机前检测线程饥饿
@Scheduled(fixedRate = 60000) // 每分钟检查一次
public void logThreadPoolHealth() {
    double utilization = (double) busyThreads / maxThreads;
    if (utilization > 0.8) {
        logger.warn("Thread pool at {}% capacity — risk of exhaustion",
            (int)(utilization * 100));
    }
}
```

> **规则：** 线程耗尽从外部看起来和完全宕机一模一样。在用户告诉你应用挂了之前，先了解你的线程池利用率。

---

## 指标5：堆内存趋势——三天后崩溃

内存泄漏不是突然发生的。它会在几个小时甚至几天内**积累**。每次部署，内存基线都比上一次稍微高一点。GC的锯齿模式变得越来越浅。最终，GC无法回收足够的空间，JVM随之崩溃。

等它崩溃时，根本原因已经埋在堆积了几个小时都没人处理的垃圾里。

```yaml
# ❌ 错误 —— 没有内存可见性，直到崩溃才知道
# 内存泄漏的第一个信号通常是凌晨2点的PagerDuty告警
# 而此时，没有堆转储已经来不及诊断了
```

**高级开发者会做什么：**

```yaml
# ✅ 正确 —— JVM内存指标通过Micrometer自动暴露
management:
  metrics:
    enable:
      jvm: true   # 堆内存、非堆内存、GC暂停时间 —— 全自动
```

在Grafana中，追踪这些JVM指标随时间的变化：

| 指标 | 含义 |
|------|------|
| `jvm_memory_used_bytes{area="heap"}` | 当前堆内存使用量 |
| `jvm_memory_max_bytes{area="heap"}` | 堆内存上限 |
| `jvm_gc_pause_seconds_sum` | GC暂停时间总和 |
| `jvm_gc_memory_promoted_bytes_total` | **晋升到老年代的对象大小 ← 重点关注** |

- **健康的应用**看起来像锯齿 —— 内存上升 → GC触发 → 内存下降
- **泄漏的应用**看起来像斜坡 —— 内存攀升 → GC触发 → 但每次底线都更高

**宕机前的告警：**

```yaml
# Grafana告警 —— 堆内存超过85%持续10分钟，意味着麻烦来了
- name: Heap memory warning
  condition: jvm_memory_used_bytes / jvm_memory_max_bytes > 0.85
  for: 10m
  severity: warning
  notify: on-call
```

> **规则：** 如果内存泄漏需要三天才导致应用崩溃，它在崩溃前三个小时就会出现在你的Grafana堆趋势中。**关注趋势**，而不仅仅是当前值。

---

## 五个指标如何串联

这些指标并不孤立。它们形成**级联反应**：

```
慢接口（高p99延迟）
     │
     ▼
因为N+1查询导致
     │
     ▼
N+1查询持有数据库连接更长时间
     │
     ▼
连接池满了 —— 请求排队 —— 线程被占用等待
     │
     ▼
线程池满了 —— 新请求被拒绝 —— 应用看起来挂了
     │
     ▼
内存随着排队对象积累而攀升 —— GC压力增大
     │
     ▼
OutOfMemoryError —— 凌晨3点PagerDuty —— 一身冷汗
```

**修复p99延迟，通常你就能修复所有后续问题。** 这就是为什么延迟总是**第一个**需要关注的指标。

---

