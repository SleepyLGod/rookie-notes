---
description: Some note about the famous LevelDB.
---

# 🤩 LevelDB

****

### **LevelDB 整体架构**

![LevelDB 整体架构](https://pic2.zhimg.com/v2-796529d39d931069e82629e73eefa8d1\_b.jpg)

:thumbsup:先嫖个好图，简单展示了 LevelDB 的整体架构。

1. <mark style="color:purple;">**MemTable**</mark>：内存数据结构，具体实现是 SkipList。 接受用户的读写请求，新的数据会先在这里写入。
2. <mark style="color:purple;">**Immutable MemTable**</mark>：当 MemTable 的大小达到设定的阈值后，会被转换成 Immutable MemTable，只接受读操作，不再接受写操作，然后由后台线程 flush 到磁盘上 —— 这个过程称为 minor compaction。
3. <mark style="color:purple;">**Log**</mark>：数据写入 MemTable 之前会先写日志，用于防止宕机导致 MemTable 的数据丢失。一个日志文件对应到一个 MemTable。
4. <mark style="color:purple;">**SSTable**</mark>：Sorted String Table。分为 level-0 到 level-n 多层，每一层包含多个 SSTable，文件内数据有序。除了 level-0 之外，每一层内部的 SSTable 的 key 范围都不相交。
5. <mark style="color:purple;">**Manifest**</mark>：Manifest 文件中记录 SSTable 在不同 level 的信息，包括每一层由哪些 SSTable，每个 SSTable 的文件大小、最大 key、最小 key 等信息。
6. <mark style="color:purple;">**Current**</mark>：重启时，LevelDB 会重新生成 Manifest，所以 Manifest 文件可能同时存在多个，Current 记录的是当前使用的 Manifest 文件名。
7. <mark style="color:purple;">**TableCache**</mark>：TableCache 用于缓存 SSTable 的文件描述符、索引和 filter。
8. <mark style="color:purple;">**BlockCache**</mark>：SSTable 的数据是被组织成一个个 block。BlockCache 用于缓存这些 block（解压后）的数据。

**解析详见：**[**doc/index.md**](https://github.com/google/leveldb/blob/master/doc/index.md) **.**&#x20;

**实现概览参考：**[**doc/impl.md**](https://github.com/google/leveldb/blob/master/doc/impl.md) **.**

公共接口在 <mark style="color:purple;">**`include/leveldb/*.h`**</mark>. 调用者不应引用包内任何其他头文件，内部API可能会变更且没有提示性警告。.

<mark style="background-color:green;">**头文件指引**</mark><mark style="background-color:green;"><mark style="color:blue;">**:**<mark style="color:blue;"></mark>

<mark style="color:blue;">`include/leveldb/db.h`</mark>: DB的主要接口.

<mark style="color:blue;">`include/leveldb/options.h`</mark>: 整个数据库的操作控制，读写操作的控制.

<mark style="color:blue;">`include/leveldb/comparator.h`</mark>: 用户指定的比较函数的抽象接口. 若只需要按key的字节比较，可使用默认的比较函数；也可以自己编写比较函数来自定义排序方式 (例如处理不同的字符编码等.).

<mark style="color:blue;">`include/leveldb/iterator.h`</mark>: 数据迭代的接口，可以从 DB 对象中获取迭代器.

<mark style="color:blue;">`include/leveldb/write_batch.h`</mark>: 使用原子性的多重更新的接口.

<mark style="color:blue;">`include/leveldb/slice.h`</mark>: 保存某个字节数组的地址和长度的简单模块.

<mark style="color:blue;">`include/leveldb/status.h`</mark>: 多数公共接口都会返回 Status ，以获取成功或不同类型失败.

<mark style="color:blue;">`include/leveldb/env.h`</mark>: 操作系统环境的抽象。posix 环境对应的实现在 <mark style="color:purple;">`util/env_posix.cc`</mark>.

<mark style="color:blue;">`include/leveldb/table.h`</mark><mark style="color:blue;">:</mark> 大多数客户端不太会直接用的底层模型.&#x20;

<mark style="color:blue;">`include/leveldb/table_builder.h`</mark>: 大多数客户端不太会直接用的底层模型.&#x20;
