---
description: A high-performance embeddable key-value storage engine derived from LevelDB.
---

# 🤩 RocksDB

RocksDB is a high-performance persistent key-value storage engine developed by Facebook based on Google's LevelDB. It is embedded into applications as a library component and is optimized for large-scale distributed applications running on SSDs.

RocksDB does not directly provide high-level system operations such as backup orchestration, load balancing, or snapshot management. Instead, it provides lower-level tools and leaves those system-level policies to the application layer.

This high degree of configurability allows RocksDB to be tuned for a wide range of requirements and workload patterns.
