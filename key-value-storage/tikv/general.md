# 🥳 General

先放一个官方文档中的总架构图：

![基于 RocksDB 的分布式 KV 数据库架构](<../../.gitbook/assets/image (2).png>)

一些基本名词：

* **Placement Driver：** PD 是 TiKV 的集群管理器，它定期检查复制约束以自动平衡负载和数据。
* **Store：**每个Store内部都有一个RocksDB，将数据存储到本地磁盘中。
* **Region：** Region是Key-Value数据移动的基本单位。每个Region被复制到多个节点。这些多个副本组成了一个 Raft 组。
*   **Node：**集群中的一个物理节点。在每个节点内，有一个或多个 Store。在每个 Store 中，有许多 Region。

    当一个节点启动时，Node、Store 和 Region 的元数据都会记录到 PD 中。每个 Region 和 Store 的状态都会定期上报给 PD。

### **multi-raft-group**

与传统的整节点备份方式不同，TiKV 参考 Spanner 设计了 <mark style="color:purple;">**multi-raft-group**</mark> 的副本机制：

将数据按照 key 的范围划分成大致相等的切片（下文统称为 <mark style="color:blue;">**Region**</mark>），每个Region的数据都保存在一个节点上。Tikv是以Region为单位做数据的复制，每一个Region会有多个副本Replica（通常是 3 个），多个Replica保存在不同的节点上，构成一个Raft Group。其中一个副本是 <mark style="color:blue;">**Leader**</mark>，提供读写服务，其余副本为Follower。所有的读和写都是通过Leader进行，再由Leader复制给Follower。

TiKV 通过 ** **<mark style="color:purple;">**PD**</mark>** ** 对这些 Region 以及副本进行调度，以保证将Region尽可能均匀的散布在集群的所有的节点上，一方面实现了存储容量的水平扩展（新增节点，会自动将其他节点中的Region调度过来），另一方面也实现了负载均衡（不会出现某个节点有很多数据，而其他节点没有什么数据的情况）。同时为了保证上次客户端能够访问所需要的数据。由TiKV driver来将上层语句映射到正确的节点。

这样的设计保证了整个集群资源的充分利用并且可以随着机器数量的增加水平扩展。

不过虽然 TiKV 将数据按照范围切割成了多个 Region，但是**同一个节点的所有 Region 数据仍然是不加区分地存储于同一个 RocksDB 实例上**，而用于 Raft 协议复制所需要的**日志**则存储于另一个 RocksDB 实例。这样设计的**原因**是随机 I/O 的性能远低于顺序 I/O，所以 TiKV 使用同一个 RocksDB 实例来存储这些数据，**以便**不同 Region 的写入可以合并在一次 I/O 中。

Region 与副本之间通过 Raft 协议来维持数据一致性，任何**写请求都只能在 Leader 上写入**，并且需要写入**多数**副本后（默认配置为 3 副本，即所有请求必须至少写入两个副本成功）才会返回客户端写入成功。

当某个 Region 的大小超过一定限制（默认是 144MB）后，TiKV 会将它分裂为两个或者更多个 Region，以保证**各个 Region 的大小是大致接近的**，这样更有利于 PD 进行调度决策。同样，当某个 Region 因为大量的删除请求导致 Region 的大小变得更小时，TiKV 会将比较小的两个相邻 Region 合并为一个。

当 PD 需要把某个 Region 的一个副本从一个 TiKV 节点调度到另一个上面时，PD 会先为这个 Raft Group **在目标节点上增加一个 Learner 副本**（虽然会复制 Leader 的数据，但是不会计入写请求的多数副本中）。当这个 Learner 副本的进度大致追上 Leader 副本时，Leader 会将它变更为 Follower，之后再移除操作节点的 Follower 副本，这样就完成了 Region 副本的一次调度。

Leader 副本的调度原理也类似，不过需要在目标节点的 Learner 副本变为 Follower 副本后，**再执行一次 Leader Transfer**，让该 Follower 主动发起一次选举成为新 Leader，之后新 Leader 负责删除旧 Leader 这个副本。

![解决问题](https://pic2.zhimg.com/v2-abe687a4a9998db8d8c8b696200978e5\_b.jpg)

Region扩张时如果存在Leader的问题，可以像这个例子一样解决：

节点A中三个Region，其他节点两个Region。为了缓解节点A的压力，将节点A中的Region 1 转移到新建节点E。但此时由于Region1的leader在节点A中，所以会先把Leader节点从节点A转到节点B。 之后在节点E中增加一个Region 1的副本。再从节点A中移除Region 1的副本。所有这一切都被Placement Driver自动执行的。唯一要做的就是发现系统繁忙时就添加节点。

Raft的优化：

* **初始的Raft** ： Leader 收到 client 发送的 request <mark style="color:blue;">**→**</mark>  Leader 将 request append 到自己的 log <mark style="color:blue;">**→**</mark> Leader 将对应的 log entry 发送给其他的 follower  <mark style="color:blue;">**→**</mark>  <mark style="color:blue;"></mark><mark style="color:blue;"></mark>  Leader 等待 follower 的结果，如果大多数节点提交了这个 log，则 apply  <mark style="color:blue;">**→**</mark>  Leader 将结果返回给 client  <mark style="color:blue;">**→**</mark>  Leader 继续处理下一次 request。
* **利用异步apply改进Raft：**Leader 接受一个 client 发送的 request <mark style="color:blue;">**→**</mark>  Leader 将对应的 log 发送给其他 follower 并本地 append <mark style="color:blue;">**→**</mark>  Leader 继续接受其他 client 的 requests，持续进行步骤 2 <mark style="color:blue;">**→**</mark>  Leader 发现 log 已经被 committed，在另一个线程 apply <mark style="color:blue;">**→**</mark>  Leader 异步 apply log 之后，返回结果给对应的 client。

### &#x20;Placement Driver

Placement Driver监控的数据有： - 总磁盘容量 - 可用磁盘容量 - 承载的 Region 数量 - 数据写入速度 - 发送/接受的 Snapshot 数量(Replica 之间可能会通过 Snapshot 同步数据) - 是否过载 - 标签信息 - 标签是具备层级关系的一系列 Tag

![整体架构](<../../.gitbook/assets/image (1) (1).png>)

每个 TiKV 实例的架构如下**图**所示：

![instance](https://tikv.org/img/tikv-instance.png)

![Node model](<../../.gitbook/assets/image (3).png>)

Placement Driver是整个系统中的一个节点，它会时刻知道现在整个系统的状态。比如说每个机器的负载，每个机器的容量，是否有新加的机器，新加机器的容量到底是怎么样的，是不是可以把一部分数据挪过去，是不是也是一样下线， 如果一个节点在十分钟之内无法被其他节点探测到，认为它已经挂了，不管它实际上是不是真的挂了，但也认为它挂了。因为这个时候是有风险的，如果这个机器万一真的挂了，意味着现在机器的副本数只有两个，有一部分数据的副本数只有两个。那么现在必须马上要在系统里面重新选一台机器出来，它上面有足够的空间，现在只有两个副本的数据重新再做一份新的复制，系统始终维持在三个副本。整个系统里面如果机器挂掉了，副本数少了，这个时候应该会被自动发现，马上补充新的副本，这样会维持整个系统的副本数。这是很重要的 ，为了避免数据丢失，必须维持足够的副本数，因为副本数每少一个，风险就会再增加。这就是Placement Driver做的事情。

同时，Placement Driver 还会根据性能负载，不断去move这个data 。比如说负载已经很高了，一个磁盘假设有 100G，现在已经用了 80G，另外一个机器上也是 100G，但是只用了 20G，所以这上面还可以有几十 G 的数据，比如 40G 的数据，可以 move 过去，这样可以保证系统有很好的负载，不会出现一个磁盘巨忙无比，数据已经多的装不下了，另外一个上面还没有东西，这是 Placement Driver 要做的东西。

### Multiversion concurrency control(MVCC)

MVCC指的是多版本并发控制，并发访问（读或者写）数据库时，对正在事务内处理的数据做多版本的管理，用来避免由于写操作的堵塞，而引发读操作失败的并发问题。

![](https://pic3.zhimg.com/v2-e2d7aa78b5fb166788d4bb3b13019286\_b.jpg)

设想这样的场景，两个 Client 同时去修改一个 Key 的 Value，如果没有 MVCC，就需要对数据上锁，在分布式场景下，可能会带来性能以及死锁问题。 TiKV 的 MVCC 实现是通过在 Key 后面添加 Version 来实现，简单来说，没有 MVCC 之前，可以把 TiKV 看做这样的:

![](https://pic3.zhimg.com/v2-00fd826e0485873e69f4e5b2db6ab852\_b.jpg)

```cpp
Key1 -> Value
    Key2 -> Value
    ……
    KeyN -> Value
```

有了 MVCC 之后，TiKV 的 Key 排列是这样的：

![](https://pic2.zhimg.com/v2-a8cac90fbf78f140ce73a29dc8498719\_b.jpg)

```cpp
Key1-Version3 -> Value
    Key1-Version2 -> Value
    Key1-Version1 -> Value
    ……
    Key2-Version4 -> Value
    Key2-Version3 -> Value
    Key2-Version2 -> Value
    Key2-Version1 -> Value
    ……
    KeyN-Version2 -> Value
    KeyN-Version1 -> Value
    ……
```

对于同一个 Key 的多个版本，我们把版本号较大的放在前面，版本号小的放在后面，这样当用户通过一个 Key + Version 来获取 Value 的时候，可以将 Key 和 Version 构造出 MVCC 的 Key，也就是 Key-Version。然后可以直接 Seek(Key-Version)，定位到第一个大于等于这个 Key-Version 的位置。

### 事务

TiKV 的事务模型类似于 Google 的[Percolator](https://ai.google/research/pubs/pub36726)，这是一个为处理大型数据集的更新而构建的系统。Percolator 使用增量更新模型代替基于批处理的模型。

TiKV 的交易模型提供：

* 带锁的**快照隔离（Snapshot isolation）**`SELECT ... FOR UPDATE`，语义类似于SQL
* 分布式事务中的外部一致性读写

TiKV 支持分布式事务，用户（或者 TiDB）可以一次性写入多个 key-value 而不必关心这些 key-value 是否处于同一个数据切片 (Region) 上，TiKV 通过**两阶段提交**保证了这些读写请求的 ACID 约束，详见 [TiDB 乐观事务模型](https://docs.pingcap.com/zh/tidb/dev/optimistic-transaction)（TiKV 的事务采用乐观锁，事务的执行过程中，不会检测写写冲突，只有在提交过程中，才会做冲突检测，冲突的双方中比较早完成提交的会写入成功，另一方会尝试重新执行整个事务）。

当业务的写入冲突不严重的情况下，这种模型性能会很好，比如随机更新表中某一行的数据，并且表很大。但是如果业务的写入冲突严重，性能就会很差，举一个极端的例子，就是计数器，多个客户端同时修改少量行，导致冲突严重的，造成大量的无效重试。
