# LSM Tree

> `HBase`, `LevelDB`, `RocksDB`, and many other `NoSQL` storage systems use LSM-trees.
>
> The core goal is to use **sequential writes** to **improve write performance**.
>
> Insert, delete, query, and update operations are first recorded in memory. After a certain amount of data accumulates, they are written to disk in batches. When data is updated, the system directly appends an update record, which is a sequential write, rather than modifying the previous key in place. By contrast, updating data in a B+ tree usually modifies the value at the original data location.
>
> Read performance is lower: the system trades read performance for write performance through multiple layers, including memory and files.
>
> `sstable` is abbreviated as `SST` below.

## **Core Idea**

![img](https://s2.loli.net/2022/07/06/labHV5Nc8FJqeZo.jpg)

### MemTable

The MemTable is an in-memory data structure, with the exact implementation chosen by the system. It keeps recently updated data organized by key order.

It is not reliable storage by itself, so it needs a `WAL`, or write-ahead log, to provide durability. When a write is executed, the system writes both the `memtable` and the `WAL` first.

### Immutable MemTable

`Memtable` -> `Immutable MemTable` -> `SSTable`

The immutable MemTable is an intermediate state. New writes are handled by a new `memtable`, so flushing the old one does not block data updates.

After a `memtable` becomes full, it is automatically converted into an `immutable` memtable and then `flush`ed to disk, forming an `L0` `sstable` file.

### Sorted String Table

An SSTable is an **ordered** collection of key-value pairs. It is the on-disk data structure used by the tree.

To speed up reads, SSTables build indexes over `key`s and use Bloom filters to accelerate key lookup.

During a **read operation**, the system first reads in-memory data. According to the principle of locality, recently written data is likely to be read soon. The lookup order is `memtable` -> `immutable memtable` -> `block cache`. If memory does not hit, the system scans `L0` `sstable`s. If the key is still not found, the system uses **binary search** in `L1` and higher-level `sstable`s to locate the corresponding key.

As more `sstable`s are written, the system opens more files, and multiple updates or deletions may accumulate for the same `key`. Since an `sstable` is immutable, the system needs `compaction` to reduce the number of files and clean up invalid data. Compaction merges multiple `sstable`s whose key ranges overlap. I do not think there is a perfect Chinese translation for "compaction": translating it as "compression" or "merge" only captures part of the meaning.

After a `MemTable` reaches a certain size and is flushed to persistent storage as an `SSTable`, different `SSTable`s may contain records for the same `Key`. Of course, **the newest record is the correct one**. This design greatly improves write performance, but it also introduces several problems:

* Redundant storage: only the newest value of a key is useful. The system needs compaction to merge multiple SSTables and remove redundant records.
* Reads must search from newest to oldest. In the worst case, the system may need to check the entire SSTable set. Indexes and Bloom filters are used to improve lookup speed.

## **Compaction Strategy**

### **Three Amplification Concepts**

* Read amplification: the actual amount of data read is greater than the amount of useful data, for example checking the memtable and then SSTables.
* Write amplification: the actual amount of data written is greater than the logical amount of user data, for example when writes trigger compaction.
* Space amplification: actual disk usage is greater than the logical data size, for example because of redundant versions.

### **Two Strategies**

#### Size-Tiered Strategy

![](https://s2.loli.net/2022/07/24/NkClzSVE6OpAfeR.jpg)

SSTables in the same level have similar sizes. Each level limits the number of SSTables to `N`. Once the number reaches `N`, compaction is triggered and the files are merged into a larger SSTable in the next level.

The advantage is that the design is simple and easy to implement. The number of SST files is small, so locating the target file is fast. However, a single SST may become very large. At higher levels, it is common to see SST files of hundreds of GB or even TB.

Space amplification can be serious. Within the same level, each key may have multiple versions until compaction for that level removes the redundancy.

However, if there are many duplicate keys, even if compaction removes space amplification within one level, duplicate key data can still exist in lower levels. Redundancy remains. Only a manually triggered full compaction can fully remove space amplification, but full compaction is extremely expensive.

#### Leveled

&#x20;

![](https://s2.loli.net/2022/07/24/wQA7VhStxNeR8Bl.jpg)

![](https://s2.loli.net/2022/07/24/xvNIaosr56mVBAt.jpg)

Leveled compaction organizes data into levels, with upper levels above lower levels. Each level limits its total file size.

For data in `L1` and above, the large SST used in `size-tiered compaction` is split into a sequence of smaller SSTs whose key ranges do not overlap. Such a sequence is called a **run**. `L0` consists of new SSTs flushed from the `memtable`, and key ranges of SSTs in this level **may overlap**. Its file-count threshold is controlled separately. Starting from `L1`, **each level contains exactly one run**, and the data-size threshold of each run grows exponentially.

Each level is split into similarly sized SSTables. The data is globally ordered, and a key does not have redundant records within the same level.

The merge strategy differs from the previous strategy:

* If the total size of `L1` exceeds the level limit, the system selects at least one file from `L1`, merges it with overlapping files in `L2`, writes the output into `L2`, and deletes the related data from `L1`.
* The same process repeats for lower levels.
* Compactions from unrelated levels can run concurrently.

Compared with size-tiered compaction, leveled compaction avoids duplicate keys within each level. Even in the worst case, where every level except the bottom level contains duplicate keys, the proportion of redundant data remains small, so space amplification is reduced.

However, write amplification becomes more prominent. When an SST in level `Ln` is merged into level `Ln+1`, it may overlap with many files, so the data is rewritten more times. In the worst case, an SSTable in level `N` has a very wide key range and overlaps all keys in level `N+1`, so the compaction writes a large amount of data.

## RocksDB Compaction Strategy

> [_**Reference slides**_](https://www.slideshare.net/FlinkForward/flink-forward-berlin-2018-stefan-richter-tuning-flink-for-robustness-and-performance)
>
> The write cache in `RocksDB`, which is the lowest level of the `LSM` tree, is called the `memtable`. It corresponds to `MemStore` in `HBase`.
>
> The read cache is called the `block cache`, the same name as the corresponding component in `HBase`.

RocksDB supports multiple compaction styles. The default and most common one is **leveled compaction**. Under leveled compaction, `L0` is special: it is produced directly by memtable flushes, and file key ranges may overlap. `L1` and higher levels are split by key range, and files in the same level usually do not overlap.

When the number of files in `L0` reaches the `level0_file_num_compaction_trigger` threshold, RocksDB triggers `L0 -> L1` compaction. Since `L0` files usually overlap with each other, this step typically needs to select multiple `L0` files and merge them with the overlapping range in `L1`.

**>= L1**

In a leveled compaction strategy, every level has a data-size threshold. In `RocksDB`, how this threshold is determined depends on two cases.

* **Parameter `level_compaction_dynamic_level_bytes` = false**

    > The threshold for `L1` is determined by `max_bytes_for_level_base`, in bytes.
    >
    > The remaining levels are derived recursively:
    >
    > **target\_size(Lk+1) = target\_size(Lk) \* max\_bytes\_for\_level\_multiplier \* max\_bytes\_for\_level\_multiplier\_addition\[k]**
    >
    > `max_bytes_for_level_multiplier` is the fixed multiplier factor, and `max_bytes_for_level_multiplier_additional[k]` is the variable multiplier factor for level `k`.
* **Parameter `level_compaction_dynamic_level_bytes` = true**

    > The highest level has **no explicit threshold limit**, so `target_size(Ln)` is the actual size of level `Ln`.
    >
    > The thresholds of lower levels are derived **backward**:
    >
    > **target\_size(Lk-1) = target\_size(Lk) / max\_bytes\_for\_level\_multiplier**
    >
    > The role of `max_bytes_for_level_multiplier` changes from a multiplication factor to a division factor. In particular, if **target\_size(Lk) < max\_bytes\_for\_level\_base / max\_bytes\_for\_level\_multiplier**, then this level **and all lower levels** will no longer store data.
    >
    >

![](https://s2.loli.net/2022/07/24/LGnD7HNgYtyZU8M.webp)

**Universal compaction**

`universal compaction` is not a mechanism specifically used for `L0` inside leveled compaction. It is a separate RocksDB compaction style. It belongs to the tiered or size-tiered family, and its usual goal is to reduce write amplification at the cost of higher read amplification and space amplification.

When universal compaction is used, SST files are organized as sorted runs divided by time range, and compaction only happens between runs in adjacent time ranges. RocksDB checks the following conditions:

* **Space amplification ratio**:

    > Suppose the existing SST files in `L0` are `(R1, R1, R2, ..., Rn)`, where `R1` is the newest SST and `Rn` is the older SST.
    >
    > **Space amplification ratio = total size of L0 files / size of Rn**
    >
    > If the ratio is greater than `max_size_amplification_percent / 100`, RocksDB merges all SST files in `L0`.
* **Adjacent file size ratio**:

    > The parameter `size_ratio` controls the threshold for the size ratio between adjacent files.
    >
    > ```go
    > if (size(R2) / size(R1)) < 1 + size_ratio / 100 
    > 	then compact R1 with R2
    > if (size(R3) / size({R1, R2})) < 1 + size_ratio / 100
    > 	then compact {R1, R2} with R3
    > ...
    > ```
    >
    > This continues until the condition no longer holds.

If neither of the two conditions triggers compaction, the strategy linearly merges files starting from `R1` until the number of files in `L0` becomes smaller than the `level0_file_num_compaction_trigger` threshold.
