---
description: 一个比LevelDB更彪悍的引擎
---

# 🤩 RocksDB

RocksDB 是face book基于Google LevelDB研发的高性能kv持久化存储引擎，以库组件形式嵌入程序中，为大规模分布式应用在ssd上运行提供优化。

RocksDB不提供高层级的操作，例如备份、负载均衡、快照等，而是选择提供工具支持将实现交给上层应用。

正是这种高度可定制化能力，允许RocksDB对广泛的需求和工作负载场景进行定制。

压缩策略可见[lsm-tree.md](../lsm-tree.md "mention")中后半部分。
