---
description: Adapted from Draveness's article with minor modifications.
---

# 🤓 LevelDB & BigTable

At OSDI 2006, Google published the paper [Bigtable: A Distributed Storage System for Structured Data](https://research.google/pubs/bigtable-a-distributed-storage-system-for-structured-data/). The paper describes Bigtable, a distributed storage system for managing structured data, including its data model, interfaces, and implementation.

![leveldb-logo](https://img.draveness.me/2017-08-12-leveldb-logo.png)

This article first gives a brief description of the distributed storage system described in the Bigtable paper, and then analyzes Google's open-source key-value storage database [LevelDB](https://github.com/google/leveldb). LevelDB can be understood as a **single-node version of the Bigtable-style system**. Although it does not include Bigtable's tablet management or other distributed logic, reading LevelDB source code can still deepen our understanding of Bigtable.

### Bigtable

Bigtable is a distributed storage system for managing **structured data**. It has excellent scalability and can handle petabytes of data across thousands of machines. Many Google projects, including web indexing, use Bigtable to store massive datasets.

The Bigtable paper claims that it achieves four goals:

![Goals-of-Bigtable](https://img.draveness.me/2017-08-12-Goals-of-Bigtable.jpg)

#### **Data Model**

Bigtable is similar to databases in many ways, but it exposes an interface different from traditional databases. It does not support the full relational data model. Instead, it uses a simpler data model that makes data easier to control and manage flexibly.

In implementation, <mark style="color:green;">**Bigtable is essentially a sparse, distributed, persistent, multidimensional sorted map**</mark>.

> A Bigtable is a sparse, distributed, persistent multi-dimensional sorted map.

This definition also determines that its data model is simple and easy to implement. It uses `row`, `column`, and `timestamp` as the three fields of the map key, and the value is a byte array, which can also be understood as a string.

![Bigtable-DataModel-Row-Column-Timestamp-Value](https://img.draveness.me/2017-08-12-Bigtable-DataModel-Row-Column-Timestamp-Value.jpg)

The most important field is the `row` value. Its maximum length is **64 KB**, and reads and writes under the same `row` can be considered atomic.

Because Bigtable sorts data by `row` value in lexicographic order, each range of `row` values is partitioned by Bigtable and assigned to a tablet for processing.

#### **Implementation**

This section introduces the implementation described in the Bigtable paper. It covers tablet organization, tablet management, read/write request handling, and data compaction.

**Tablet organization**

Bigtable uses a three-level structure similar to a B+ tree to store tablet location information. The first level is a single [Chubby](https://research.google/pubs/the-chubby-lock-service-for-loosely-coupled-distributed-systems/) file that stores the location of the root tablet.

![Tablet-Location-Hierarchy](https://img.draveness.me/2017-08-12-Tablet-Location-Hierarchy.jpg)

Every METADATA tablet, including the tablet on the root node, stores **the tablet location and the minimum and maximum keys in that tablet**.

**Each METADATA row stores about 1 KB of data in memory**. If each METADATA tablet is 128 MB, then the entire three-level structure can store `2^61` bytes of data.

**Tablet management**

Given the large number of tablet servers and data-partition tablets in Bigtable, how does Bigtable manage such a large amount of data? Like many distributed systems, Bigtable uses a master server to assign tablets to different server nodes.

![Master-Manage-Tablet-Servers-And-Tablets](https://img.draveness.me/2017-08-12-Master-Manage-Tablet-Servers-And-Tablets.jpg)

To reduce load on the master server, clients only use the Master to obtain tablet-server location information. They do not request the Master for every read or write; instead, they connect directly to tablet servers.

The clients themselves also cache tablet-server locations to reduce the number and frequency of communications with the Master.

**Read and write request handling**

The request-handling path shows how the components inside Bigtable cooperate, including logs, memtables, and SSTable files.

![Tablet-Serving](https://img.draveness.me/2017-08-12-Tablet-Serving.jpg)

When a client sends a write operation to a tablet server, the tablet server first appends a record to its log. After the log append succeeds, it inserts the record into the memtable.

This is the same pattern used by many modern databases: append records to the log through sequential writes, then apply writes to the database. Random writes are much more expensive than appends. If data were written directly through random writes, then the long execution time of random writes would increase the chance of data loss if a device failure occurred during the write operation.

When a tablet server receives a read operation, it performs a merged lookup across the memtable and SSTables. Because both memtables and SSTables store keys in lexicographic order, the read operation can execute quickly.

**Table compaction**

As writes proceed, the memtable grows over time. When its size exceeds a threshold, the current memtable is **frozen**, and a new memtable is created. The frozen memtable is converted into an SSTable and written into GFS. This compaction method is called _Minor Compaction_.

![Minor-Compaction](https://img.draveness.me/2017-08-12-Minor-Compaction.jpg)

Each Minor Compaction creates a new SSTable. It effectively reduces memory usage and shortens recovery time after a server process exits abnormally, because an oversized log would otherwise take longer to replay.

Since there is Minor Compaction for compacting memtable data, there must also be a corresponding Major Compaction operation.

![Major-Compaction](https://img.draveness.me/2017-08-12-Major-Compaction.jpg)

Bigtable performs _Major Compaction_ **periodically in the background**. It takes the data in memtables and some SSTables as input, **merge-sorts** their key-value entries, generates a new SSTable, and removes the old memtable and SSTables. The newly generated SSTable contains all data and metadata from the old inputs, while entries marked as deleted are permanently removed.

**Summary**

This completes the basic introduction to the Bigtable paper. This article only covers part of the paper; it does not cover details such as compression algorithms, caching, and commit-log implementation. For more information, it is strongly recommended to read the original paper [Bigtable: A Distributed Storage System for Structured Data](https://research.google/pubs/bigtable-a-distributed-storage-system-for-structured-data/).

### LevelDB

The Bigtable discussion above is preparation for understanding [LevelDB](https://github.com/google/leveldb). This does not mean the Bigtable material is unimportant. LevelDB is a single-node implementation of the key-value storage system described in the Bigtable paper. It provides an extremely fast key-value storage system, and it was developed by Bigtable authors [Jeff Dean](https://research.google.com/pubs/jeff.html) and Sanjay Ghemawat. In many ways, it closely follows the implementation ideas described in the Bigtable paper.

Because Bigtable is only described in a paper, and because its implementation depends on several closed-source Google infrastructure systems such as GFS and Chubby, it is difficult to access its source code. LevelDB provides a practical way to understand many of the ideas discussed in the paper.

#### **Overview**

As a key-value storage "warehouse", LevelDB provides a very simple set of interfaces for create, update, delete, and lookup operations:

```cpp
class DB {
 public:
  virtual Status Put(const WriteOptions& options, const Slice& key, const Slice& value) = 0;
  virtual Status Delete(const WriteOptions& options, const Slice& key) = 0;
  virtual Status Write(const WriteOptions& options, WriteBatch* updates) = 0;
  virtual Status Get(const ReadOptions& options, const Slice& key, std::string* value) = 0;
}
```

> Internally, the `Put` method eventually calls the `Write` method. It simply provides a different upper-level interface for callers.

`Get` and `Put` are the read and write interfaces LevelDB provides to upper layers. If we have a clear understanding of the read and write paths, LevelDB's implementation becomes much easier to understand.

This section starts from read and write operations, uses them to understand implementation details in the project, and explains new concepts as they appear. Through this process, it introduces several important LevelDB modules step by step.

#### **Starting from Writes**

First, look at the write method behind <mark style="color:blue;">`Get`</mark> and <mark style="color:purple;">`Put`</mark>:

```cpp
Status DB::Put(const WriteOptions& opt, const Slice& key, const Slice& value) {
  WriteBatch batch;
  batch.Put(key, value);
  return Write(opt, &batch);
}

Status DBImpl::Write(const WriteOptions& options, WriteBatch* my_batch) {
    ...
}
```

As described above, `DB::Put` wraps the input parameters into a `WriteBatch`, and then still executes `DBImpl::Write` to write data into the database. The write method `DBImpl::Write` is a complex process that contains many checks on contextual state. First, look at the overall logic of one write operation:

![LevelDB-Put](https://img.draveness.me/2017-08-12-LevelDB-Put.jpg)

At a high level, LevelDB performs three steps when executing a write operation:

1. Call **`MakeRoomForWrite`** to provide enough room for the upcoming write.
   * During this process, insufficient memtable space may freeze the current memtable, trigger <mark style="color:purple;">**Minor Compaction**</mark>, and create a new `MemTable` object.
   * When certain conditions are met, <mark style="color:purple;">**Major Compaction**</mark> may also occur to compact SSTables in the database.
2. Append one write-operation record to the **log** through <mark style="color:purple;">**`AddRecord`**</mark>.
3. After the log record is written successfully, insert it directly into the memtable through <mark style="color:purple;">**`InsertInto`**</mark>, completing the write flow.

Here, we do not show the full LevelDB implementation of <mark style="color:purple;">**`Put`**</mark>. Instead, we show a simplified version to help understand the overall write path:

```cpp
Status DBImpl::Write(const WriteOptions& options, WriteBatch* my_batch) {
  Writer w(&mutex_);
  w.batch = my_batch;

  MakeRoomForWrite(my_batch == NULL);

  uint64_t last_sequence = versions_->LastSequence();
  Writer* last_writer = &w;
  WriteBatch* updates = BuildBatchGroup(&last_writer);
  WriteBatchInternal::SetSequence(updates, last_sequence + 1);
  last_sequence += WriteBatchInternal::Count(updates);

  log_->AddRecord(WriteBatchInternal::Contents(updates));
  WriteBatchInternal::InsertInto(updates, mem_);

  versions_->SetLastSequence(last_sequence);
  return Status::OK();
}
```

**Immutable memtable**

In the write implementation `DBImpl::Put`, the write-preparation method <mark style="color:purple;">**`MakeRoomForWrite`**</mark> deserves attention:

```cpp
Status DBImpl::MakeRoomForWrite(bool force) {
  uint64_t new_log_number = versions_->NewFileNumber();
  WritableFile* lfile = NULL;
  env_->NewWritableFile(LogFileName(dbname_, new_log_number), &lfile);

  delete log_;
  delete logfile_;
  logfile_ = lfile;
  logfile_number_ = new_log_number;
  log_ = new log::Writer(lfile);
  imm_ = mem_;
  has_imm_.Release_Store(imm_);
  mem_ = new MemTable(internal_comparator_);
  mem_->Ref();
  MaybeScheduleCompaction();
  return Status::OK();
}
```

When LevelDB's memtable is filled with data and memory is nearly insufficient, LevelDB **freezes the memtable and creates a new `MemTable`** object.

![Immutable-MemTable](https://img.draveness.me/2017-08-12-Immutable-MemTable.jpg)

Unlike the structure described in the Bigtable paper, **LevelDB introduces an immutable memtable structure named `imm`. Its structure is exactly the same as the memtable, except that all data in it is immutable**.

![LevelDB-Serving](https://img.draveness.me/2017-08-12-LevelDB-Serving.jpg)

After switching to the new memtable, LevelDB may execute <mark style="color:purple;">**`MaybeScheduleCompaction`**</mark> to trigger a Minor Compaction that persists data in `imm` as an SSTable in the database.

The introduction of `imm` solves the problem where writes would otherwise be blocked during compaction because the memtable had grown too large.

With `imm`, if the memtable contains too much data, LevelDB can assign the memtable pointer directly to `imm` and create a new MemTable instance. This allows LevelDB to continue accepting external write operations without waiting for Minor Compaction to finish.

**Log record format**

As a persistent key-value database, LevelDB must have a log module to support data recovery when errors occur. To understand LevelDB deeply, the log format is unavoidable. This section does not show the implementation of <mark style="color:purple;">**`AddRecord`**</mark>, because the method mainly concatenates headers and strings.

In LevelDB, logs are stored in blocks. Each block is 32 KB long. The **fixed block length** means a log record may be stored at any position in a block. LevelDB introduces a one-byte **`RecordType`** to indicate the current record's position inside a block:

```cpp
enum RecordType {
  // Zero is reserved for preallocated files
  kZeroType = 0,
  kFullType = 1,
  // For fragments
  kFirstType = 2,
  kMiddleType = 3,
  kLastType = 4
};
```

The log record type is stored in the record header, along with the 4-byte CRC checksum, record length, and other information:

![LevelDB-log-format-and-recordtype](https://img.draveness.me/2017-08-12-LevelDB-log-format-and-recordtype.jpg)

The figure above contains 4 blocks and 6 log records. LevelDB can use **`RecordType`** to mark each complete log record or fragment of a log record, and later reconstruct the record from this information when the log needs to be used.

```cpp
virtual Status Sync() {
  Status s = SyncDirIfManifest();
  if (fflush_unlocked(file_) != 0 ||
      fdatasync(fileno(file_)) != 0) {
    s = Status::IOError(filename_, strerror(errno));
  }
  return s;
}
```

Because new log records are written sequentially, write speed is usually high. It is important to note that LevelDB's durability semantics depend on `WriteOptions.sync`. By default, `sync=false`: writes enter the log-file path, but LevelDB does not guarantee that every write executes `fdatasync`, so recent writes may still be lost after power failure or OS crash. Only when `sync=true` does LevelDB call the log file's `Sync()` after writing the log, providing stronger persistence. In other words, the WAL provides the infrastructure for crash recovery, but "synchronously flushing every write to disk" is not the default behavior.

**Record insertion**

When a data record is written to the log, it still cannot be queried. It becomes queryable only after being written into the memtable. This is the topic of this section. Both data insertion and data deletion add one record to the memtable.

![LevelDB-Memtable-Key-Value-Format](https://img.draveness.me/2017-08-12-LevelDB-Memtable-Key-Value-Format.jpg)

The difference between add and delete records is that they use different <mark style="color:purple;">`ValueType`</mark> markers. Inserted data is marked as <mark style="color:purple;">`kTypeValue`</mark>, and a delete operation is marked as <mark style="color:purple;">`kTypeDeletion`</mark>. In reality, both operations insert one record into the memtable.

```cpp
virtual void Put(const Slice& key, const Slice& value) {
  mem_->Add(sequence_, kTypeValue, key, value);
  sequence_++;
}
virtual void Delete(const Slice& key) {
  mem_->Add(sequence_, kTypeDeletion, key, Slice());
  sequence_++;
}
```

Both methods call the memtable's `Add` method and **insert data into the internal skiplist using the format shown above**. The inserted record contains the key, value, sequence number, and record type. These fields are concatenated and stored in the skiplist. Since we do not delete data directly from the memtable, how does LevelDB guarantee that each query retrieves the latest data?

First, the skiplist uses a custom <mark style="color:purple;">**`comparator`**</mark>:

```cpp
int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const {
  int r = user_comparator_->Compare(ExtractUserKey(akey), ExtractUserKey(bkey));
  if (r == 0) {
    const uint64_t anum = DecodeFixed64(akey.data() + akey.size() - 8);
    const uint64_t bnum = DecodeFixed64(bkey.data() + bkey.size() - 8);
    if (anum > bnum) {
      r = -1;
    } else if (anum < bnum) {
      r = +1;
    }
  }
  return r;
}
```

> The two keys being compared may contain different amounts of data. Some contain the complete key, sequence number, and other information, while a key passed from `Get` may contain only the key length, key value, and sequence number. This does not affect extraction here, **because LevelDB extracts information only from the head of each key**. Therefore, whether the input is a complete key/value entry or a standalone key, LevelDB does not read beyond the key.

This method extracts the key and sequence number from two different keys and compares them.

The comparison process uses <mark style="color:purple;">`InternalKeyComparator`</mark>. It sorts by `user_key` and `sequence_number`: **`user_key` is sorted in increasing order, while `sequence_number` is sorted in decreasing order**. Because sequence numbers keep increasing as data is inserted, LevelDB can ensure that the first record it sees is always the newest data or the newest deletion marker.

![LevelDB-MemTable-SkipList](https://img.draveness.me/2017-08-12-LevelDB-MemTable-SkipList.jpg)

With sequence numbers, LevelDB does not need to delete historical data immediately, and it can also speed up writes and improve write performance.

#### **Data Reads**

Reading data from LevelDB is not complicated. The memtable and `imm` are more like **two levels of cache**. They are in memory and provide faster access. If LevelDB can obtain the corresponding value directly from these two structures, that value must be the newest data.

> LevelDB always writes new key-value pairs first, and removes historical data during compaction.

![LevelDB-Read-Processes](https://img.draveness.me/2017-08-12-LevelDB-Read-Processes.jpg)

Data reads are performed in the order of MemTable, Immutable MemTable, and then **SSTables at different levels**. The first two are in memory, while SSTables at different levels are persisted on disk as `*.ldb` files. **Because the database contains SSTables at different levels, it is called LevelDB.**

The simplified implementation of the read operation is shown below. The method's flow is clear, so it does not require much additional explanation:

```cpp
Status DBImpl::Get(const ReadOptions& options, const Slice& key, std::string* value) {
  LookupKey lkey(key, versions_->LastSequence());
  if (mem_->Get(lkey, value, NULL)) {
    // Done
  } else if (imm_ != NULL && imm_->Get(lkey, value, NULL)) {
    // Done
  } else {
    versions_->current()->Get(options, lkey, value, NULL);
  }

  MaybeScheduleCompaction();
  return Status::OK();
}
```

When LevelDB finds a result in the memtable or `imm`, **finding a record does not necessarily mean the current value exists. LevelDB still needs to inspect `ValueType` to determine whether the current record has been deleted**.

**Multi-level SSTables**

Only when LevelDB cannot find the corresponding data in memory does it search SSTables at multiple disk levels. This process is slightly more complicated. LevelDB searches level by level and does not skip any level. During lookup, one important **data structure, `FileMetaData`**, is involved:

![FileMetaData](https://img.draveness.me/2017-08-12-FileMetaData.jpg)

`FileMetaData` contains **all information about a file**, including the **maximum and minimum keys**, **allowed seek count**, **file reference count**, **file size**, and **file number**. Because **all `SSTable` files are stored in the same directory using a fixed naming scheme**, LevelDB can easily locate the corresponding file by its **file number**.

![LevelDB-Level0-Laye](https://img.draveness.me/2017-08-12-LevelDB-Level0-Layer.jpg)

Lookup proceeds from lower levels to higher levels. **LevelDB first searches Level 0 for the corresponding key**. Unlike other levels, **multiple SSTables in Level 0 may have overlapping key ranges**. Therefore, the read path checks all L0 files whose key ranges cover the target key, from newest to oldest. In classic LevelDB, 4 is the default L0 compaction trigger, not the maximum number of L0 files that one query may inspect. When L0 files accumulate, the actual number of inspected files may exceed 4.

![LevelDB-LevelN-Layers](https://img.draveness.me/2017-08-12-LevelDB-LevelN-Layers.jpg)

For **higher-level SSTables**, SSTables within the same level do not overlap. Therefore, during lookup, LevelDB can **use the known `smallest/largest` key information in each SSTable to quickly locate the corresponding SSTable**. It then checks whether the SSTable contains the queried key. If not, it continues to the next level until reaching **the final level `kNumLevels`** **(7 by default)** or finding the corresponding value.

**SSTable compaction**

Since LevelDB organizes data through SSTables across multiple levels, how does it merge and compact SSTables from different levels? The two compaction methods are almost the same as those described in the Bigtable paper, and LevelDB implements both.

**Both** read and write operations may call <mark style="color:purple;">**`MaybeScheduleCompaction`**</mark> during execution to attempt SSTable compaction.

When compaction conditions are met, LevelDB eventually executes <mark style="color:purple;">**`BackgroundCompaction`**</mark> in the background.

![LevelDB-BackgroundCompaction-Processes](https://img.draveness.me/2017-08-12-LevelDB-BackgroundCompaction-Processes.jpg)

There are two cases:

One is <mark style="color:purple;">**Minor Compaction**</mark>. When data in memory exceeds the maximum memtable size, the memtable is frozen as immutable `imm`, and LevelDB executes <mark style="color:purple;">**`CompactMemTable()`**</mark> to **compact the memtable**.

```cpp
void DBImpl::CompactMemTable() {
  VersionEdit edit;
  Version* base = versions_->current();
  WriteLevel0Table(imm_, &edit, base);
  versions_->LogAndApply(&edit, &mutex_);
  DeleteObsoleteFiles();
}
```

`CompactMemTable` executes <mark style="color:purple;">**`WriteLevel0Table`**</mark> to **convert the current `imm` into a Level 0 SSTable file**. Because the number of Level 0 files increases, this may continue to trigger a new Major Compaction. At that point, LevelDB needs to choose an appropriate level to compact:

```cpp
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit, Version* base) {
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();
  Iterator* iter = mem->NewIterator();
  BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);

  const Slice min_user_key = meta.smallest.user_key();
  const Slice max_user_key = meta.largest.user_key();
  int level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
  edit->AddFile(level, meta.number, meta.file_size, meta.smallest, meta.largest);
  return Status::OK();
}
```

**All modifications to current SSTable data** are recorded and managed by one unified <mark style="color:red;">**`VersionEdit`**</mark> object. This object is discussed later. If the write succeeds, LevelDB returns the file's metadata, **`FileMetaData`**. Finally, it calls **`VersionSet`**'s **`LogAndApply`** method to faithfully record all file changes and then clean up data.

If the operation is <mark style="color:purple;">**Major Compaction**</mark>, the logic is slightly more complex. However, the simplified <mark style="color:purple;">**`BackgroundCompaction`**</mark> method is still clear:

```cpp
void DBImpl::BackgroundCompaction() {
  if (imm_ != NULL) {
    CompactMemTable();
    return;
  }

  Compaction* c = versions_->PickCompaction();
  CompactionState* compact = new CompactionState(c);
  DoCompactionWork(compact);
  CleanupCompaction(compact);
  DeleteObsoleteFiles();
}
```

LevelDB finds the file information that needs compaction from the current `VersionSet` and **packages it into a `Compaction` object**. This object needs to select SSTables from **two levels**. The lower-level table is easy to choose: it selects SSTables whose **size exceeds the limit or whose seek count is too high**. After choosing one lower-level SSTable, LevelDB selects higher-level SSTables whose keys **overlap** with that SSTable. With the help of `FileMetaData`, LevelDB can quickly find all data that needs compaction.

> **Too many seeks** means that when the client calls `Get` many times, if one `Get` finds the corresponding key in an SSTable at a certain level, **this counts as one seek on the SSTable in the previous level that contains that key**. In other words, because key ranges overlap across levels, this lookup consumed extra time. After an SSTable is created, its <mark style="color:blue;">**`allowed_seeks`**</mark> is set to 100. When <mark style="color:blue;">**`allowed_seeks < 0`**</mark>, LevelDB triggers compaction of that file with the higher level to reduce the number of lookups required by future queries.

![LevelDB-Pick-Compactions](https://img.draveness.me/2017-08-12-LevelDB-Pick-Compactions.jpg)

LevelDB's <mark style="color:purple;">**`DoCompactionWork`**</mark> method uses **merge sort** to merge key-value entries from all input SSTables, and finally generates a new SSTable in the higher level, Level 2 in the figure.

![LevelDB-After-Compactions](https://img.draveness.me/2017-08-12-LevelDB-After-Compactions.jpg)

After this, the next query for values between 17 and 40 can avoid one binary search inside an SSTable and one file read, improving read performance.

**VersionSet that stores DB state**

**All LevelDB state is stored in a `VersionSet`**. A `VersionSet` contains **a group of `Version` structures**. All `Version` objects, including historical versions, are connected by a <mark style="color:red;">**doubly linked list**</mark>, but only one version is the current version.

![VersionSet-Version-And-VersionEdit](https://img.draveness.me/2017-08-12-VersionSet-Version-And-VersionEdit.jpg)

When SSTables change in LevelDB, LevelDB generates a **`VersionEdit`** structure and eventually executes **`LogAndApply`**:

```cpp
Status VersionSet::LogAndApply(VersionEdit* edit, port::Mutex* mu) {
  Version* v = new Version(this);
  Builder builder(this, current_);
  builder.Apply(edit);
  builder.SaveTo(v);

  std::string new_manifest_file;
  new_manifest_file = DescriptorFileName(dbname_, manifest_file_number_);
  env_->NewWritableFile(new_manifest_file, &descriptor_file_);

  std::string record;
  edit->EncodeTo(&record);
  descriptor_log_->AddRecord(record);
  descriptor_file_->Sync();

  SetCurrentFile(env_, dbname_, manifest_file_number_);
  AppendVersion(v);

  return Status::OK();
}
```

The main job of this method is to **create a new version object from the current version and `VersionEdit`**, **append the `Version` changes to the MANIFEST log, and update the global current-version information of the database**.

> The **MANIFEST file** records all tables in all LevelDB levels, the key range of each SSTable, and other important metadata. It is stored as a log, and all file additions and deletions are appended to this log.

**SSTable format (overview here; see [sstable-in-leveldb.md](sstable-in-leveldb.md "mention") for details)**

SSTables do not store only data. They also store **metadata, indexes**, and other information to speed up reads and writes. Although the Bigtable paper does not describe the SSTable data format, LevelDB's implementation shows that SSTables store data in the following format:

![SSTable-Format](https://img.draveness.me/2017-08-12-SSTable-Format.jpg)

When LevelDB reads an existing `.ldb` SSTable file, it first reads the `Footer` information from the file.

![SSTable-Footer](https://img.draveness.me/2017-08-12-SSTable-Footer.jpg)

```cpp
// 40B:(40==2*BlockHandle::kMaxEncodedLength)
metaindex_handle: char[p];      // Block handle for metaindex
index_handle:     char[q];      // Block handle for index
padding:          char[40-p-q]; // zeroed bytes to make fixed length
// 8B:static const uint64_t kTableMagicNumber = 0xdb4775248b80fb57ull;
magic:            fixed64;      // == 0xdb4775248b80fb57 (little-endian)
```

The entire `Footer` occupies a fixed 48 bytes in the file. From it, LevelDB can obtain the locations of the **MetaIndex block and Index block**, and then use the index to locate the value.

For the file to be self-describing, it must contain **pointers** to other file positions to indicate where each `section` begins and ends. The variable responsible for this is called <mark style="color:purple;">**BlockHandle**</mark>. It has two member variables, `offset_` and `size_`, which record the starting position and length of a data block:

```cpp
class BlockHandle {
private:
  uint64_t offset_;
  uint64_t size_;
};
```

**A `uint64` integer occupies at most 10 bytes after varint64 encoding**. A `BlockHandle` contains two `uint64` values, `size` and `offset`, so one `BlockHandle` occupies at most 20 bytes, namely <mark style="color:purple;">`BlockHandle::kMaxEncodedLength=20`</mark>. `metaindex_handle` and `index_handle` can occupy at most 40 bytes in total.

The `magic number` occupies 8 bytes. It is a fixed value used to **verify during reads that the value matches what was written**. If it does not match, the file is not an SSTable file, producing a bad magic number error. `padding` is used to **pad** the handle area to 40 bytes.

From the `footer` of an SSTable file, LevelDB can decode the `BlockHandle` of the `index block` nearest to the `footer` at the end of the file, as well as the `BlockHandle` of the `metaindex block`. This determines the locations of those two components in the file.

In fact, in `Status TableBuilder::Finish()` in **table/table_build.cc**, we can see the write order of each component when an SSTable file is generated:

```cpp
Status TableBuilder::Finish() {
  Rep* r = rep_;
  Flush();  /* Write any Block that has not yet been flushed. */
  assert(!r->closed);
  r->closed = true;

  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

  // Write the filter_block, namely the meta block in the diagram.
  if (ok() && r->filter_block != NULL) {
    WriteRawBlock(r->filter_block->Finish(), kNoCompression, &filter_block_handle);
  }

  // Write the metaindex block.
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);
    if (r->filter_block != NULL) {
      // Add mapping from "filter.Name" to location of filter data
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }
    // TODO(postrelease): Add stats and other meta blocks
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

  // Write the index block.
  if (ok()) {
    if (r->pending_index_entry) {
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }
    WriteBlock(&r->index_block, &index_block_handle);
  }

  // Write the footer. The footer has a fixed length and is placed at the end of the file.
  if (ok()) {
    Footer footer;
    // Record the location of the metaindex block in the file.
    footer.set_metaindex_handle(metaindex_block_handle);

    // Record the location of the index block in the SSTable file.
    footer.set_index_handle(index_block_handle);
    std::string footer_encoding;
    footer.EncodeTo(&footer_encoding);
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }
  return r->status;
}
```

The `Finish` function also shows that the positions of the components in the file are exactly as drawn in the figure above.

What are the `index block`, `metaindex block`, `filter block`, the `meta block` in the figure, and even the final `data block` used for? How is the data organized?

* `Data Blocks`: store a sequence of ordered key-value pairs.
* `Meta Block`: stores filters corresponding to key-value data, Bloom Filter by default.
* `metaindex block`: an index pointing to the Meta Block.
* `Index Blocks`: indexes pointing to Data Blocks.
* `Footer`: an index pointing to indexes.

Their relationship is shown in the figure below. [sstable-in-leveldb.md](sstable-in-leveldb.md "mention") explains the relationships and purposes of these components in more detail.

![img](https://s2.loli.net/2022/07/24/qfa37TZkJtQVhAS.png)

The <mark style="color:purple;">**`TableBuilder::Rep`**</mark> structure contains **all information needed to create one file**, including data blocks and index blocks:

```cpp
struct TableBuilder::Rep {
  WritableFile* file;
  uint64_t offset;
  BlockBuilder data_block;
  BlockBuilder index_block;
  std::string last_key;
  int64_t num_entries;
  bool closed;
  FilterBlockBuilder* filter_block;
  ...
}
```

At this point, the full data-read path has been analyzed. For reads, we can understand LevelDB as searching its internal "multi-level cache" in sequence to determine whether a corresponding key exists, and returning the value directly if it exists.

**The only difference from a normal cache is that after data is "hit", LevelDB does not move the data closer. Instead, it moves data farther away to reduce future access time.** This may sound surprising, but it is exactly what compaction achieves.

### **Summary**

In this article, by analyzing read and write operations in LevelDB source code, we learned most of the framework's implementation details, including LevelDB's data format, multi-level SSTables, compaction, and version management. Due to space limitations, some topics are not expanded in detail, such as error recovery and caching. However, reading LevelDB source code deepens our understanding of the distributed key-value storage database described in the Bigtable paper.

LevelDB source code is easy to read and is also an excellent resource for learning C++. If there are questions about the article, they can be discussed in the blog comments.

### **Reference**

* [Bigtable: A Distributed Storage System for Structured Data](https://research.google/pubs/bigtable-a-distributed-storage-system-for-structured-data/)
* [LevelDB](https://github.com/google/leveldb)
* [The Chubby lock service for loosely-coupled distributed systems](https://research.google/pubs/the-chubby-lock-service-for-loosely-coupled-distributed-systems/)
* [LevelDB · Impl](https://github.com/google/leveldb/blob/master/doc/impl.md)
* [LevelDB SSTable](http://bean-li.github.io/leveldb-sstable/)
