# 如何给 10 亿级订单表安全加字段(MySQL)

> 日期:2025-10-23 | 阅读时间:8 分钟 | 专栏:MySQL

## 一、背景

作者曾在一家餐饮公司工作多年,公司运营时间久,MySQL 5.6 的订单表早已超过 **10 亿行**。经典问题随之而来:

**如何给这张超大表新增一个字段?**

## 二、当时的处理方式

假设业务需求在 2025 年 10 月 23 日上线,团队安排在 10 月 22 日或 23 日凌晨提交数据库变更:

```sql
ALTER TABLE order_items ADD COLUMN new_flag TINYINT DEFAULT 0;
```

- 变更通常需要 **4~6 小时** 完成
- 期间表被锁定,**无法写入**
- 但凌晨是门店打烊时间,基本没有客流,影响可控

## 三、当时是不是最佳方案?

**答案:是。** 原因如下:

1. 使用的是 MySQL 5.6,**不支持 Instant ADD COLUMN**(该特性在 MySQL 8.0.12 引入)
2. 当时使用腾讯云 MySQL,其"无锁结构变更"功能尚未公开可用
3. 团队(开发 + DBA)缺乏使用 gh-ost 等工具的能力和经验
4. 公司提供了业务低峰期的运维窗口(凌晨门店关门时)

因此只能使用 MySQL 原生 DDL,注定**慢,但安全**。

## 四、更好的方案

主要有两种情况:

1. 使用 MySQL 8 的 **Instant ADD COLUMN** 特性
2. 使用 **gh-ost**(适用于 MySQL 5.6 / 5.7 / 8.0,但不满足 Instant 条件的场景)

---

## 方案一:使用 MySQL 8 的 Instant ADD COLUMN

### 原理

从 MySQL 8.0.12 开始支持 Instant ADD COLUMN,核心原理:

- **不重建表**
- **不拷贝数据**
- **只修改表的元数据**(.sdi 文件)

新增列的默认值通过一条**隐藏的默认值记录**实现,读取旧行时动态补全。

> 仅适用于:给表**末尾**添加可为 NULL 或带默认值的列。

### 传统加列 vs Instant 加列

| 对比项 | 传统方式(MySQL 5.7 / 早期 8.0) | Instant ADD COLUMN |
| --- | --- | --- |
| 是否重建表 | 是 | 否 |
| 是否读写每一行数据 | 是 | 否 |
| 是否修改 .ibd 数据文件 | 是 | 否 |
| 耗时 | 小时级(10 亿行) | 毫秒级(< 100ms) |
| 锁定时间 | 长(写锁持续数小时) | 极短(元数据锁 < 10ms) |

### 工作机制(惰性求值)

旧行(物理数据)→ 读取时(SELECT)→ 返回给服务端(包含新列)

- 物理 .ibd 文件保持不变
- 元数据中存储新列的默认值(隐藏的默认记录)

### 示例

```sql
-- 原表有 10 亿行,没有 new_flag 列
ALTER TABLE order_items ADD COLUMN new_flag TINYINT NULL DEFAULT 0;   -- 瞬间完成

SELECT new_flag FROM order_items WHERE id = 1;
-- InnoDB 返回:0(即使该列在旧行中并不物理存在)
```

### 为什么这么快?

1. **不读数据**:不扫描 .ibd 文件,不触碰任何数据页
2. **不写数据**:不修改数据行,不产生 redo/undo
3. **只改元数据**:仅更新数据字典(内存 + 少量日志)
4. **无 I/O 压力**:磁盘、CPU、内存消耗几乎为零
5. **极小的锁粒度**:仅短暂持有 MDL(元数据锁),时长在毫秒级

### 支持条件(必须全部满足)

- MySQL ≥ 8.0.12
- 新列位于最后一列
- 新列允许 NULL,或指定了 DEFAULT 值
- 表使用 InnoDB 引擎
- 不能是 FULLTEXT / SPATIAL 索引相关的表

### 支持 vs 不支持

**✅ 支持(Instant ADD):**

```sql
ALTER TABLE order_items
ADD COLUMN new_flag TINYINT NULL DEFAULT 0;
```

**❌ 不支持(触发全表重建):**

```sql
ALTER TABLE order_items
ADD COLUMN new_flag TINYINT NOT NULL;

ALTER TABLE order_items
ADD COLUMN new_flag TINYINT AFTER user_id;
```

> 如果你的业务数据库已升级到 MySQL 8.0.12+ 且满足上述条件,给 10 亿行的表加字段只需 **1 秒**!

---

## 方案二:使用 gh-ost

### 什么是 gh-ost?

gh-ost(GitHub Online Schema Change)是 GitHub 开源的在线 DDL 工具,专为拥有 **10 亿行** 级别数据的表设计。它可以在业务持续高并发写入的情况下,**无锁、安全、可控**地完成表结构变更。

### 相比原生 DDL 的优势(MySQL 5.7)

| 能力 | 原生 DDL(MySQL 5.7) | gh-ost |
| --- | --- | --- |
| 表是否会被锁定 | 长时间写锁 | 毫秒级锁,仅在切换时 |
| 是否支持限流 | 否 | 是 |
| 是否可暂停 | 否 | 是 |
| 是否支持 5.7 | ADD COLUMN 必须重建 | 完全支持 |
| 失败回滚成本 | 高(需要回滚并重来) | 低(直接删除影子表即可) |

### 工作流程(含持续写入场景)

1. **创建影子表**:创建 `_order_items_gho`(结构同原表并增加新列),瞬间完成,无锁
2. **分块拷贝**:按主键分批复制数据(支持限流,如 1000 行/秒),原表仍可写入
3. **Binlog 同步(核心)**:gh-ost 监听 ROW 格式 binlog,并实时将 DML 回放到影子表
4. **处理冲突**:通过 binlog 的最新变更覆盖旧的拷贝数据,保证最终一致性
5. **最终切换(Cut-over)**:原子性执行 `RENAME TABLE` 交换,瞬间切换到新表
6. **清理**:验证后删除旧表(`_order_items_del`)

### 典型 DML 通过 Binlog 同步

| 原表上的操作 | gh-ost 在影子表上的执行 |
| --- | --- |
| `INSERT INTO order_items (...) VALUES (...)` | `INSERT INTO _order_items_gho (...) VALUES (...)` |
| `UPDATE order_items SET price=100 WHERE id=123` | `UPDATE _order_items_gho SET price=100, new_flag=0 WHERE id=123` |
| `DELETE FROM order_items WHERE id=456` | `DELETE FROM _order_items_gho WHERE id=456` |

### 失败处理

如果 gh-ost 执行失败:

- 原表保持不受影响
- 影子表可直接删除
- 支持从中断处恢复

---

## 五、最终建议

| 场景 | 推荐方案 |
| --- | --- |
| MySQL ≥ 8.0.12 且满足 Instant 条件 | 使用原生 `ALTER TABLE`(秒级完成) |
| MySQL 5.6 / 5.7 / 8.0(不满足 Instant 条件)且使用云厂商 | 使用云厂商的"无锁变更"功能(基于 gh-ost) |
| 自建数据库 + 具备运维能力 | 自行部署 gh-ost 或 pt-online-schema-change |


--- 
## pt-online-schema-change工具的方案

pt 指 Percona Toolkit 里的 `pt-online-schema-change`(简称 pt-osc)。它和 gh-ost 解决的是同一个问题,也适用于 MySQL 5.6/5.7/8.0,区别在于它靠触发器同步增量,而 gh-ost 靠 binlog。

## 工作流程

1. 创建新表 `_order_items_new`,结构同原表,并执行你指定的 ALTER(比如加列)
2. 在原表上创建 INSERT / UPDATE / DELETE 三个触发器,把变更同步写到新表
3. 按主键分块把旧数据拷贝到新表,每块大小会根据服务器负载自动调节
4. 拷贝完成后,原子执行 `RENAME TABLE`,交换新旧表
5. 删除旧表和触发器

## 命令示例

```bash
pt-online-schema-change \
  --alter "ADD COLUMN new_flag TINYINT NULL DEFAULT 0" \
  D=mydb,t=order_items \
  --chunk-size=1000 --chunk-time=0.5 \
  --max-load Threads_running=50 \
  --critical-load Threads_running=100 \
  --max-lag=5 --check-interval=1 \
  --dry-run          # 先演练,确认无误后改成 --execute
```

要点:`--alter` 里不要写 `ALTER TABLE`,只写变更子句。正式执行前一定先跑 `--dry-run`。

## 与 gh-ost 对比

| 对比项 | pt-osc | gh-ost |
| --- | --- | --- |
| 增量同步方式 | 触发器 | 解析 binlog(ROW 格式) |
| 对原表写入的影响 | 有,每次 DML 都要多执行一次触发器写入,与业务事务同一个事务 | 几乎没有,不依赖触发器 |
| 暂停 / 动态调参 | 较弱,主要靠 `--max-load`、`--max-lag` 自动退让 | 强,可通过 socket 文件随时暂停、限流 |
| 复制延迟 | 变更走主从复制,从库可能延迟 | 可在从库上运行,不增加主库负担 |
| 外键表 | 支持(`--alter-foreign-keys-method`) | 基本不支持 |
| 上手成本 | 低,单条命令即可 | 稍高,需要理解 binlog 和切换机制 |
| 失败回滚 | 删除新表和触发器即可 | 删除影子表即可 |

## 使用注意

- 原表必须有主键或唯一索引,否则无法分块。
- 触发器会放大写入压力。订单表写入高峰期使用时,要盯住 `Threads_running` 和主从延迟,并设好 `--critical-load`,超限自动中止。
- 原表已有触发器时,MySQL 5.6 不支持同类型多个触发器,pt-osc 会拒绝执行。
- 需要预留约一倍的磁盘空间,10 亿行的表要提前确认。
- 切换瞬间需要短暂的元数据锁,如果有长事务,可能会等待或卡住业务,建议在低峰期切换。
- 默认拷贝完成后会自动删除旧表,删除大表本身会产生 I/O 压力。可以加 `--no-drop-old-table` 保留旧表,之后再择机处理。

## 选型建议

- 没有外键、写入量大、需要精细控速:优先 gh-ost
- 有外键、环境简单、希望一条命令搞定:用 pt-osc
- 已经是 MySQL 8.0.12+ 且满足 Instant 条件:两者都不用,直接 `ALTER TABLE`

需要的话,我可以把这一节追加到之前的 Markdown 文档里。