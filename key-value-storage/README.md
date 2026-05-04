---
coverY: 0
---

# Key-value Storage

这里整理键值存储、LSM-tree、经典 Google 存储论文以及 LevelDB/RocksDB/TiKV 相关笔记。

建议阅读顺序：

1. 先读 [CAP theorem](cap-theorem.md)、[LSM Tree](lsm-tree.md)、[Bloom Filter](bloom-filter.md) 和 [Cuckoo Filter](cuckoo-filter.md)，建立基础概念。
2. 再读 Google 三篇经典系统论文：[BigTable](BigTable.md)、[Google File System](gfs.md)、[MapReduce](mapreduce.md)。
3. 最后进入实现层：[LevelDB](leveldb/README.md)、[RocksDB](rocksdb/README.md)、[TiKV](tikv/README.md)。

内容状态：这一组包含原创整理、课程/论文阅读笔记和部分转载改写。涉及具体版本行为时，以对应项目当前官方文档或源码为准。
