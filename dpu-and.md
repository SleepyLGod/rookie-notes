---
coverY: 0
---

# 🦸 🦸♂DPU &

这一节收集 DPU/SmartNIC、存储设备和 RDMA 相关笔记。

建议阅读顺序：

1. [General](dpu-and/general.md)：先明确 DPU、IPU、SmartNIC 的边界，以及它们和 RDMA/NVMe/SPDK/DOCA 等技术的关系。
2. [SSD](dpu-and/ssd.md) 与 [NVMe](dpu-and/nvme.md)：理解存储设备和主机接口。
3. [RDMA](dpu-and/rdma/README.md)：再进入 verbs、QP、CQ、MR、PD、RoCE 等细节。

内容状态：目前 RDMA 章节较完整，DPU 总览仍偏薄，后续应优先补架构图、典型 offload 场景和软件栈边界。
