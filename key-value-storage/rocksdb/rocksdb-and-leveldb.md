# 😙 RocksDB & LevelDB

RocksDB是在LevelDB原来的代码上进行改进完善的，所以在用法上与LevelDB非常的相似，请欣赏如下两段代码：

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

**然而：**（具体可见 [**RocksDB文档**](https://github.com/facebook/rocksdb/wiki/Features-Not-in-LevelDB)）

*   虽然在代码层面上RocksDB 是在LevelDB原有的代码上进行开发的，但也开始支持HDFS，允许从HDFS读取数据。

    LevelDB则是一个比较单一的存储引擎。也是因为LevelDB的单一性，在做具体的应用的时候一般需要对其作进一步扩展。
*   RocksDB支持一次获取多个K-V，还支持Key范围查找；

    LevelDB只能获取单个Key。
* RocksDB 除了简单的Put、Delete操作，还提供了一个Merge操作，对多个Put操作进行合并。
* RocksDB提供一些方便的工具，这些工具包含解析sst文件中的K-V记录、解析MANIFEST文件的内容等。有了这些工具，就不用再像使用LevelDB那样，只能在程序中才能知道sst文件K-V的具体信息了。
* RocksDB 支持多线程合并，而LevelDB是单线程合并的。LSM型的数据结构，最大的性能问题就出现在其合并的时间损耗上，在多CPU的环境下，多线程合并那是 LevelDB所无法比拟的。不过据其官网上的介绍，似乎多线程合并还只是针对那些与下一层没有Key重叠的文件，只是简单的rename而已，至于在真正数据上的合并方面是否也有用到多线程，就只能看代码了。
* RocksDB增加了合并时过滤器，对一些不再符合条件的K-V进行丢弃，如根据K-V的有效期进行过滤。
*   压缩方面RocksDB可采用多种压缩算法，除了LevelDB用的snappy，还有zlib、bzip2。

    LevelDB里面按数据的压缩率（压缩后低于75%）判断是否对数据进行压缩存储，而RocksDB典型的做法是Level 0-2不压缩，最后一层使用zlib，而其它各层采用snappy。
* 在故障方面，RocksDB支持增量备份和全量备份，允许将已删除的数据备份到指定的目录，供后续恢复。
* RocksDB支持在单个进程中启用多个实例，而LevelDB只允许单个实例。
* RocksDB 支持管道式的Memtable，也就说允许根据需要开辟多个Memtable，以解决Put与Compact速度差异的性能瓶颈问题。在LevelDB里面因为只有一个Memtable，如果Memtable满了却还来不及持久化，这个时候LevelDB将会减缓Put操作，导致整体性能下降。笔者目前写的引擎在这方面竟然跟RocksDB不谋而合，这里偷偷乐一下，呵呵。

不过虽然RocksDB在性能上提升了不少，但在文件存储格式上跟LevelDB还是没什么变化的， 稍微有点更新的只是RocksDB对原来LevelDB中sst文件预留下来的MetaBlock进行了具体利用。
