# 😙 RocksDB & LevelDB

RocksDB evolved from the original LevelDB codebase, so its basic API style is very similar to LevelDB. The following two examples show the similarity:

**RocksDB:**

```cpp
#include "rocksdb/db.h"

rocksdb::DB* db;
rocksdb::Options options;
options.create_if_missing = true;

rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);

assert(status.ok());

status = db->Get(rocksdb::ReadOptions(), key1, &value);
status = db->Put(rocksdb::WriteOptions(), key2, value);
status = db->Delete(rocksdb::WriteOptions(), key1);

delete db;
```

**LevelDB:**

```cpp
#include "leveldb/db.h"

leveldb::DB *db;
leveldb::Options options;
options.create_if_missing = true;

leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);

assert(status.ok());

status = db->Get(leveldb::ReadOptions(), key1, &value);
status = db->Put(leveldb::WriteOptions(), key2, value);
status = db->Delete(leveldb::WriteOptions(), key1);

delete db;
```

**However:** for details, see the [**RocksDB documentation**](https://github.com/facebook/rocksdb/wiki/Features-Not-in-LevelDB).

* Although RocksDB evolved from LevelDB at the code level, RocksDB targets server-side, flash-based, and multi-core environments. It exposes more tunable parameters, background-thread controls, and operational tools. LevelDB is a smaller and more focused embedded storage engine; applications usually need to implement backup, monitoring, throttling, sharding, and similar capabilities at a higher layer.
* RocksDB supports `MultiGet`, prefix/range-related optimizations, and richer iterator options. LevelDB also supports range scans through iterators, but its interface and optimization options are simpler; it should not be summarized as "only able to fetch a single key".
* In addition to simple `Put` and `Delete` operations, RocksDB provides `Merge`, which can combine multiple updates through an application-defined merge operator.
* RocksDB provides convenient tools for tasks such as parsing key-value records in SST files and inspecting MANIFEST contents. With these tools, users do not have to write a program just to inspect detailed SST key-value information, which is often necessary when working with LevelDB alone.
* RocksDB supports multi-threaded compaction, while LevelDB compaction is single-threaded. For LSM-based structures, compaction cost is one of the major performance bottlenecks. In multi-CPU environments, multi-threaded compaction is a major advantage over LevelDB. According to the official description, some multi-threaded compaction paths may still apply only to files that do not overlap with the next level and can be handled by simple file movement or renaming. Whether full data-merging compaction also benefits from multi-threading needs to be verified from the implementation.
* RocksDB adds compaction filters, which can discard key-value pairs that no longer satisfy application-defined conditions, such as expiration based on TTL.
* RocksDB supports multiple compression algorithms. In addition to Snappy, which LevelDB uses, RocksDB can use algorithms such as zlib and bzip2.

    LevelDB decides whether to store data in compressed form based on the compression ratio, for example when compressed data is less than 75% of the original size. A typical RocksDB configuration is to leave levels 0-2 uncompressed, use zlib for the last level, and use Snappy for the intermediate levels.
* For fault recovery and operations, RocksDB supports both incremental and full backups, and can back up deleted data to a specified directory for later recovery.
* Both LevelDB and RocksDB use a `LOCK` file in the database directory to prevent multiple processes or multiple handles from opening the same DB directory concurrently. The restriction is that the same DB directory cannot be opened concurrently, not that a single process can only have one LevelDB instance.
* RocksDB supports pipelined MemTables, meaning it can create multiple MemTables as needed to reduce the performance bottleneck caused by mismatched `Put` and compaction speeds. In LevelDB, there is only one active MemTable. If the MemTable is full but has not yet been persisted, LevelDB slows down `Put` operations, which hurts overall performance. The engine I was writing at the time happened to make a similar design choice to RocksDB here.

Although RocksDB improves performance substantially, its on-disk file format is still broadly similar to LevelDB's. One notable change is that RocksDB makes more concrete use of the MetaBlock area that LevelDB had reserved in SST files.
