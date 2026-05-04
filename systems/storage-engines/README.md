# Storage Engines

这一节放存储引擎、LSM-tree、过滤器以及 LevelDB/RocksDB/TiKV 实现笔记。

建议阅读顺序：

1. [Metadata](metadata.md)、[LSM Tree](lsm-tree.md)、[SSTable Data Block](sstable.md)：先看存储引擎的基础结构。
2. [Bloom Filter](bloom-filter.md) 和 [Cuckoo Filter](cuckoo-filter.md)：再看读路径常用的过滤结构。
3. [LevelDB](leveldb/README.md)、[RocksDB](rocksdb/README.md)、[TiKV](tikv/README.md)：最后进入具体系统实现。

内容状态：这里既有概念笔记也有源码阅读。涉及默认参数、版本行为和性能结论时，以目标版本源码或官方文档为准。
