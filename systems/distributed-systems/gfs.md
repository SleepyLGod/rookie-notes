---
description: The second paper in Google's classic infrastructure trilogy.
---

# Google File System

### **Main Requirements of GFS**

* Node failure is the norm. The system is built from a large number of commodity machines, so machine failures are expected rather than exceptional. GFS must provide strong **fault tolerance**, continuously monitor its own state, and recover quickly from node failures.
* **The stored data is mostly large files**. In the common case, the system stores a relatively small number of large files, each usually hundreds of MB or several GB in size. The system should support small files, but it does not need to optimize for them.
* The dominant workloads are **large streaming reads**, **small random reads**, and **append-style streaming writes**.
* The system should support **efficient and atomic file append operations**, because in Google's setting these files are often used in producer-consumer pipelines or multi-way merge workloads.
* When the system must choose, it should prefer **high aggregate throughput over low latency**.

### **GFS Cluster Components**

At a high level, apart from clients, a GFS cluster contains one **Master** and multiple **Chunk Servers**. They run as user-level processes on ordinary Linux machines.

When **storing a file**, GFS splits the file into **fixed-size chunks**.

When the Master creates a chunk, it assigns the chunk a **globally unique 64-bit handle** and hands it to Chunk Servers. A Chunk Server stores each chunk as a normal file on its local disk. To keep chunks available, **GFS stores each chunk as multiple replicas distributed across different Chunk Servers**.

The Master maintains cluster metadata, including the cluster namespace, chunk lease management, garbage chunk collection, and other system-level operations.

A Chunk Server stores chunks and periodically communicates with the Master through heartbeat messages.

Through these heartbeats, the Master collects each Chunk Server's current state and sends instructions back to it.

Because the whole cluster has only one Master, a client first contacts the Master to obtain GFS metadata.

Actual file data transfer happens directly between the client and Chunk Servers, so that the Master does not become the data-transfer bottleneck.

In addition, clients cache metadata returned by the Master for a limited time.

#### **Metadata**

All GFS cluster metadata is stored in the Master's **memory**. Since the cluster has only one Master, metadata management becomes much simpler.

GFS metadata mainly consists of three kinds of information:

* The file and chunk namespace.
* The mapping between files and chunks.
* The location of every chunk replica.

Keeping metadata in the Master's memory makes metadata changes very easy for the Master. It also lets the Master scan cluster metadata efficiently in order to trigger system-level management operations such as chunk garbage collection and chunk rebalancing.

The only drawback is that the total **number of chunks in the cluster is limited by the Master's memory size**. According to the paper, this bottleneck was never reached at Google, because for a 64 MB chunk the Master needs less than 64 bytes of metadata. Moreover, compared with increasing code complexity, adding more memory to the Master is much cheaper.

To keep metadata available, before the Master performs **any** metadata operation, it first records the operation in a **WAL-style operation log**. Only after the log record is written does it apply the actual operation. These **logs are also replicated** to multiple machines.

Chunk replica locations are not persisted in the log. Instead, **when the Master starts, it asks each Chunk Server which replicas it currently has**. This avoids the cost of synchronizing location state between the Master and Chunk Servers, and further simplifies log persistence on the Master. After all, the Chunk Server itself is the authority on which replicas it actually stores.

### **Data Consistency**

When users work with a storage system like GFS, they first need to understand the consistency it provides. As readers, we should also start by understanding the consistency behavior that GFS **exposes externally**.

**Namespace:**

The namespace is managed entirely by the single Master in memory. Concurrent modification can be handled by having the Master use mutual exclusion for namespace changes. Therefore, namespace modifications can be made **fully atomic**.

**File:**

File data modification is more complicated.

After a region of a file is modified, it may be in one of the following three states:

* If clients may read different contents from different replicas, that file region is **inconsistent**.
* If every client reads the same content regardless of which replica it reads from, that file region is **consistent**.
* If every client can **see the complete content of the last successful mutation**, and that region is also consistent, then the region is **defined**.

After a mutation, the current state of a file region depends on the **type** of mutation and whether the mutation **succeeds**. More concretely:

* If a write succeeds and does **not overlap** with any concurrent write, the file region is **defined** and therefore also consistent.
* If several writes execute successfully at the same time, the region is **consistent** but **undefined**. In this case, the data visible to clients usually cannot be interpreted as the result of any one of the concurrent writes.
* A failed write leaves the region **inconsistent**.

[![img](https://mr-dai.github.io/img/gfs/consistency-model.png)](https://mr-dai.github.io/img/gfs/consistency-model.png)

GFS supports two kinds of file data mutation:

* Writing data at a specified offset, called Write.
* Appending data, called Record Append.

> **Write, or random write:**

For a write, the specified data is written directly at the offset chosen by the client and **overwrites** the old data.

GFS does not provide strong consistency guarantees for this operation. If different clients **write concurrently to the same file region**, the resulting region may contain **fragments** from multiple writes and become **undefined**.

> **Record Append, or append-at-end write:**

Record Append atomically appends data smaller than 16 MB to the end of a specified file.

GFS provides this interface because Record Append is not simply a Write whose offset is set to the current end of the file. It is a different operation with atomicity guarantees.

GFS guarantees that even under concurrency, Record Append is atomic and at-least-once.

After the operation finishes, GFS returns the **actual offset where the data was written** to the client. This offset marks the start of a **defined** file region containing the appended data. Because Record Append is at-least-once, GFS may insert padding or duplicate data into the file, although this should be uncommon.

When reading, applications usually write extra metadata such as checksums together with the data before appending it. This lets the GFS client library **validate the data** and automatically skip padding or corrupted records during reads.

However, because Record Append is at-least-once, a client may still read duplicate records. In that case, the **upper-level application must filter duplicates**, for example by checking a unique record ID.

**Impact on Applications**

GFS has a relatively relaxed consistency model, so applications using GFS must adapt to the semantics it provides. In simple terms, applications can do this in two ways:

* **Prefer append operations** over overwrite operations.
* Write data together with validation information.

The reason to prefer append over overwrite is clear. GFS is heavily optimized for append operations, which makes this write pattern faster and also gives it **stronger consistency semantics**. Nevertheless, because append is at-least-once, clients may still read padding or duplicate records, so clients must tolerate this invalid data.

One practical approach is to **write a checksum for every record and verify it during reads**, so invalid data can be discarded.

If clients cannot tolerate duplicate data, **they can also write a unique identifier for every record** and use that identifier to remove duplicates during reads.

### **Common Operation Flows in a GFS Cluster**

#### **Master Namespace Management**

The namespace is part of GFS metadata. It is kept in the Master's memory and managed by the Master.

Logically, the GFS Master does not manage this data with a hierarchical structure based on file-directory relationships. Instead, it represents the namespace as a mapping from **full pathnames to file metadata**, and applies **prefix compression** to pathnames to reduce memory consumption.

To handle concurrent namespace modifications from different clients, the Master assigns a **read-write lock** to every file and directory in the namespace. This allows concurrent requests that touch different namespace regions to proceed at the same time.

Every Master operation must acquire a sequence of locks before it executes.

Usually, when an operation involves a path such as `/d1/d2/.../dn/leaf`, the Master first acquires read locks from `/d1`, `/d1/d2`, through `/d1/d2/.../dn`, and then acquires either a read lock or a write lock on `/d1/d2/.../dn/leaf` depending on the operation. The reason for acquiring read locks on parent directories is to prevent a parent directory from being renamed or deleted while the operation is running.

Because a large number of read-write locks could consume significant memory, these locks are created only when needed and destroyed when no longer needed.

In addition, all **lock acquisition** follows a consistent order to avoid **deadlocks**:

Locks are ordered first by namespace tree level. Within the same level, they are ordered lexicographically by pathname.

#### **Reading a File**

The process for reading data from a GFS cluster is roughly as follows:

![img](https://pic2.zhimg.com/v2-670802df1fc494933e0a4edf61e3791d\_b.jpg)

* Given a **file name and a read offset**, the **client** can compute which **chunk** contains that offset based on the fixed chunk size.
* The client sends a request to the Master containing the **file name and chunk index**.
* The Master responds with the chunk handle and the locations of all current replicas of that chunk, as reported by Chunk Servers.

    The client caches this information using the **file name and chunk index** as the key.
* The client then **chooses one Chunk Server that has a replica and sends it a request**. The request specifies the **chunk handle and the byte range to read**.

#### **Chunk Lease**

When a client modifies **a chunk**, GFS handles concurrent modifications by **granting the chunk's lease to one replica, making it the Primary**.

The Primary assigns a **serial order** to all modifications, and the other replicas apply those modifications in the same order.

The initial timeout for a **chunk lease is 60 seconds**.

Before the lease expires, the Primary can ask the Master to **extend** the lease.

When necessary, the Master can also **revoke** a lease that it previously granted.

#### **Writing a File**

The process for a client writing data to **a specified offset in a chunk** is roughly as follows:

![img](https://pic4.zhimg.com/v2-d92b00ff01399aa34176c04f2c14a79b\_b.jpg)

* The client asks the Master which **Chunk Server** currently holds the lease for the chunk.
* The Master returns the locations of the Primary and the other replicas.
* **The client pushes the data to all replicas**. Chunk Servers store the data in buffers and wait for the actual write request.
* After all replicas have received the data, **the client sends the write request to the Primary**.

    The Primary assigns consecutive sequence numbers to mutation requests from different clients, then applies them to its local stored data in that order.
* The Primary forwards the **write request to the Secondary replicas**. The replicas apply the mutations in the same order.
* The Secondary replicas respond to the Primary to indicate completion.
* The Primary **responds to the client** and reports any errors that occurred.

If an error occurs during this process, the mutation may have succeeded on the Primary and only some of the Secondaries. If the error occurs on the Primary, the write request will not be forwarded to the Secondaries.

At that point, the mutation is considered unsuccessful because the data is now **inconsistent**. In practice, the GFS client library retries the operation.

It is worth noting that this process deliberately separates the **data flow from the control flow**. The client first sends data to Chunk Servers, then sends the write request to the Primary. This lets GFS make better use of network bandwidth.

As the steps above show, the control flow goes from the client to the Primary and then to the Secondary replicas through the write request.

In contrast, the **data flow is transferred through a linear data pipeline**:

The client uploads the data to the nearest replica first, following the principle of locality. That replica writes the data into its **LRU buffer** and forwards it along the chain to the nearest next Chunk Server.

After receiving the data, each replica forwards it to the next nearest replica, and this continues _**recursively**_ until every replica has received the data. This lets the system use the outgoing bandwidth of every machine in the pipeline.

#### **Appending to a File**

The append process is similar to the write process:

* Same setup as above.
* The client pushes data to every replica, then sends the request to the Primary.
* **The Primary first checks whether appending the data to the chunk would exceed the chunk size limit**.

    If it would, the Primary writes padding to the chunk until it reaches the size limit.

    It then tells the other replicas to perform the same padding operation.

    Finally, it responds to the client and tells the client to retry the append on the next chunk.
* If the data fits in the current chunk, the Primary appends the data to its own replica, obtains the offset returned by the successful append, and then tells the other replicas to **write** the data at that same offset.
* Finally, the Primary responds to the client.

If the append succeeds on only some replicas, the Primary responds to the client that the operation failed, and the client retries it. As with normal writes, some replicas may already have successfully written the data. **Retrying the append can therefore create duplicate data on those replicas**. GFS's consistency model does not guarantee that every replica is byte-for-byte identical in every such case.

**GFS only guarantees that the data is appended as an atomic whole at least once**. Therefore, when an append operation succeeds, the data must have been written at the **same offset on all replicas**, and every replica's length is at least past the end of the appended record. The next append will be assigned an offset greater than that value, or it will be assigned to a new chunk.

#### **File Snapshot**

GFS also provides a snapshot operation that **creates a copy of a specified file or directory**.

The snapshot operation is implemented with the idea of **copy-on-write**:

* When the **Master receives a snapshot request, it first revokes the leases for the relevant chunks**.

    This ensures that later client writes to those chunks must contact the Master to learn the Primary's location.

    The Master can use this opportunity to create new chunks.
* After the chunk leases are revoked or expire, the Master writes the operation to its WAL and then **copies the namespace metadata**. The **new records point to the original chunks**.
* When a client later tries to write to one of these chunks, the Master notices that the chunk's reference count is greater than 1.

    The Master generates a new **handle** for the new chunk that is about to be created.

    It then tells all **Chunk Servers that hold the original chunk** to make a local copy, **apply the new handle**, and return the result to the client.

#### **Replica Management**

To further improve the efficiency of a GFS cluster, the Master uses several strategies when choosing **where replicas should be placed**.

The Master's replica-placement policy has two main goals:

* **Maximize data availability**.
* **Maximize network bandwidth utilization**.

For this reason, replicas should not only be stored on different machines, but also on different racks.

If an entire rack becomes unavailable, the data can still survive.

This also lets different clients reading the same chunk use outgoing bandwidth from different racks.

The downside is that writes must transfer data across racks. The GFS designers considered this a reasonable trade-off.

Replica lifecycle transitions only have two real operations: **creation and deletion**.

First, replica creation may be caused by three events: creating a new chunk, re-replicating an existing chunk, or rebalancing replicas.

When the Master creates a new chunk, it first decides where to place the new replicas. The Master considers the following factors:

* The Master prefers to place new replicas on **Chunk Servers with lower disk utilization**.
* The Master **tries to ensure that each Chunk Server does not have too many "recent" replicas**, because creating a new chunk usually means that many writes will arrive soon. If some Chunk Servers have too many new chunk replicas, write pressure will be concentrated on those machines.
* As mentioned above, the Master prefers to place replicas on **different racks**.

When the number of replicas for a chunk falls below the user-specified threshold, the Master **re-replicates** that chunk.

This may be triggered by a Chunk Server failure, a Chunk Server reporting corrupted replica data, or a user increasing the replica-count threshold.

The Master first assigns **a priority to every chunk that needs re-replication** according to the following factors:

* How far the chunk's replica count is below the user-specified **replica threshold**.
* Whether the chunk belongs to a **non-deleted file**, which is prioritized.
* In addition, the Master **raises the priority of any chunk that is blocking user operations**.

After priorities are assigned, the Master selects the highest-priority chunk and **instructs several Chunk Servers to copy data directly from existing replicas**.

The Master's choice of target Chunk Servers uses the same placement factors described above. In addition, to reduce the impact of re-replication on users, the Master **limits** the total number of ongoing replication operations in the cluster, and each Chunk Server also **limits** the bandwidth used for copying.

Finally, the Master periodically checks the distribution of each chunk in the cluster. When necessary, it migrates some replicas to better balance disk utilization and load across nodes.

The placement strategy for a new replica is broadly the same as above. In addition, the Master must choose which existing replica to remove. In short, **the Master tends to remove replicas from Chunk Servers with high disk usage in order to balance disk utilization**.

#### **Deleting a File**

When a user deletes a file, GFS does not immediately delete the data. Instead, it removes data lazily at both the **file level and chunk level**.

First, when a user deletes a file, GFS does not directly remove the file record from the namespace. It **renames** the file to a hidden name and attaches the **deletion timestamp**.

During the Master's periodic namespace scan, it finds files that have been "deleted" for a long time, for example three days. Only then does the Master actually remove them from the namespace.

Before the file is completely removed from the namespace, a client can still read the file using its hidden renamed path, and can even rename it again to undo the deletion.

The Master maintains the mapping between files and chunks in metadata.

When a file is removed from the namespace, **the reference count of its chunks automatically decreases by 1**.

Again during the Master's periodic metadata scan, the Master finds **chunks whose reference count has reached 0** and removes their metadata from memory. During the periodic heartbeat between Chunk Servers and the Master, each Chunk Server reports the chunk replicas it holds. The Master then tells the Chunk Server which chunks no longer exist in metadata, and the Chunk Server can delete the corresponding replicas by itself.

This deletion mechanism has three main advantages:

* For a large-scale distributed system, this mechanism is more **reliable**.

    When a chunk is created, the creation may succeed on some Chunk Servers and fail on others. This means **some Chunk Servers may contain replicas unknown to the Master**.

    In addition, a request to delete a replica may fail to be delivered, and the Master would have to remember to retry it. In contrast, letting Chunk Servers delete replicas proactively solves these problems in a more uniform way.
* This deletion mechanism **merges storage reclamation with the Master's routine periodic scans**, so the operations can be processed in batches and resource consumption is reduced.

    This also lets the Master perform cleanup when the system is relatively **idle**.
* The delay between the user's delete request and actual data deletion also protects users from accidental deletion.

However, if **storage resources are scarce**, this mechanism may make storage-space tuning harder for users.

For this reason, GFS allows the client to **explicitly delete** the file again in order to truly remove it at the namespace level.

In addition, GFS lets users specify different replication and deletion policies for different namespace regions, such as instructing GFS not to replicate chunks for files under a certain directory.

### **High Availability Mechanisms**

#### Master

As mentioned earlier, the Master persists cluster metadata through an operation log. Before the log is durably written, the Master does not respond to the client request and subsequent changes do not proceed. In addition, the log is replicated to several other machines. **A log record is considered written only after it has been written to persistent storage both locally and on remote backups**.

On restart, the Master recovers its state by replaying saved operation records.

To make recovery fast, after the operation log reaches a certain size, the Master creates a **checkpoint** of its current state and **deletes** log records before that checkpoint. During restart, it recovers from the **most recent checkpoint**.

The checkpoint file is organized as a **B-tree**. Once mapped into memory, it can be searched without additional parsing to retrieve the stored namespace. This further reduces Master recovery time.

To simplify the design, only one Master is active at a time. When the Master fails, an external monitoring system detects the event and starts a new Master process elsewhere.

In addition, the cluster has **Shadow Masters** that provide **read-only** service.

They synchronize state changes from the Master, but may lag by several seconds. Their main purpose is to offload read pressure from the Master. A Shadow Master keeps its state synchronized by reading **a backup** of the Master's operation log.

Like the Master, a Shadow Master also polls Chunk Servers at startup to learn which chunk replicas they hold, and it continuously monitors their state. **In practice, after the Master fails, a Shadow Master can still provide read-only service for the GFS cluster. Its dependency on the Master is limited to replica-location update events**.

#### Chunk Server

As the slave role in the cluster, a Chunk Server is much more likely to fail than the Master. As mentioned earlier, **when a Chunk Server fails, the replica count of the chunks stored on it decreases**, and the Master discovers chunks whose replica count is below the user-specified threshold and schedules re-replication.

In addition, while a Chunk Server is down, users may continue to perform **writes**. When the Chunk Server restarts, the replica data on it may therefore be stale.

To handle this, the Master maintains **a version number for every chunk** to distinguish fresh replicas from stale replicas.

Whenever the Master grants a **chunk lease to a Chunk Server, it increases the chunk version number and informs the other up-to-date replicas to update their version numbers**. If a Chunk Server is down at that moment, the version number of its replica does not change.

When the Chunk Server restarts, it reports the chunk replicas it holds and their version numbers to the Master. If the Master finds that a replica's version number is too low, it treats that replica as nonexistent. The stale replica will then be removed during the **next replica garbage-collection process**.

In addition, when the Master returns replica-location information to a client, it also returns the current chunk version number, so the client will not read stale data.

#### **Data Integrity**

As discussed earlier, every chunk is replicated on different Chunk Servers, and users can assign different replication policies to different parts of the namespace.

To preserve data integrity, each Chunk Server uses **checksums** to detect whether its stored data has been corrupted.

After detecting corrupted data, the Chunk Server can also use another replica to **recover** the data.

First, a Chunk Server divides each chunk replica into multiple 64 KB blocks and computes a 32-bit checksum for every block.

Like the Master's metadata, these checksums are stored in the Chunk Server's memory. Before each modification, changes are protected with a write-ahead-log style mechanism to ensure availability.

When a Chunk Server receives a **read** request, it first uses checksums to verify whether the requested data is corrupted. This prevents the Chunk Server from sending corrupted data to any requester, whether the requester is a client or another Chunk Server.

After detecting corruption, the Chunk Server sends an error to the **requester** and reports the corruption event to the **Master**. After receiving the error, the requester retries the request against another Chunk Server. The Master then uses another replica to re-replicate the chunk. After the new replica is created, the Master tells the corrupted Chunk Server to delete the bad replica.

During append operations, a Chunk Server can **incrementally update the checksum of the checksum block at the end of the chunk**, or **compute a new checksum** when a new checksum block is created.

Even if the checksum block being appended to was already corrupted, the incrementally updated checksum will still fail to match the actual data, so the corruption can still be detected on the next read. During write operations, the Chunk Server must read and verify the checksum blocks containing the start and end of the write range, perform the write, and then recompute the checksums.

In addition, when idle, a Chunk Server periodically scans and verifies the data of inactive chunk replicas. This ensures that corruption can be detected even for chunk replicas that are rarely read, and also prevents corrupted replicas from causing the Master to believe that a chunk has enough valid replicas.

### **Appendix**

#### **Node Caching**

In GFS, **clients and Chunk Servers do not cache file data**.

* For clients, most applications **sequentially read large files**, so caching file data has little value. However, clients do **cache GFS metadata to reduce communication with the Master**.
* For Chunk Servers, caching file data is unnecessary because the data is already stored on local disk. In addition, **the Linux kernel buffer cache already keeps frequently accessed disk data in memory**.

#### **Chunk Size**

For GFS, chunk size is an important parameter. GFS chooses 64 MB as its chunk size.

A large chunk size brings several benefits:

* It reduces the frequency of client-Master communication.
* It increases the probability that a client's operations fall on the same chunk.
* It reduces the amount of metadata the Master must store.

However, large chunks can make small files consume extra storage space. A typical small file usually occupies only one chunk, and those chunks can easily become **load hot spots**.

But as assumed in the original requirements, such files were not common in Google's workload, and this problem did not really appear at Google. Even if it did appear, the load could be balanced by increasing the replica count for those files.

#### **Fast Component Recovery**

GFS components are designed with a strong emphasis on **fast state recovery**, and they can usually start within a few seconds.

With this guarantee, GFS components do not distinguish normal shutdown from abnormal exit in an important way. To shut down a component, it is acceptable to directly use `kill -9`.

### FAQ

The MIT 6.824 course material provides a FAQ about GFS. I translate the important parts here.

> Q: Why is atomic record append at-least-once rather than exactly-once?

Making append exactly-once is not easy, because the Primary would need to keep state to detect duplicate data. That state would also need to be replicated to other servers so that it is not lost when the Primary fails. In Lab 3, you will implement exactly-once behavior, but using a more complex protocol, Raft.

> Q: How does an application know which parts of a chunk are padding or duplicate data?

To detect padding, an application can put a magic number before every valid record.

It can also use checksums to validate data.

An application can detect duplicate data by adding a unique ID to every record. When reading data, the application can use IDs it has already read to filter out duplicates. GFS itself provides library support for these typical use cases.

> Q: Since **atomic record append writes data at an unpredictable offset in the file**, how does a client find its data?

Append operations, and GFS itself, are mainly intended for applications that read complete files.

These applications read all records, so they do not need to know the position of a record in advance. For example, a file might contain all link URLs collected by several parallel web crawlers. The offset of each URL in the file is not important. The application only wants to read all URLs.

> Q: If an application uses the standard POSIX file API, does it need to be modified to use GFS?

Yes. GFS was not designed for existing applications. It was mainly designed for newly developed applications, such as MapReduce programs.

> Q: How does GFS determine which replica is nearest?

The paper mentions that GFS determines distance **based on the IP address of the server that stores the replica**.

In 2003, Google's IP address allocation should have ensured that if two servers were close in the IP address space, they were also close in the data center.

> Q: Is Google still using GFS?

Google still uses GFS, and it is used as the backend for other storage systems such as Bigtable. As workloads grew and technology changed, GFS's design has certainly been adjusted significantly over the years, but I do not know the details. HDFS is a public imitation of the GFS design, and many companies use it.

> Q: Does the Master become a performance bottleneck?

It is possible. The GFS designers made many efforts to avoid this problem. For example, the Master keeps its state in memory to respond quickly. According to the experimental results, for large-file reads, which are the main workload GFS targets, the Master is not the bottleneck. For small-file operations and directory operations, Master performance is also sufficient, see Section 6.2.4.

> Q: How reasonable is GFS's choice to sacrifice correctness for performance and simplicity?

This is an old question in distributed systems. Strong consistency usually requires more complex protocols and more communication between machines, as we will see in later courses. By taking advantage of the fact that certain applications can tolerate relaxed consistency, people can design systems with good performance and sufficient consistency. For example, GFS is specially optimized for MapReduce applications. These applications need efficient reads of large files and can tolerate holes, duplicate records, or inconsistent read results. On the other hand, GFS is not suitable for storing bank-account balances.

> Q: What happens if the Master fails?

A GFS cluster has replica Masters that hold complete backups of the Master's state. Through a mechanism not described in the paper, GFS switches to one of these replicas when the Master fails, see Section 5.1.3. This may require a human administrator to designate a new Master. In any case, we can be sure that the cluster has a latent single point of failure that could theoretically prevent automatic recovery from Master failure. Later in the course, we will learn how to use Raft to implement a fault-tolerant Master.

#### **Question**

Besides the FAQ, the course also asks students to answer one question after reading the GFS paper:

> Describe a sequence of events that result in a client reading stale data from the Google File System
>
> Describe an event sequence in which a client reads stale data from Google File System.

By checking the paper, it is not hard to find two answers. One is stale file data caused by a Chunk Server that failed and restarted, combined with cached chunk-location data on the client, see Sections 4.5 and 2.7.1. The other is stale file metadata read by a Shadow Master, see Section 5.1.3. These are two situations where a client may read stale data even when all write operations succeed. If a write operation fails, the data enters an **undefined** state, and naturally the client may also read stale or invalid data.

### **Conclusion**

This note does not summarize Chapters 6 and 7 of the paper.

Chapter 6 contains the evaluation results for GFS. Due to space limits, I did not include them here. Readers interested in how Google measured GFS performance can refer to that chapter.

Chapter 7 describes some pitfalls Google encountered while developing GFS, mainly related to Linux bugs. I did not include that part because those bugs mostly involved Linux 2.2 and 2.4, which are no longer timely today. Those bugs were also likely fixed by the GFS developers and submitted to later Linux versions.

From a content perspective, reading the GFS paper is a useful case study of **the tension between high performance and strong consistency**.

When facing strong consistency, GFS chose higher throughput and a simpler architecture.

The tension between high performance and strong consistency is a long-standing topic in distributed systems because they are usually difficult to achieve at the same time.

In addition, to achieve ideal consistency, a system must also handle challenges caused by concurrent operations, machine failures, network partitions, and similar problems.

Conceptually, consistency refers to a correct state. A system may have many different correct states, and these are often collectively called the system's **consistency model**. We will continue to see this concept in later papers.

The Google File System paper later inspired **HDFS**, which is still one of the most important open-source distributed file-system solutions. Google MapReduce and Google File System can be seen as foundational papers of the big-data era. Together with Google Bigtable, they are often called Google's three classic infrastructure papers. These papers are still very worth studying.
