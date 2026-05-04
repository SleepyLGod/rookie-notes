# 😇 CAP theorem

### CAP定理（CAP theorem）

在计算机科学中，CAP 定理（CAP theorem）又被称作布鲁尔定理（Brewer's theorem）。它讨论的是发生网络分区时，分布式系统在一致性和可用性之间必须做出的取舍。

CAP 中三个概念更准确地说是：

* **一致性（Consistency）**：读写表现得像访问单个最新副本，通常接近线性一致/单副本语义。
* **可用性（Availability）**：每个非故障节点收到请求后，都必须在有限时间内返回非错误响应。
* **分区容忍（Partition tolerance）**：即使节点之间发生网络分区、消息丢失或延迟，系统仍然继续运行。

CAP 理论的核心不是简单地“最多只能选两个”，而是：在真实分布式系统中，网络分区是必须面对的故障模型；一旦分区发生，系统无法同时保持强一致和完全可用，只能选择牺牲其中一个。

因此，常见的 CA/CP/AP 分类需要谨慎理解：

* **CA**：通常只适合没有网络分区假设的单机或强受控环境；在真实跨节点系统中，P 不是可以随意关闭的属性。
* **CP**：分区发生时优先保持一致性，必要时拒绝或阻塞部分请求。
* **AP**：分区发生时优先保持可用性，允许返回旧数据或产生短暂不一致，再通过后续机制收敛。

![img](https://www.runoob.com/wp-content/uploads/2013/10/cap-theoram-image.png)

### NoSQL的优点/缺点

优点:

* \- 高可扩展性
* \- 分布式计算
* \- 低成本
* \- 架构的灵活性，半结构化数据
* \- 没有复杂的关系

缺点:

* \- 没有标准化
* \- 有限的查询功能（到目前为止）
* \- 最终一致是不直观的程序

### BASE

BASE：Basically Available, Soft-state, Eventually Consistent。它通常用于描述在可用性和伸缩性优先的系统中，对强一致性进行放松后的设计取向。

BASE是NoSQL数据库通常对可用性及一致性的弱要求原则:

* Basically Available --基本可用
* Soft-state --软状态/柔性事务。 "Soft state" 可以理解为"无连接"的, 而 "Hard state" 是"面向连接"的
* Eventually Consistency -- 最终一致性， 也是 ACID 的最终目的。

### ACID vs BASE

| ACID                 | BASE                              |
| -------------------- | --------------------------------- |
| 原子性(**A**tomicity)   | 基本可用(**B**asically **A**vailable) |
| 一致性(**C**onsistency) | 软状态/柔性事务(**S**oft state)          |
| 隔离性(**I**solation)   | 最终一致性 (**E**ventual consistency)  |
| 持久性 (**D**urable)    |                                   |

### NoSQL 数据库分类

| 类型          | 部分代表                                             | 特点                                                                  |
| ----------- | ------------------------------------------------ | ------------------------------------------------------------------- |
| 列存储         | HbaseCassandraHypertable                         | 顾名思义，是按列存储数据的。最大的特点是方便存储结构化和半结构化数据，方便做数据压缩，对针对某一列或者某几列的查询有非常大的IO优势。 |
| 文档存储        | MongoDBCouchDB                                   | 文档存储一般用类似json的格式存储，存储的内容是文档型的。这样也就有机会对某些字段建立索引，实现关系数据库的某些功能。        |
| key-value存储 | Tokyo Cabinet / TyrantBerkeley DBMemcacheDBRedis | 可以通过key快速查询到其value。一般来说，存储不管value的格式，照单全收。（Redis包含了其他功能）            |
| 图存储         | Neo4JFlockDB                                     | 图形关系的最佳存储。使用传统关系数据库来解决的话性能低下，而且设计使用不方便。                             |
| 对象存储        | db4oVersant                                      | 通过类似面向对象语言的语法操作数据库，通过对象的方式存取数据。                                     |
| xml数据库      | Berkeley DB XMLBaseX                             | 高效的存储XML数据，并支持XML的内部查询语法，比如XQuery,Xpath。                            |
