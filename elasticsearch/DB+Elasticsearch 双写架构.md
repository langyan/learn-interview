我来设计一个完整的 DB + Elasticsearch 双写架构，并配套 Java 实现。先看整体架构图，然后逐层展开。架构分为 **写路径**（DB 事务 + MQ 解耦）和 **查询路径**（ES 承担全文检索），通过 Sync Consumer 异步同步，CDC 作为备选方案。下面是分层的 Java 实现代码。

![双写架构](../assets/img/db_es_architecture_overview.svg "双写架构)

---

## 1. 核心依赖 (`pom.xml`)

```xml
<dependencies>
  <!-- Spring Boot -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
  </dependency>
  <!-- Kafka -->
  <dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
  </dependency>
  <!-- MySQL -->
  <dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
  </dependency>
</dependencies>
```

---

## 2. 实体 & ES 文档

```java
// DB 实体
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String description;
    private BigDecimal price;
    @Enumerated(EnumType.STRING)
    private ProductStatus status;
    private LocalDateTime updatedAt;
}

// ES 文档（字段可与 DB 不一致，专为搜索设计）
@Document(indexName = "products")
@Setting(settingPath = "es/product-settings.json")
public class ProductDocument {
    @Id
    private String id;

    @MultiField(mainField = @Field(type = FieldType.Text, analyzer = "ik_max_word"),
                otherFields = @InnerField(suffix = "keyword", type = FieldType.Keyword))
    private String name;

    @Field(type = FieldType.Text, analyzer = "ik_smart")
    private String description;

    @Field(type = FieldType.Double)
    private BigDecimal price;

    @Field(type = FieldType.Keyword)
    private String status;

    @Field(type = FieldType.Date, format = DateFormat.date_time)
    private LocalDateTime updatedAt;
}
```

---

## 3. 写服务（DB + 发布领域事件）

```java
@Service
@RequiredArgsConstructor
public class ProductWriteService {

    private final ProductRepository productRepo;
    private final ApplicationEventPublisher eventPublisher;

    /**
     * 先写 DB（事务保障），事务提交后再发布事件
     * 使用 @TransactionalEventListener 确保 DB 成功后才发 MQ
     */
    @Transactional
    public Product createProduct(CreateProductRequest req) {
        Product product = mapper.toEntity(req);
        product.setUpdatedAt(LocalDateTime.now());
        Product saved = productRepo.save(product);
        // 事务内发布，TransactionalEventListener 在 AFTER_COMMIT 阶段处理
        eventPublisher.publishEvent(new ProductChangedEvent(saved, ChangeType.CREATE));
        return saved;
    }

    @Transactional
    public Product updateProduct(Long id, UpdateProductRequest req) {
        Product product = productRepo.findById(id).orElseThrow();
        mapper.update(product, req);
        product.setUpdatedAt(LocalDateTime.now());
        productRepo.save(product);
        eventPublisher.publishEvent(new ProductChangedEvent(product, ChangeType.UPDATE));
        return product;
    }

    @Transactional
    public void deleteProduct(Long id) {
        productRepo.deleteById(id);
        eventPublisher.publishEvent(new ProductChangedEvent(
            Product.ofId(id), ChangeType.DELETE));
    }
}
```

---

## 4. 事件监听 → Kafka 发送

```java
@Component
@RequiredArgsConstructor
public class ProductEventListener {

    private final KafkaTemplate<String, ProductChangedEvent> kafkaTemplate;

    // AFTER_COMMIT 确保 DB 事务已提交，避免 ES 看到未提交数据
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onProductChanged(ProductChangedEvent event) {
        kafkaTemplate.send("product.changes", 
                           String.valueOf(event.getProductId()), 
                           event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Kafka send failed, productId={}", event.getProductId(), ex);
                    // 可写入本地 outbox 表做补偿
                }
            });
    }
}
```

---

## 5. Sync Consumer（MQ → ES）

```java
@Component
@RequiredArgsConstructor
public class ProductSyncConsumer {

    private final ElasticsearchOperations esOps;
    private final ProductRepository productRepo;

    @KafkaListener(topics = "product.changes",
                   groupId = "es-sync",
                   containerFactory = "kafkaListenerContainerFactory")
    public void consume(ProductChangedEvent event,
                        Acknowledgment ack) {
        try {
            switch (event.getChangeType()) {
                case CREATE, UPDATE -> {
                    // 从 DB 重新查最新数据（避免消息乱序问题）
                    productRepo.findById(event.getProductId()).ifPresent(p -> {
                        esOps.save(toDocument(p));
                    });
                }
                case DELETE -> {
                    esOps.delete(String.valueOf(event.getProductId()), 
                                 ProductDocument.class);
                }
            }
            ack.acknowledge();
        } catch (Exception ex) {
            log.error("ES sync failed, id={}", event.getProductId(), ex);
            throw ex; // 触发重试策略
        }
    }
}
```

---

## 6. 查询服务（ES）

```java
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations esOps;

    public SearchHits<ProductDocument> search(ProductSearchRequest req) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(buildQuery(req))
            .withAggregation("price_range", buildPriceAgg())
            .withSort(SortOptions.of(s -> s.field(
                f -> f.field("_score").order(SortOrder.Desc))))
            .withPageable(PageRequest.of(req.getPage(), req.getSize()))
            .build();

        return esOps.search(query, ProductDocument.class);
    }

    private Query buildQuery(ProductSearchRequest req) {
        BoolQuery.Builder bool = new BoolQuery.Builder();

        if (StringUtils.hasText(req.getKeyword())) {
            bool.must(MultiMatchQuery.of(m -> m
                .query(req.getKeyword())
                .fields("name^3", "description")  // name 权重更高
                .type(TextQueryType.BestFields)
                .fuzziness("AUTO")
            )._toQuery());
        }

        if (req.getMinPrice() != null || req.getMaxPrice() != null) {
            bool.filter(RangeQuery.of(r -> r
                .field("price")
                .gte(JsonData.of(req.getMinPrice()))
                .lte(JsonData.of(req.getMaxPrice()))
            )._toQuery());
        }

        if (req.getStatus() != null) {
            bool.filter(TermQuery.of(t -> t
                .field("status")
                .value(req.getStatus().name())
            )._toQuery());
        }

        return bool.build()._toQuery();
    }
}
```

---

## 7. 数据一致性保障策略

```java
/**
 * Outbox 模式：在同一事务内写 DB + outbox 表
 * 由 Outbox Poller 定时扫描发送，彻底解决 MQ 发送失败丢消息
 */
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id @GeneratedValue
    private Long id;
    private String aggregateType;
    private Long aggregateId;
    private String eventType;
    @Column(columnDefinition = "JSON")
    private String payload;
    private boolean published;
    private LocalDateTime createdAt;
}

@Component
@RequiredArgsConstructor
public class OutboxPoller {

    private final OutboxEventRepository outboxRepo;
    private final KafkaTemplate<String, String> kafka;

    @Scheduled(fixedDelay = 1000)
    @Transactional
    public void poll() {
        outboxRepo.findUnpublished(PageRequest.of(0, 100)).forEach(e -> {
            kafka.send("product.changes", String.valueOf(e.getAggregateId()), e.getPayload());
            e.setPublished(true);
            outboxRepo.save(e);
        });
    }
}
```

---

## 8. ES 索引配置 (`es/product-settings.json`)

```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "ik_max_word": { "type": "custom", "tokenizer": "ik_max_word" },
        "ik_smart":    { "type": "custom", "tokenizer": "ik_smart" }
      }
    }
  }
}
```

---

## 架构关键决策总结

| 关注点 | 方案 | 原因 |
|---|---|---|
| 写顺序 | DB first → 事务提交后发 MQ | 避免 ES 写入了但 DB 回滚 |
| 消息可靠性 | Outbox 模式 | MQ 宕机不丢消息 |
| 乱序处理 | Consumer 回查 DB 最新值 | 消息乱序时 ES 数据仍正确 |
| 同步失败 | 重试 + DLQ | 自动补偿 + 人工介入入口 |
| 全量修复 | Full Sync Job | ES 索引损坏时可完整重建 |
| 读写分离 | 写只查 DB，搜索只查 ES | 职责清晰，ES 不作主数据源 |

需要我深入展开某一层（比如 Outbox 完整实现、ES 聚合查询、CDC Debezium 接入方式）吗？