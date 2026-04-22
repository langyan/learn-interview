这是个很实用的问题，分析如下：

## `@Transactional(readOnly=true)` vs 不加注解

### 一、不加 `@Transactional`

```java
public User findById(Long id) {
    return userRepository.findById(id).orElse(null);
}
```

**行为：**
- 每次数据库操作都是**独立的、自动提交**的连接（auto-commit）
- 没有事务上下文，连接用完即还给连接池
- 如果方法内有多次查询，每次查询获取**不同的连接**

**适用场景：**
- 单次简单查询，无需一致性保证
- 对性能极度敏感、事务开销不可接受的场景（极少见）

---

### 二、`@Transactional(readOnly=true)`

```java
@Transactional(readOnly=true)
public List<Order> findOrdersWithDetails(Long userId) {
    User user = userRepository.findById(userId);
    List<Order> orders = orderRepository.findByUser(user);
    // 两次查询在同一事务/连接内，数据一致
    return orders;
}
```

**行为：**
- 开启一个**只读事务**，整个方法共享同一个连接
- Spring 会将该 hint 传递给底层驱动/数据库
- Hibernate 会**关闭脏检查（dirty checking）**，不追踪实体状态变更
- 部分数据库（如 MySQL）会做读优化，比如跳过 undo log 构建

**核心优势：**

| 优势 | 说明 |
|---|---|
| **一致性读** | 多次查询在同一快照内，避免不可重复读 |
| **关闭脏检查** | Hibernate 不再对每个实体做 diff，内存和 CPU 开销更低 |
| **连接复用** | 方法内多次查询共用同一连接，减少连接池压力 |
| **数据库优化** | MySQL InnoDB 在只读事务中可走优化路径 |
| **语义表达** | 明确告诉代码阅读者"此方法不写数据" |

---

### 三、关键对比：脏检查的影响

这是 `readOnly=true` 最重要的性能收益之一：

```java
// 没有 readOnly=true —— Hibernate 会在 Session 关闭时
// 对所有加载的实体做 snapshot 对比（即使你没改任何东西）
@Transactional
public List<Product> getProducts() {
    return productRepository.findAll(); // 加载1000条，每条都被追踪
}

// readOnly=true —— Hibernate 跳过快照，不追踪实体
@Transactional(readOnly=true)
public List<Product> getProducts() {
    return productRepository.findAll(); // 加载1000条，零追踪开销
}
```

---

### 四、结论：怎么选？

```
单次查询、无一致性要求       →  可以不加（或加 readOnly=true 也无害）
方法内多次查询需要数据一致   →  必须加 @Transactional(readOnly=true)
返回大量实体（List）         →  强烈推荐 readOnly=true（关脏检查省内存）
Service 层的查询方法（通用） →  推荐统一加 readOnly=true，是最佳实践
```

**最佳实践模式：**

```java
@Service
@Transactional(readOnly=true)   // 类级别默认只读
public class OrderService {

    public Order findById(Long id) { ... }        // 继承只读

    @Transactional                                 // 覆盖为读写
    public Order createOrder(OrderDTO dto) { ... }

    @Transactional                                 // 覆盖为读写
    public void updateOrder(Long id, ...) { ... }
}
```

类级别加 `readOnly=true`，写操作方法单独用 `@Transactional` 覆盖，是 Spring 项目中最常见也最推荐的做法。