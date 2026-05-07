# Storage Engines

This section collects notes on storage engines, LSM-tree based designs, probabilistic filters, and the implementations of LevelDB, RocksDB, and TiKV.

Suggested reading order:

1. [Metadata](metadata.md), [LSM Tree](lsm-tree.md), and [SSTable Data Block](sstable.md): start with the basic structures used by storage engines.
2. [Bloom Filter](bloom-filter.md) and [Cuckoo Filter](cuckoo-filter.md): then study the filtering structures commonly used on read paths.
3. [LevelDB](leveldb/README.md), [RocksDB](rocksdb/README.md), and [TiKV](tikv/README.md): finally move into concrete system implementations.

Content status: this area contains both conceptual notes and source-code reading notes. For default parameters, version-specific behavior, and performance conclusions, prefer the target version's source code or official documentation.
