

# 10个正在默默扼杀你的生产应用的Spring Boot错误

---

## 错误一：从未触碰HikariCP池大小

Spring Boot默认有10个连接池。在真实流量下，线程排队等待空闲连接，你的响应时间无缘无故地增加。

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 30
      minimum-idle: 10
      connection-timeout: 20000
```

**原则：** 根据你的数据库容量来计算规模，而不是框架对你容量的假设。

---

## 错误二：忽略 readOnly = true

对每个服务方法（包括纯读）应用 `@Transactional`，每次都会打开一次完整的写入事务。在处理每秒数千次读取的服务中，锁定开销会迅速累积。

```java
// ❌ 为纯粹的读操作打开完整事务
@Transactional
public User findById(Long id) {
    return repo.findById(id).orElseThrow();
}

// ✅ 正确做法
@Transactional(readOnly = true)
public User findById(Long id) {
    return repo.findById(id).orElseThrow();
}
```

一个关键词。在生产负载下有可测量的差异。

---

## 错误三：在循环内打印日志

每处理一条记录写一条日志，听起来无害，直到你的批处理每15分钟处理5万条记录。你现在制造了一场自我制造的灾难，降低了性能并膨胀了你的可观测性账单。

```java
// ❌ 错误
for (Order o : orders) {
    process(o);
    log.info("Processed order: {}", o.getId()); // 删掉这行
}

// ✅ 正确
log.info("Batch done. Count: {}", orders.size());
```

**原则：** 记录结果，而不是每一步。

---

## 错误四：JPA代码中的N+1查询问题

你查询了100个用户。JPA会再发送100次查询以加载每个用户的订单。我曾经评估过的一个服务，为了一个API响应，竟然要调用847次数据库。该接口用时6.3秒。一次 `JOIN FETCH` 将时间降至180毫秒。

```java
// ❌ 1 + N 次查询静默发生
List<User> users = userRepo.findAll();
users.forEach(u -> u.getOrders().size());

// ✅ 修复方法
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

在开发中启用 `spring.jpa.show-sql=true`，真正查看被执行的SQL语句。

---

## 错误五：外部HTTP调用无超时

你的服务调用外部API。那个API会变慢。你的线程在等待。你的连接池会满。你的整个应用程序停止响应——不是因为你的代码，而是因为你信任了一个没有时间限制的外部服务。

```java
factory.setConnectTimeout(Duration.ofSeconds(3));
factory.setReadTimeout(Duration.ofSeconds(5));
```

**原则：** 每个外拨电话都需要一个明确的超时上限。没有它，一个缓慢的依赖就能让你完全停机。

---

## 错误六：无限制地暴露执行器端点

你添加了执行器用于健康检查，这很合理。但 `/actuator/env` 可以向任何能通过内部服务访问的人暴露包括数据库凭证在内的环境变量。这在真实公司中引发了真正的安全事件。

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info   # 只暴露必要的端点
```

**原则：** 每次生产发布前都要审核执行器端点暴露配置。

---

## 错误七：在HTTP会话中存储大型对象

有人在会话中存储了一个完整的用户配置文件对象。随着产品上线，这个对象不断增长。

会话会在节点间复制。内存压力悄悄积累，直到Pod在负载下被杀死，没人能立刻明白原因。

**原则：** 在会话中只存储一个ID。其他东西需要时再去数据库获取。

---

## 错误八：相信@Async能修复阻塞代码

用 `@Async` 注解方法，然后在里面做阻塞式JDBC调用，并不会让你的应用更快。

它把瓶颈转移到了一个你不易察觉的线程池，使问题更难诊断，而不是更简单。

**原则：** 异步是用于真正非阻塞的工作。它不会改变阻塞的本质——它只是隐藏了它。

---

## 错误九：外部调用周围没有断路器

```
Your Service
     |
     v
[External API slows down]
     |
     v
Threads pile up waiting
     |
     v
Thread pool exhausts
     |
     v
Your entire service becomes unresponsive
```

一个依赖出问题，不应该让你的整个应用都被拖垮。

```java
@CircuitBreaker(name = "payments", fallbackMethod = "fallback")
public String charge(Order o) {
    return paymentClient.charge(o);
}

public String fallback(Order o, Exception e) {
    return "PENDING";
}
```

**原则：** 保护你的应用所跨越的每一条外部边界。

---

## 错误十：跳过优雅关机

部署发生了。你的应用收到了终止信号。当前请求会在执行中途被切断。数据库事务保持半提交状态。用户在本应是干净、隐形的部署过程中会看到错误。

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

两条配置。从那时起，每次部署都没有丢弃请求。

---
