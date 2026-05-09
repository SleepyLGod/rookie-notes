# 🥳 General

Start with the overall architecture diagram from the official documentation:

![Distributed KV database architecture based on RocksDB](<../../../.gitbook/assets/image (2) (2) (1).png>)

Some basic terms:

* **Placement Driver:** PD is TiKV's cluster manager. It periodically checks replication constraints to automatically balance load and data.
* **Store:** each Store contains a RocksDB instance that stores data on local disk.
* **Region:** a Region is the basic unit for moving key-value data. Each Region is replicated to multiple nodes, and these replicas form a Raft group.
*   **Node:** a physical node in the cluster. Each node contains one or more Stores, and each Store contains many Regions.

    When a node starts, the metadata of Nodes, Stores, and Regions is recorded in PD. The status of each Region and Store is also reported to PD periodically.

### **multi-raft-group**

Unlike the traditional approach of backing up an entire node, TiKV adopts a replica mechanism inspired by Spanner: <mark style="color:purple;">**multi-raft-group**</mark>.

TiKV divides data into roughly equal slices by key range. These slices are referred to as <mark style="color:blue;">**Regions**</mark>. The data of each Region is stored on one node. TiKV replicates data at Region granularity. Each Region has multiple replicas, usually 3, stored on different nodes, and these replicas form a Raft Group. One replica is the <mark style="color:blue;">**Leader**</mark>, which serves reads and writes, while the remaining replicas are Followers. All reads and writes go through the Leader, and then the Leader replicates the changes to Followers.

TiKV uses <mark style="color:purple;">**PD**</mark> to schedule Regions and replicas, so that Regions are distributed as evenly as possible across all nodes in the cluster. This provides horizontal scaling of storage capacity, because when a new node is added, Regions from other nodes can be scheduled to it automatically. It also provides load balancing, preventing situations where one node stores a large amount of data while other nodes store very little. At the same time, TiKV must ensure that clients can access the data they need. The TiKV driver maps upper-layer statements to the correct node.

This design ensures that cluster resources are fully utilized and that the cluster can scale horizontally as the number of machines increases.

Although TiKV splits data by range into multiple Regions, **all Region data on the same node is still stored together in the same RocksDB instance**, while the **logs** required by Raft replication are stored in another RocksDB instance. The **reason** for this design is that random I/O is much slower than sequential I/O. TiKV stores these data in one RocksDB instance so writes from different Regions can be merged into one I/O operation.

Regions and replicas maintain consistency through the Raft protocol. Any **write request can only be written on the Leader**, and it is returned to the client as successful only after being written to a **majority** of replicas. With the default configuration of 3 replicas, every request must be written successfully to at least 2 replicas.

When the size of a Region exceeds a certain limit, 144 MB by default, TiKV splits it into two or more Regions to keep **Region sizes roughly close to each other**, which helps PD make scheduling decisions. Similarly, when a Region becomes smaller because of many delete requests, TiKV can merge two smaller adjacent Regions into one.

When PD needs to move one replica of a Region from one TiKV node to another, PD first adds a **Learner replica** of that Raft Group on the target node. Although the Learner copies data from the Leader, it is not counted in the majority for write requests. When the Learner's progress roughly catches up with the Leader, the Leader changes it into a Follower and then removes the Follower replica on the source node. This completes one Region replica scheduling operation.

Leader replica scheduling is similar. After the Learner replica on the target node becomes a Follower, TiKV performs one more **Leader Transfer**, allowing that Follower to initiate an election and become the new Leader. The new Leader then deletes the old Leader replica.

![Problem-solving example](https://pic2.zhimg.com/v2-abe687a4a9998db8d8c8b696200978e5\_b.jpg)

When a Region expands and the Leader becomes a concern, the problem can be handled as in this example:

Node A has three Regions, while other nodes have two Regions. To reduce pressure on node A, Region 1 on node A is moved to a newly created node E. However, because the Leader of Region 1 is currently on node A, TiKV first transfers the Leader from node A to node B. Then it adds a replica of Region 1 on node E, and finally removes the replica of Region 1 from node A. All of this is automatically executed by Placement Driver. The only manual action is to add nodes when the system becomes busy.

Raft optimization:

* **Initial Raft:** the Leader receives a request from a client <mark style="color:blue;">**→**</mark> the Leader appends the request to its own log <mark style="color:blue;">**→**</mark> the Leader sends the corresponding log entry to other Followers <mark style="color:blue;">**→**</mark> the Leader waits for Follower results; if a majority of nodes commit the log, the Leader applies it <mark style="color:blue;">**→**</mark> the Leader returns the result to the client <mark style="color:blue;">**→**</mark> the Leader continues processing the next request.
* **Improving Raft with asynchronous apply:** the Leader receives a request from a client <mark style="color:blue;">**→**</mark> the Leader sends the corresponding log to other Followers and appends it locally <mark style="color:blue;">**→**</mark> the Leader continues accepting requests from other clients and keeps repeating step 2 <mark style="color:blue;">**→**</mark> the Leader finds that the log has been committed and applies it in another thread <mark style="color:blue;">**→**</mark> after asynchronously applying the log, the Leader returns the result to the corresponding client.

### &#x20;Placement Driver

Placement Driver monitors the following data: total disk capacity, available disk capacity, number of Regions hosted, data write speed, number of Snapshots sent and received, whether a node is overloaded, label information, and hierarchical tags. Snapshots may be used to synchronize data between replicas.

![Overall architecture](<../../../.gitbook/assets/image (1) (1).png>)

The architecture of each TiKV instance is shown in the following **figure**:

![instance](https://tikv.org/img/tikv-instance.png)

![Node model](<../../../.gitbook/assets/image (3) (1).png>)

Placement Driver is a node in the system that always knows the current state of the entire system. For example, it knows the load of each machine, the capacity of each machine, whether a new machine has been added, the capacity of that new machine, whether some data can be moved there, and whether a machine should be taken offline. If a node cannot be detected by other nodes for ten minutes, PD treats it as failed, regardless of whether it has actually failed. This is because the situation is risky: if the machine has truly failed, then some data may now have only two replicas. The system must immediately choose another machine with enough space and create a new copy of the data that has only two replicas. The system must maintain three replicas. If a machine fails and the replica count drops, the system should automatically detect this and replenish replicas immediately. This is important: to avoid data loss, the system must maintain enough replicas, because every lost replica increases risk. This is what Placement Driver does.

At the same time, Placement Driver continuously moves data according to performance load. For example, suppose one disk has 100 GB capacity and has already used 80 GB, while another machine also has 100 GB capacity but has only used 20 GB. The latter can still store tens of GB of data, such as 40 GB, so PD can move data there. This keeps the system balanced and avoids a situation where one disk is extremely busy and almost full while another disk has little data. This is also Placement Driver's responsibility.

### Multiversion Concurrency Control (MVCC)

MVCC stands for multiversion concurrency control. When concurrently accessing a database, whether for reads or writes, MVCC manages multiple versions of data being processed inside transactions. This avoids concurrency problems where reads fail because they are blocked by writes.

![](https://pic3.zhimg.com/v2-e2d7aa78b5fb166788d4bb3b13019286\_b.jpg)

Consider this scenario: two clients modify the value of the same key at the same time. Without MVCC, the data would need to be locked, which can bring performance and deadlock problems in a distributed setting. TiKV implements MVCC by appending a version to the key. Simply put, without MVCC, TiKV can be viewed as:

![](https://pic3.zhimg.com/v2-00fd826e0485873e69f4e5b2db6ab852\_b.jpg)

```cpp
Key1 -> Value
    Key2 -> Value
    ...
    KeyN -> Value
```

With MVCC, TiKV's key ordering looks like this:

![](https://pic2.zhimg.com/v2-a8cac90fbf78f140ce73a29dc8498719\_b.jpg)

```cpp
Key1-Version3 -> Value
    Key1-Version2 -> Value
    Key1-Version1 -> Value
    ...
    Key2-Version4 -> Value
    Key2-Version3 -> Value
    Key2-Version2 -> Value
    Key2-Version1 -> Value
    ...
    KeyN-Version2 -> Value
    KeyN-Version1 -> Value
    ...
```

For multiple versions of the same key, larger version numbers are placed before smaller version numbers. When a user fetches a value through `Key + Version`, TiKV can construct the MVCC key, namely `Key-Version`, and then directly `Seek(Key-Version)` to locate the first position greater than or equal to that `Key-Version`.

### Transactions

TiKV's transaction model is similar to Google's [Percolator](https://research.google/pubs/large-scale-incremental-processing-using-distributed-transactions-and-notifications/), a system built for processing updates to large datasets. Percolator uses an incremental update model instead of a batch-processing model.

TiKV's transaction model provides:

* **Snapshot Isolation** with locks, such as `SELECT ... FOR UPDATE`, with semantics similar to SQL.
* Externally consistent reads and writes in distributed transactions.

TiKV supports distributed transactions. A user, or TiDB, can write multiple key-value pairs at once without caring whether those key-value pairs are in the same data slice, or Region. TiKV guarantees transaction semantics for cross-Region writes through a two-phase prewrite/commit protocol.

It is important to note that TiKV can no longer be summarized as "only using optimistic locking". Early TiKV first implemented optimistic transactions: during transaction execution, the client accumulates read and write sets locally, and locks written keys and detects conflicts only during prewrite at commit time. If conflicts are heavy, this can produce many retries. Later, TiKV also added pessimistic transactions, and pessimistic transactions have become the default path: locking reads such as `SELECT ... FOR UPDATE` read and lock keys first, and write statements also acquire locks first, although the actual write is still delayed until the prewrite/commit phase.

Therefore, when understanding TiKV transactions, distinguish optimistic 2PC, pessimistic locking reads/writes, and isolation semantics such as Snapshot Isolation and Read Committed under MVCC. When business write conflicts are not severe, the optimistic model has lower cost. High-conflict scenarios such as hot rows and counters rely more on pessimistic locks, retries, and hotspot mitigation strategies.
