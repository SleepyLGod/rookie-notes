---
description: Notes on LevelDB architecture, public interfaces, and implementation details.
---

# 🤩 LevelDB

### **Overall Architecture of LevelDB**

![Overall architecture of LevelDB](https://pic2.zhimg.com/v2-796529d39d931069e82629e73eefa8d1\_b.jpg)

:thumbsup: The diagram gives a concise overview of LevelDB's architecture.

1. <mark style="color:purple;">**MemTable**</mark>: an in-memory data structure implemented as a SkipList. It serves user read and write requests, and new writes are first inserted here.
2. <mark style="color:purple;">**Immutable MemTable**</mark>: once a MemTable reaches the configured size threshold, it is converted into an immutable MemTable. It accepts reads but no further writes, and a background thread flushes it to disk. This process is called minor compaction.
3. <mark style="color:purple;">**Log**</mark>: before data is written into the MemTable, LevelDB first writes it to a log to avoid losing MemTable data after a crash. One log file corresponds to one MemTable.
4. <mark style="color:purple;">**SSTable**</mark>: Sorted String Table. LevelDB organizes SSTables into levels from level-0 to level-n. Each level contains multiple SSTables, and records inside each file are sorted. Except for level-0, SSTables within the same level have non-overlapping key ranges.
5. <mark style="color:purple;">**Manifest**</mark>: the Manifest records SSTable metadata across levels, including which SSTables belong to each level, each file's size, and each SSTable's minimum and maximum keys.
6. <mark style="color:purple;">**Current**</mark>: after restart, multiple Manifest files may exist. The Current file records the name of the Manifest currently in use.
7. <mark style="color:purple;">**TableCache**</mark>: caches SSTable file descriptors, indexes, and filters.
8. <mark style="color:purple;">**BlockCache**</mark>: SSTable data is organized into blocks. BlockCache caches the decompressed data of these blocks.

**For the official overview, see:** [**doc/index.md**](https://github.com/google/leveldb/blob/master/doc/index.md) **.**&#x20;

**For the implementation overview, see:** [**doc/impl.md**](https://github.com/google/leveldb/blob/master/doc/impl.md) **.**

Public interfaces are defined in <mark style="color:purple;">**`include/leveldb/*.h`**</mark>. Callers should not include any other headers from the package, because internal APIs may change without warning.

<mark style="background-color:green;">**Header Guide**</mark><mark style="background-color:green;"><mark style="color:blue;">**:**<mark style="color:blue;"></mark>

<mark style="color:blue;">`include/leveldb/db.h`</mark>: the main DB interface.

<mark style="color:blue;">`include/leveldb/options.h`</mark>: controls database-wide behavior and read/write operation behavior.

<mark style="color:blue;">`include/leveldb/comparator.h`</mark>: the abstract interface for user-defined comparison functions. If bytewise key ordering is sufficient, the default comparator can be used; otherwise, users can implement their own comparator to customize ordering, for example to handle different character encodings.

<mark style="color:blue;">`include/leveldb/iterator.h`</mark>: the data-iteration interface. Iterators can be obtained from a DB object.

<mark style="color:blue;">`include/leveldb/write_batch.h`</mark>: the interface for atomic batches of multiple updates.

<mark style="color:blue;">`include/leveldb/slice.h`</mark>: a lightweight module that stores the address and length of a byte array.

<mark style="color:blue;">`include/leveldb/status.h`</mark>: most public interfaces return `Status`, which represents success or different categories of failure.

<mark style="color:blue;">`include/leveldb/env.h`</mark>: the abstraction over the operating-system environment. The POSIX implementation is in <mark style="color:purple;">`util/env_posix.cc`</mark>.

<mark style="color:blue;">`include/leveldb/table.h`</mark><mark style="color:blue;">:</mark> a low-level model that most clients are unlikely to use directly.&#x20;

<mark style="color:blue;">`include/leveldb/table_builder.h`</mark>: a low-level model that most clients are unlikely to use directly.&#x20;
