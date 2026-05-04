---
coverY: 0
---

# Networking and I/O

这一节收集 DPU/SmartNIC、存储设备和 RDMA 相关笔记。

建议阅读顺序：

1. [DPU and SmartNIC](dpu-smartnic/README.md)：先明确 DPU、IPU、SmartNIC 的边界，以及它们和 RDMA/NVMe/SPDK/DOCA 等技术的关系。
2. [Storage Devices](storage-devices/README.md)：理解 SSD 和 NVMe。
3. [RDMA](rdma/README.md)：再进入 verbs、QP、CQ、MR、PD、RoCE 等细节。

内容状态：目前 RDMA 章节较完整，DPU 总览仍偏薄，后续应优先补架构图、典型 offload 场景和软件栈边界。
