# General

DPU, IPU, and SmartNIC are hardware platforms that process network, storage, security, and virtualization data planes outside the host CPU. A rough distinction is:

* **SmartNIC**: centered on the network interface card, emphasizing network protocol processing, flow tables, encryption/decryption, telemetry, or partially programmable pipelines.
* **DPU / IPU**: usually add more complete general-purpose compute cores, memory, accelerators, and isolation boundaries on top of SmartNIC capabilities, and are used to host infrastructure services.
* **Software stack**: common keywords include RDMA verbs, RoCE, NVMe-oF, SPDK, DPDK, virtio/vDPA, DOCA, OVS offload, and storage/network/security offload.

The current focus of this directory is still RDMA and storage-interface fundamentals. When adding DPU topics later, the suggested order is: hardware architecture -> host interface -> data-plane offload -> control plane / management plane -> typical product stacks.
