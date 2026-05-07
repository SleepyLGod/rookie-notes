---
description: Architecture and core mechanisms.
---

# 😚 General

## **Basic**

Start with the [**official RocksDB blog**](http://rocksdb.org/blog/).

The following diagram shows the main components and the flow of read, write, and compaction operations:

![Main RocksDB components and read, write, and compaction flow](https://s2.loli.net/2022/07/25/7gQGbrYaPyt2q3Z.png)

Some scattered notes:

#### **memtable**:

* To preserve key ordering and keep insertion and lookup efficient at `O(log n)`, the default MemTable is implemented with a `SkipList`. RocksDB can also use `HashLinkList`, `HashSkipList`, or `Vector` to accelerate certain query patterns:
  *   **`SkipList MemTable`**

      Provides generally good performance for reads, writes, random access, and sequential scans. It also supports useful features that other MemTable implementations may not support, such as concurrent insertions and insertions with `Hint`.
  *   **`HashSkipList MemTable`**

      `HashSkipList` organizes data in a hash table, where each hash bucket is an ordered `SkipList`. The hash key is the prefix key extracted from the original key by `Options.prefix_extractor`. It is mainly used to reduce the number of comparisons during lookup.

      It is usually used together with the `PlainTable SST` format when data is stored in RAMFS.

      The major limitation of hash-based MemTables is that scanning across multiple prefixes requires copying and sorting, which is very slow and has high memory cost.
* WAL is used for data recovery after failures and can be disabled.
* In addition to data blocks, each SSTable contains an **index block used for binary search**.
* Scenarios that trigger a MemTable **flush to disk**:
  * The MemTable size exceeds `ColumnFamilyOptions::write_buffer_size` after a write.
  * The total MemTable usage across all column families exceeds `DBOptions::db_write_buffer_size`, or `write_buffer_manager` sends a flush signal. In this case, the largest MemTable is flushed.
  * The WAL file size exceeds `DBOptions::max_total_wal_size`.

#### **Block Cache**:

*   RocksDB caches data in memory for reads:

    Recently and frequently accessed data is stored in Block Cache.

    Next, RocksDB checks MemTables in reverse write-time order.

    Then it searches SST files from Level 0 onward on disk.

    For a target key, RocksDB first checks whether it falls between an SST's `min_key` and `max_key`.&#x20;

    A Bloom filter then checks whether the key may exist in the SST. If the filter says the key is not present, RocksDB moves to the next SST file. If the key may be in that SST, RocksDB uses binary search to locate it.
* **One Cache object can be shared by multiple RocksDB instances in the same process**, so users can control the overall cache capacity.
*   Block Cache stores **uncompressed blocks**.

    Users can optionally configure a secondary block cache for compressed blocks.

    Reads first try to fetch the data block from the uncompressed block cache, then from the compressed block cache.

    When `Direct-IO` is used, the compressed block cache can replace the OS page cache.
* RocksDB has two block-cache implementations: `LRUCache` and `ClockCache`. Both cache types use **sharding** to reduce lock contention. Capacity is evenly distributed across shards, and shards do not share capacity. By default, each cache can be split into at most 64 shards, and each shard has at least 512 KB of capacity.
  *   `LRUCache`: the default cache implementation.

      It uses an LRU-based cache with an 8 MB capacity.

      Each cache shard maintains its own **LRU list** and its own **hash table** for lookup. **Synchronization is implemented through a mutex per shard, so both lookup and insertion need to lock the shard**.

      In rare cases, if blocks are being read or iterated and the total size of pinned blocks exceeds the limit, the cache size may exceed its configured capacity.

      If the host does not have enough memory, this can cause an unexpected OOM and crash the database.
  *   `ClockCache`: ClockCache implements the **CLOCK algorithm**.

      Each ClockCache shard maintains a circular list of cache entries.

      A clock hand runs over the circular list and looks for unpinned entries to evict. If an entry has been used since the last scan, it receives a second chance to stay in the cache.

      ClockCache is not stable yet and is not recommended.

![Query operation](<../../../.gitbook/assets/image (1) (2).png>)

#### **Write Buffer Manager**:

The Write Buffer Manager controls total MemTable memory usage across **multiple column families or multiple database instances**.

Usage: users create a `write buffer manager` object and pass it to the column families or database instances whose memory usage should be controlled.

There are two limiting modes:

1. **Limit total MemTable memory usage**

Either of the following conditions triggers a **flush on the instance's column family**:

* Active MemTables use **more than 90% of the threshold**.
* Total memory exceeds the limit and active MemTables also use more than 50% of the threshold.

2. **Charge MemTable memory usage to Block Cache**

In most cases, the blocks actually used in Block Cache are **much smaller** than the cache's configured capacity. When users enable this feature, **the Block Cache capacity covers the combined memory usage of both Block Cache and MemTables**.

**If `cache_index_and_filter_blocks` is also enabled, then RocksDB's three major memory areas, `index and filter cache`, `memtables`, and `block cache`, are all accounted under Block Cache.**

#### **SSTable**:

Default table format: `BlockBaseTable`.

The concrete structure is not very different from LevelDB:

* DataBlock: a **sequence of key-value pairs** ordered according to the comparator and divided into a series of data blocks. These blocks are laid out one after another at the beginning of the file, and each data block can optionally be compressed.
*   MetaBlock: placed after the data blocks. MetaBlocks **include** the filter block, index block, compression dictionary block, range deletion block, and properties block.

    More specifically:

    *   <mark style="color:purple;">**filter block**</mark>: implemented with a `bloom filter`.

        Global filter, or **Full Filter**: the entire SST file has only one filter block.

        **Partitioned Filter**: the Full Filter is split into multiple sub-filter blocks, with an index block above them that maps keys to the corresponding sub-filter block.
    *   <mark style="color:purple;">**index block**</mark>: used to find the data block that contains a given key. It is a **binary-search based** data structure.

        A file may contain one index block or a set of [**partitioned index blocks**](https://github.com/facebook/rocksdb/wiki/Partitioned-Index-Filters), depending on the configuration.

        Therefore, there are two index modes: **global index** and **partitioned index**.
    * <mark style="color:purple;">**Compression Dictionary Block**</mark>: contains the dictionary used to prepare the compression library before compressing or decompressing each block.
    *   <mark style="color:purple;">**range deletion block**</mark>: contains deletion ranges over **keys and sequence numbers** in the file. When a read request reaches an SST, RocksDB can check the relevant range inside the SST to determine whether the key falls within a `DeleteRange`; if it does, RocksDB can return `NotFound` directly.

        MemTables also contain an area that implements the same function.

        During compaction or flush, obsolete tombstone data is removed.
    *   <mark style="color:purple;">**properties block**</mark>: each property is a key-value pair.

        `data size`: total size of data blocks.

        `index size`: total size of index blocks.

        `filter size`: total size of filter blocks.

        `raw key size`: total raw size of all keys.

        `raw value size`: total raw size of all values.

        `number of entries`

        `number of data blocks`
* MetaIndexBlock: the meta-index block contains a **mapping table** pointing to **each meta block**. The key is the meta-block name, and the value is a pointer to that meta block. The pointer locates the block through offset and size.
* Footer: the fixed-size footer is stored at the end of the file. It includes a pointer to the **metaindex block**, a pointer to the **index block** in the meta block area, and a **magic number**.

#### Column Family:

![CF column family](<../../../.gitbook/assets/image (4) (2).png>)

Key-value pairs can be assigned to different column families according to different attributes. This allows the data stored in certain MemTables and SST files to have the same type, which can greatly improve read/write efficiency and data compression ratio.

When data is written, the target column family, such as CF1, CF2, or `default`, determines which logical partition receives the data.

Both **MemTables and SST** files are separated by column family, but the **WAL** is not separated by column family.
