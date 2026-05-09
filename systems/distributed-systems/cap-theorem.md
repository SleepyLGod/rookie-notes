# CAP Theorem

### CAP theorem

In computer science, the CAP theorem is also known as Brewer's theorem. It describes the tradeoff a distributed system must make between consistency and availability when a network partition occurs.

The three concepts in CAP are more precisely:

* **Consistency**: reads and writes behave as if they access a single up-to-date replica. This is usually close to linearizability or single-copy semantics.
* **Availability**: every request received by a non-failing node must eventually return a non-error response.
* **Partition tolerance**: the system continues operating even when network partitions, message loss, or message delays occur between nodes.

The core of CAP is not simply "you can choose at most two." In real distributed systems, network partitions are a failure model that must be considered. Once a partition happens, the system cannot simultaneously preserve strong consistency and complete availability; it must sacrifice one of them.

Therefore, the common CA/CP/AP classification needs to be interpreted carefully:

* **CA**: usually only applies to a single-node system or a tightly controlled environment where network partitions are not part of the model. In real cross-node systems, P is not a property that can simply be turned off.
* **CP**: when a partition occurs, the system prioritizes consistency and may reject or block some requests.
* **AP**: when a partition occurs, the system prioritizes availability and may return stale data or allow temporary inconsistency, then converge later through reconciliation mechanisms.

![img](https://www.runoob.com/wp-content/uploads/2013/10/cap-theoram-image.png)

### Advantages and disadvantages of NoSQL

Advantages:

* High scalability
* Distributed computation
* Low cost
* Flexible schema design and support for semi-structured data
* No complex relational model

Disadvantages:

* Lack of standardization
* Limited query capabilities in some systems
* Eventual consistency can be unintuitive for application developers

### BASE

BASE stands for Basically Available, Soft-state, Eventually Consistent. It is commonly used to describe a design direction that relaxes strong consistency in systems that prioritize availability and scalability.

BASE is often used to describe the weaker availability and consistency principles adopted by NoSQL databases:

* Basically Available: the system remains basically available under failure or load, possibly with degraded behavior.
* Soft-state: the system state may change over time even without new input, because of asynchronous replication, expiration, or reconciliation. In a loose sense, "soft state" can be understood as less tightly bound than "hard state."
* Eventually Consistent: if no new updates occur, replicas eventually converge to the same value.

### ACID vs BASE

| ACID                    | BASE                       |
| ----------------------- | -------------------------- |
| **A**tomicity           | **B**asically **A**vailable |
| **C**onsistency         | **S**oft state             |
| **I**solation           | **E**ventual consistency   |
| **D**urability          |                            |

### NoSQL database categories

| Type            | Representative systems                              | Characteristics                                                                                                                                                  |
| --------------- | --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Column store    | HBase, Cassandra, Hypertable                        | Data is stored by column. The main advantage is efficient storage of structured and semi-structured data, easier compression, and large I/O advantages for queries over one or a few columns. |
| Document store  | MongoDB, CouchDB                                    | Documents are usually stored in a JSON-like format. Because the stored content is document-shaped, indexes can be built on selected fields, enabling some relational-database-like functionality. |
| Key-value store | Tokyo Cabinet / Tyrant, Berkeley DB, MemcacheDB, Redis | Values can be queried quickly by key. In general, the store accepts values regardless of their internal format. Redis also provides additional data-structure functionality. |
| Graph store     | Neo4J, FlockDB                                      | Best suited for storing graph relationships. Solving graph-relationship problems with a traditional relational database can have poor performance and inconvenient modeling. |
| Object store    | db4o, Versant                                       | Data is accessed and stored as objects through syntax similar to object-oriented programming languages. |
| XML database    | Berkeley DB XML, BaseX                              | Efficiently stores XML data and supports XML query languages such as XQuery and XPath. |
