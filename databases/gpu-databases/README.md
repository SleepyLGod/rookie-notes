---
description: Notes of papers on DB & GPU.
---

# GPU & DB

_<mark style="color:purple;">**GPU database**</mark><mark style="color:purple;">:</mark> <mark style="color:blue;">GPU is a promising solution for data analytics, driven by the rapid growth of GPU computation power, GPU memory capacity and bandwidth, and PCIe bandwidth. We investigate techniques that can fully unleash the power of GPU in online analytical processing (OLAP) databases.</mark>_

Content status: this page is currently closer to a paper index than a complete survey. When expanding it later, focus on several core GPU OLAP issues: CPU-GPU data movement, PCIe/NVLink bandwidth, operator placement, GPU memory pressure, compression/decompression trade-offs, and the boundary between OLTP and OLAP workloads.

Suggested reading order: first read Crystal to understand the GPU SQL pipeline, then read GPU-compression to understand compression and bandwidth bottlenecks, and finally read Mordred and Active-memory for cross-CPU/GPU/RDMA system design.

* _**Crystal** \[_[_code_](https://github.com/anilshanbhag/crystal)_]\[_[_SIGMOD'20_](https://pages.cs.wisc.edu/\~yxy/pubs/crystal.pdf)_]: A library that can run full SQL queries in GPU and saturate GPU memory bandwidth._
* _**GPU-compression** \[_[_code_](https://github.com/anilshanbhag/gpu-compression)_]\[_[_SIGMOD'22_](https://dl.acm.org/doi/10.1145/3514221.3526132)_]: A highly optimized GPU compression scheme that achieves a high compression ratio and fast decompression speed._
* _**Mordred** \[_[_code_](https://github.com/bwyogatama/Mordred)_]\[_[_VLDB'22_](https://www.vldb.org/pvldb/vol15/p2491-yogatama.pdf)_]: A heterogeneous CPU-GPU query execution engine that optimizes data placement (i.e., semantic-aware caching) and query execution (i.e., segment-level query execution)._

_<mark style="color:purple;">**Advanced network technologies**</mark><mark style="color:purple;">:</mark> <mark style="color:blue;">The network is a bottleneck in distributed databases. Emerging network technologies including RDMA, SmartNIC, and programmable switches support different levels of computation within the network and are promising in accelerating distributed databases.</mark>_

* _**Active-memory** \[_[_VLDB'19_](https://pages.cs.wisc.edu/\~yxy/pubs/p1637-zamanian.pdf)_]: Active-memory replication is a new high-availability scheme that leverages RDMA to directly update replica's memory and eliminate the computation overhead of log replay._
