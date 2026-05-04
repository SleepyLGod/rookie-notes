# 😄 General

DPU、IPU、SmartNIC 都是在主机 CPU 之外处理网络、存储、安全和虚拟化数据面的硬件平台。粗略区分如下：

* **SmartNIC**：以网卡为中心，强调网络协议处理、流表、加解密、telemetry 或部分可编程 pipeline。
* **DPU / IPU**：通常在 SmartNIC 能力上加入更完整的通用计算核心、内存、加速器和隔离边界，用于承载基础设施服务。
* **软件栈**：常见关键词包括 RDMA verbs、RoCE、NVMe-oF、SPDK、DPDK、virtio/vDPA、DOCA、OVS offload、storage/network/security offload。

本目录目前的重点还是 RDMA 和存储接口基础。后续补 DPU 主题时，建议按“硬件架构 -> 主机接口 -> 数据面 offload -> 控制面/管理面 -> 典型产品栈”的顺序展开。
