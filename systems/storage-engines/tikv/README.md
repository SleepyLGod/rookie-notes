---
description: A more advanced RocksDB-based storage system.
---

# 🤩 TiKV

TiKV is a distributed transactional key-value database. It provides distributed transaction interfaces that satisfy ACID constraints, and it uses the [Raft protocol](https://raft.github.io/raft.pdf) to guarantee replicated-data consistency and high availability. As the storage layer of TiDB, TiKV provides persistence and read/write services for data written through TiDB, and it also stores TiDB statistics data.

#### Advantages

* Geo-replication: uses Raft and Placement Driver to replicate data across locations and protect data safety.
* Horizontal scalability: the system can be expanded by directly adding nodes.
* Consistent distributed transactions: uses an optimized two-phase commit protocol based on Google Percolator to support distributed transactions. A transaction can be started with `begin`, committed with `commit`, or rolled back with `rollback`.
* Coprocessor for distributed computation: like HBase, TiKV supports a coprocessor framework that lets users execute computation directly inside TiKV.
* Deep integration with TiDB: TiKV acts as TiDB's backend storage engine and provides a distributed relational database storage layer.

#### RocksDB in TiKV (excerpted from the official documentation)

RocksDB is a persistent key-value store for fast storage environments. Some highlights of RocksDB are:

1. RocksDB uses a log-structured database engine written entirely in C++ for the best performance. Keys and values are arbitrary byte streams.
2. RocksDB is optimized for fast, low-latency storage such as flash drives and high-speed disk drives. RocksDB fully exploits the high read/write rates provided by flash or RAM.
3. RocksDB can adapt to different workloads.
4. From database storage engines such as MyRocks, to application data caches, to embedded workloads, RocksDB can serve many data needs.
5. RocksDB provides basic operations such as opening and closing a database, as well as reads, writes, and more advanced operations such as merge and compaction filters.

TiKV uses RocksDB because RocksDB is mature and high-performance. In this section, we explore how TiKV uses RocksDB.

The following subsections highlight some special features used by TiKV.

### [Prefix Bloom Filter](https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter)

A Bloom Filter is a useful data structure that uses very few resources while providing significant benefit. This section does not explain the entire algorithm. If you are not familiar with Bloom Filters, you can think of one as a black box for a dataset: without actually searching the dataset, it can tell you whether a key _may_ exist or _definitely_ does not exist. Sometimes a Bloom Filter returns a false positive, although this happens rarely when configured properly.

TiKV uses Bloom Filters and a variant called **Prefix Bloom Filter (PBF)**. A PBF does not tell you whether a whole key exists in the dataset; instead, it tells you whether another key with the same prefix may exist. Because a PBF stores only unique prefixes rather than all unique full keys, it can save memory, with the tradeoff of a higher false-positive rate.

TiKV supports MVCC, which means the same row stored in RocksDB can have multiple versions. All versions of the same row share the same prefix, the row key, but have different timestamps as suffixes. When we want to read a row, we usually do not know the exact version to read; instead, we want to read the latest version at a specific timestamp. This is where PBF is useful. PBF can filter out data that cannot possibly contain keys with the same prefix as the row key we provide. Then we only need to search data that may contain different versions of that row key and find the specific version we need.

### Table Properties

RocksDB allows users to register table-property collectors. When RocksDB builds an SST file, it passes sorted key-value pairs one by one to each collector's callback, so the collector can gather whatever information it needs. When the SST file is completed, the collected properties are stored in the SST file.

TiKV uses this feature to optimize two functions.

The first is split checking. Split checking is the worker that checks whether a region is large enough to split. In the original implementation, TiKV had to scan all data in a region to compute its size, which consumed resources. With `TableProperties`, TiKV records the size of small sub-ranges in each SST file, so it can compute an approximate region size from table properties without scanning any data.

The second is MVCC garbage collection (GC). GC is the process of cleaning obsolete versions of each row, namely versions older than the configured lifetime. If we do not know whether a region contains obsolete versions, we have to check all regions periodically. To skip unnecessary GC, TiKV records some MVCC statistics in each SST file, such as row count and version count. Therefore, before checking each region row by row, TiKV first checks table properties to decide whether GC is necessary for that region.

### Compaction

Sometimes, certain regions may contain many tombstone entries because of GC or other delete operations. Tombstone entries are bad for scan performance and waste disk space.

Therefore, using <mark style="color:purple;">**`TableProperties`**</mark>, TiKV can periodically check each region to see whether it contains many tombstones. If it does, TiKV manually compacts the region range to remove tombstones and release disk space.

TiKV also uses **`CompactRange`** to recover RocksDB from certain errors, such as table-property incompatibilities across different TiKV versions.

### [Event Listener](https://github.com/facebook/rocksdb/wiki/EventListener)

`EventListener` lets TiKV listen to special events such as flush, compaction, or write-stall condition changes. When a specific event is triggered or completed, RocksDB calls the callback and provides information about that event.

TiKV observes region-size changes by listening to compaction events. As described above, TiKV computes the approximate size of each region from table properties. The approximate size is recorded in memory, so if nothing changes, TiKV does not need to recompute it repeatedly. However, during compaction, some entries are removed, so the approximate size of some regions should be updated. This is why TiKV listens to compaction events and recomputes the approximate size of certain regions when necessary.

### [Ingest External File](https://github.com/facebook/rocksdb/wiki/Creating-and-Ingesting-SST-files)

RocksDB allows users to generate an SST file externally and then ingest that file directly into RocksDB. This feature can save a large amount of I/O, because RocksDB is smart enough to ingest the file into a lower level when possible. This reduces write amplification because the ingested file does not need to be compacted repeatedly.

TiKV uses this feature to handle Raft snapshots. For example, when adding a replica to a new server, TiKV can first generate a snapshot file from another server and then send the file to the new server. The new server can then ingest the file directly into its RocksDB instance, saving a large amount of work.

TiKV also uses this feature to import massive datasets. Some tools can generate sorted SST files from different data sources and then ingest those files into different TiKV servers. This is much faster than writing key-value pairs into a TiKV cluster in the usual way.

### DeleteFilesInRange

Previously, TiKV used a straightforward method to delete data in a range: scan all keys in the range and delete them one by one. However, disk space is not released until the tombstones are compacted. Worse, because new tombstones are written, disk-space usage may temporarily increase.

Over time, users store more and more data in TiKV until disk space becomes insufficient. Then users may try to delete some tables or add more storage, expecting disk-space usage to drop quickly. With the previous method, TiKV did not meet that expectation. TiKV first tried to solve this problem with RocksDB's `DeleteRange` feature. However, `DeleteRange` turned out to be unstable and did not release disk space quickly enough.

A faster way to release disk space is to delete some files directly, which led TiKV to use `DeleteFilesInRange`. However, this feature is not perfect and is quite dangerous because it breaks snapshot consistency. If you obtain a snapshot from RocksDB, use `DeleteFilesInRange` to delete some files, and then try to read that data, some data will be missing. Therefore, this feature must be used carefully.

TiKV uses `DeleteFilesInRange` to destroy tombstone regions and tables discarded by GC. Both cases have the same precondition: the deleted range must no longer be accessible.
