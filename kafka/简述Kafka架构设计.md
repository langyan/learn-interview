# 简述 Kafka 架构设计

- **Consumer Group（消费者组）**：消费者组内每个消费者负责消费不同分区的数据，提高消费能力。逻辑上的一个订阅者。
- **Topic**：可以理解为一个队列，Topic 将消息分类，生产者和消费者面向的是同一个 Topic。
- **Partition**：为了实现扩展性，提高并发能力，一个 Topic 以多个 Partition 的方式分布到多个 Broker 上，每个 Partition 是一个有序的队列。一个 Topic 的每个 Partition 都有若干个副本（Replica），一个 Leader 和若干个 Follower。生产者发送数据的对象，以及消费者消费数据的对象，都是 Leader。Follower 负责实时从 Leader 中同步数据，保持和 Leader 数据的同步。Leader 发生故障时，某个 Follower 还会成为新的 Leader。
- **Offset**：消费者消费的位置信息，监控数据消费到什么位置，当消费者挂掉再重新恢复的时候，可以从消费位置继续消费。
- **Zookeeper**：Kafka 集群能够正常工作，需要依赖于 Zookeeper，Zookeeper 帮助 Kafka 存储和管理集群信息。

# Kafka 在什么情况下会出现消息丢失及解决方案

## 消息发送

### 1. ack=0，不重试
producer 发送消息完，不管结果了，如果发送失败也就丢失了。

### 2. ack=1，leader crash
producer 发送消息完，只等待 lead 写入成功就返回了，leader crash 了，这时 follower 没来及同步，消息丢失。

### 3. unclean.leader.election.enable 配置 true
允许选举 ISR 以外的副本作为 leader，会导致数据丢失，默认为 false。producer 发送异步消息完，只等待 lead 写入成功就返回了，leader crash 了，这时 ISR 中没有 follower，leader 从 OSR 中选举，因为 OSR 中本来落后于 Leader 造成消息丢失。

## 解决方案

- **配置**：`ack=all / -1`，`tries > 1`，`unclean.leader.election.enable : false`
  - producer 发送消息完，等待 follower 同步完再返回，如果异常则重试。副本的数量可能影响吞吐量。
  - 不允许选举 ISR 以外的副本作为 leader。
- **配置**：`min.insync.replicas > 1`
  - 副本指定必须确认写操作成功的最小副本数量。如果不能满足这个最小值，则生产者将引发一个异常（要么是 NotEnoughReplicas，要么是 NotEnoughReplicasAfterAppend）。
  - `min.insync.replicas` 和 ack 更大的持久性保证。确保如果大多数副本没有收到写操作，则生产者将引发异常。
- **失败的 offset 单独记录**
  - producer 发送消息，会自动重试，遇到不可恢复异常会抛出，这时可以捕获异常记录到数据库或缓存，进行单独处理。

## 消费

- **先 commit 再处理消息**：如果在处理消息的时候异常了，但是 offset 已经提交了，这条消息对于该消费者来说就是丢失了，再也不会消费到了。

## broker 的刷盘

- 减小刷盘间隔

# Kafka 是 pull？push？优劣势分析

## pull 模式

**优点：**
- 根据 consumer 的消费能力进行数据拉取，可以控制速率
- 可以批量拉取、也可以单条拉取
- 可以设置不同的提交方式，实现不同的传输语义

**缺点：**
- 如果 kafka 没有数据，会导致 consumer 空循环，消耗资源

**解决：** 通过参数设置，consumer 拉取数据为空或者没有达到一定数量时进行阻塞

## push 模式

**优点：** 不会导致 consumer 循环等待

**缺点：** 速率固定、忽略了 consumer 的消费能力，可能导致拒绝服务或者网络拥塞等情况

# Kafka 中 zk 的作用

- `/brokers/ids`：临时节点，保存所有 broker 节点信息，存储 broker 的物理地址、版本信息、启动时间等，节点名称为 brokerID，broker 定时发送心跳到 zk，如果断开则该 brokerID 会被删除
- `/brokers/topics`：临时节点，节点保存 broker 节点下所有的 topic 信息，每一个 topic 节点下包含一个固定的 partitions 节点，partitions 的子节点就是 topic 的分区，每个分区下保存一个 state 节点、保存着当前 leader 分区和 ISR 的 brokerID，state 节点由 leader 创建，若 leader 宕机该节点会被删除，直到有新的 leader 选举产生、重新生成 state 节点
- `/consumers/[group_id]/owners/[topic]/[broker_id-partition_id]`：维护消费者和分区的注册关系
- `/consumers/[group_id]/offsets/[topic]/[broker_id-partition_id]`：分区消息的消费进度 Offset

> client 通过 topic 找到 topic 树下的 state 节点、获取 leader 的 brokerID，到 broker 树中找到 broker 的物理地址，但是 client 不会直连 zk，而是通过配置的 broker 获取到 zk 中的信息

# 简述 Kafka 的 rebalance 机制

consumer group 中的消费者与 topic 下的 partion 重新匹配的过程。

## 何时会产生 rebalance

- consumer group 中的成员个数发生变化
- consumer 消费超时
- group 订阅的 topic 个数发生变化
- group 订阅的 topic 的分区数发生变化

## rebalance 流程

1. **coordinator**：通常是 partition 的 leader 节点所在的 broker，负责监控 group 中 consumer 的存活，consumer 维持到 coordinator 的心跳，判断 consumer 的消费超时
2. coordinator 通过心跳返回通知 consumer 进行 rebalance
3. consumer 请求 coordinator 加入组，coordinator 选举产生 leader consumer
4. leader consumer 从 coordinator 获取所有的 consumer，发送 syncGroup（分配信息）给到 coordinator
5. coordinator 通过心跳机制将 syncGroup 下发给 consumer
6. 完成 rebalance

> leader consumer 监控 topic 的变化，通知 coordinator 触发 rebalance

## 问题与解决

**问题：** 如果 C1 消费消息超时，触发 rebalance，重新分配后、该消息会被其他消费者消费，此时 C1 消费完成提交 offset、导致错误

**解决：** coordinator 每次 rebalance，会标记一个 Generation 给到 consumer，每次 rebalance 该 Generation 会+1，consumer 提交 offset 时，coordinator 会比对 Generation，不一致则拒绝提交

# Kafka 的性能好在什么地方

- Kafka 不基于内存，而是硬盘存储，因此消息堆积能力更强
- **顺序写**：利用磁盘的顺序访问速度可以接近内存，kafka 的消息都是 append 操作，partition 是有序的，节省了磁盘的寻道时间，同时通过批量操作、节省写入次数，partition 物理上分为多个 segment 存储，方便删除

## 传统数据拷贝流程

1. 读取磁盘文件数据到内核缓冲区
2. 将内核缓冲区的数据 copy 到用户缓冲区
3. 将用户缓冲区的数据 copy 到 socket 的发送缓冲区
4. 将 socket 发送缓冲区中的数据发送到网卡、进行传输

## 零拷贝

- 直接将内核缓冲区的数据发送到网卡传输
- 使用的是操作系统的指令支持

> Kafka 不太依赖 JVM，主要理由操作系统的 pageCache，如果生产消费速率相当，则直接用 pageCache 交换数据，不需要经过磁盘 IO