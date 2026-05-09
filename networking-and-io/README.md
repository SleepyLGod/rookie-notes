---
coverY: 0
---

# Networking and I/O

This section collects notes on DPU/SmartNIC, storage devices, and RDMA.

Suggested reading order:

1. [DPU and SmartNIC](dpu-smartnic/README.md): first clarify the boundaries between DPU, IPU, and SmartNIC, and how they relate to RDMA, NVMe, SPDK, DOCA, and related technologies.
2. [Storage Devices](storage-devices/README.md): understand SSD and NVMe.
3. [RDMA](rdma/README.md): then move into details such as verbs, QP, CQ, MR, PD, and RoCE.

Content status: the RDMA chapter is currently relatively complete, while the DPU overview is still thin. Later work should prioritize architecture diagrams, typical offload scenarios, and software-stack boundaries.
