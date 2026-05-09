# 😆 NVMe

### **1. NVMe Protocol**

The NVMe protocol was released in 2011 after PCIe SSDs began appearing in large numbers on the market. At that time, vendors used private protocols that were not mutually compatible and could not integrate seamlessly with existing operating systems. Intel introduced NVMe to **standardize the interface protocol and build an ecosystem**.

![](http://pic.doit.com.cn/2018/01/1.png)

NVMe uses **multiple command queues**, supporting up to 65,536 command queues. Each command can have variable data length, from 512 B to 2 MB. On the host-memory side, data can be described using **Physical Region Page** and **Scatter Gather List** structures.

The NVMe protocol supports **out-of-order** execution between commands. It also supports out-of-order transfer of data blocks within a command, and variable-weight scheduling across command queues.

![](http://pic.doit.com.cn/2018/01/2.png)

![](http://pic.doit.com.cn/2018/01/3.png)

![](http://pic.doit.com.cn/2018/01/4.png)

Compared with the traditional ATA-based **SATA** protocol, which was designed around PC-era disk interfaces, NVMe makes many protocol-level optimizations for multi-core hosts and NAND storage media.

![](http://pic.doit.com.cn/2018/01/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE\_20180110102920.png)

Because PCIe provides high interface bandwidth, NVMe improves SSD access efficiency. At the same time, it makes the NVMe protocol-stack implementation on the SSD controller side more difficult. The main challenges are:

1. Managing and fetching commands from multiple weighted command queues.
2. Parsing, decomposing, and dispatching commands.
3. Transferring data.

Before discussing possible solutions, first look at the basic architecture of an SSD controller.

### **2. SSD Controller**

A typical SSD controller architecture is shown below:

![](http://pic.doit.com.cn/2018/01/5.png)

**Host interface controller**

The host interface is responsible for communication and data transfer between the host and the SSD. It receives and parses I/O requests, and maintains one or more request queues.

Mainstream physical interfaces include SATA, SAS, PCIe, and M.2.

SATA is a low-cost disk interface. SATA 3.0 has a theoretical bandwidth of 600 MB/s.

SAS is backward compatible with SATA and provides higher transfer rate, reliability, and availability. SAS 3.0 has a theoretical bandwidth of 1.2 GB/s.

PCIe is the main bus for connecting high-speed peripheral devices and host processors. SATA/SAS devices usually connect to the system through a host controller or HBA, and those controllers may themselves sit on PCIe, but the SATA/SAS protocols are not equivalent to PCIe. PCIe SSDs directly use PCIe lanes, shortening the path and response latency from the host processor to the storage medium while also providing greater bandwidth. PCIe 3.0 x4 and x8 have theoretical bandwidths of roughly 4 GB/s and 8 GB/s respectively.

M.2 is a newer interface standard. It supports both PCIe lanes and SATA channels, and has more flexible physical specifications. It is better than the SATA interface in transfer bandwidth, capacity, and thin-and-light device design. Consumer SSDs mainly use SATA and M.2 interfaces, while enterprise SSDs mainly use SAS and PCIe interfaces. The logical interface protocols used by these physical interfaces are usually AHCI or NVMe. These protocols define how the host communicates with the storage device and transfers data. AHCI was designed mainly for high-latency mechanical disks using the SATA interface. Although it can be used with SATA SSDs, it becomes a bottleneck for high-performance SSDs. NVMe is a non-volatile memory standard led by Intel and other companies in 2011, tailored for flash storage and PCIe interfaces. It is cross-platform compatible, with mainstream support in Windows, Linux, VMware, and other platforms, and compared with AHCI it offers lower latency, higher IOPS, and lower power consumption. For example, AHCI supports only one command queue with a maximum depth of 32, while NVMe can support up to 64,000 command queues with a maximum depth of 64,000, fully exploiting SSD internal parallelism. Therefore, NVMe better exposes high SSD performance and has been adopted increasingly widely.

**Multi-core processor**

SSD management involves many complex tasks, such as host-interface protocols, scheduling algorithms, FTL algorithms, and caching algorithms. Therefore, powerful multi-core processors are needed to improve task-processing efficiency and reduce software latency. For example, one compute core can process the host-interface protocol, while multiple compute cores handle the heavier FTL algorithm work.

**Cache chip**

An SSD includes an internal cache chip, usually DRAM, for caching user data and metadata used by software algorithms. Cache can accelerate data access, improve SSD performance, reduce flash writes, and extend SSD lifetime. The part used for caching user data is called the data cache, while the part used for caching the address mapping table is called the mapping cache. To prevent sudden power loss from losing data in RAM, SSDs usually include backup capacitors and suitable data-protection techniques, ensuring that critical dirty data in RAM can be flushed back to flash when power is unexpectedly lost.

**Central controller**

The central controller is the core of the SSD controller. It configures the SSD operating mode and manages communication and data flow between modules. The central controller contains a small high-speed SRAM cache for temporary data caching.

**Error-correction-code engine**

The error-correction-code engine encodes data before it is written into flash, and the added correction-code redundancy is written into the flash page's extra storage area. When data is read from flash, the engine decodes both the data and its correction-code redundancy. If the number of bit errors is within the correction capability, the errors are corrected and the correct data is obtained. Otherwise, if there is no other data-recovery method, the stored data is lost. Encoding and decoding are on the critical path of flash access, so the error-correction-code engine is usually implemented as dedicated hardware to improve encoding and decoding efficiency. To reduce hardware overhead and decoding latency, the engine divides data into fixed-size fragments, usually 1 KB, 2 KB, or 4 KB, as encoding and decoding units. A data segment and its correction-code redundancy form a codeword. The ratio between the data-fragment length and the codeword length is the code rate. A lower code rate gives stronger correction capability, but also higher cost, such as redundant storage overhead and decoding latency. To increase encoding and decoding parallelism, the engine usually contains multiple encoders and decoders. Common SSD error-correction-code algorithms include BCH and LDPC. LDPC has become preferred because of its stronger correction capability.

**Channel controller**

To improve performance, SSDs place many flash chips across multiple channels. Multiple chips on each channel share one I/O bus. Each channel has an independent channel controller. It is mainly responsible for communicating with the central controller and the flash chips on the channel, maintaining multiple flash-operation command queues, such as one queue per chip plus a high-priority global queue, and sending commands to target chips for execution. Therefore, the SSD has four internal levels of parallelism: channels, chips, dies, and groups. Efficiently using this parallelism is key to high SSD performance.

**Flash chips**

Flash chips store both user data and metadata that must be persisted, such as the address mapping table.

The physical storage capacity provided by an SSD is larger than the user-visible capacity, usually by 7% to 28%. The extra capacity is called over-provisioning (OP) space. It is mainly used to improve software algorithms such as garbage collection and to compensate for capacity loss caused by flash bad blocks. Some SSDs also build RAID 5 across flash chips to improve storage reliability.

**DMA engine**

The DMA engine controls fast data transfer between two RAM regions, such as between the central controller's SRAM and the cache chip.

Different SSDs may vary in module design. For example, a pair of error-correction-code encoders and decoders may be placed inside each channel controller instead of using one centralized engine. Some SSDs may also include more modules, such as power management, RAID engines, data compression engines, and encryption engines. However, the modules described above explain the main internal architecture of an SSD.

### **3. NVMe Host Interface Controller Implementation**

Generally, an NVMe host interface controller includes the following parts:

Admin command and completion queue processing module, I/O command queue processing module, I/O completion queue processing module, I/O write-data processing module, I/O read-data processing module, and interrupt processing module.

![](http://pic.doit.com.cn/2018/01/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE\_20180110103110.png)

Returning to the challenges faced by the NVMe protocol stack: 1. managing and fetching commands from multiple weighted command queues; 2. parsing, decomposing, and dispatching commands; 3. transferring data.

**Command queue management**

![](http://pic.doit.com.cn/2018/01/6.png)

NVMe command queues are divided into two types: admin command queues and I/O command queues.

The admin command queue is responsible for protocol initialization and management. It is not responsible for user-data transfer and has no strict bandwidth or response-time requirement. It can usually be handled by CPU firmware responsible for front-end protocol processing. In this model, firmware processes the doorbell of the admin command queue, fetches commands from host memory, parses the commands, returns or fetches the data segment for admin commands according to command fields, and then sends completion entries back to the completion queue.

I/O command queues are used for data transfer and support up to 65,536 command queues. In practice, the number of command queues is often between 16 and 256 depending on the number of host cores. Data transfer imposes high requirements on command-queue processing. The SSD controller's interface controller must efficiently fetch, parse, and dispatch commands from command queues with different weights. This creates a major design challenge for the host interface controller. There are usually three design approaches for I/O command queues:

1. Hardware logic processing.
2. Parallel firmware processing on multiple CPU cores.
3. Hardware logic plus CPU firmware processing.

We can compare the advantages and disadvantages of these three approaches:

![](http://pic.doit.com.cn/2018/01/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE\_20180110104356.png)

Because the NVMe protocol is complex, pure hardware logic processing is not recommended. Approaches 2 and 3 are more feasible.

Command queue management must solve several difficult problems: polling order and priority across multiple weighted queues, command-field parsing and decomposition, PRP-list processing for I/O read/write commands, and Scatter Gather List processing for I/O read/write commands.

**1. Polling order and priority across multiple weighted queues**

![](http://pic.doit.com.cn/2018/01/7.png)

Polling I/O command queues and fetching commands require continuous polling of doorbell registers. Command fetching also requires frequent activation of the PCIe IP block. This is not well suited for CPU firmware, because it would seriously affect command-processing efficiency. Each command field is fixed at 64 B, so hardware logic is recommended for this part. However, handling command polling across different weights requires very complex hardware logic and creates a demanding design challenge.

**2. Command-field parsing and decomposition**

The fields of an NVMe I/O command are shown below.

![](http://pic.doit.com.cn/2018/01/8.png)

![](http://pic.doit.com.cn/2018/01/9.png)

Each I/O command consists of 64 B. DWORD 0-9 have fixed definitions, while the remaining fields have different meanings for different I/O commands. As the protocol evolves, new commands and parameters may also be introduced. Therefore, protocol parsing should be completed by CPU firmware to preserve flexibility.

The data blocks corresponding to an I/O command can be stored in host memory in two forms: PRP list and SGL.

PRP list:

![PRP list](http://pic.doit.com.cn/2018/01/10.png)

Physical Region Page.

SGL:

![SGL](http://pic.doit.com.cn/2018/01/11.png)

Scatter Gather List.

Data in host memory is generally stored in 4 KB granules, but the physical addresses of those granules are not necessarily contiguous, and the offset of the starting granule is not necessarily 4 KB aligned, as shown below.

![](http://pic.doit.com.cn/2018/01/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE\_20180110104724.png)

When the host controller starts PCIe DMA access to host memory, it must handle the details caused by 4 KB misalignment: the starting block may not be 4 KB aligned, the starting block size may be partial, and the ending block size may also be partial. These blocks require non-4 KB PCIe DMA transfers. CPU firmware provides enough flexibility for host-memory access processing, but because host data transfer has high bandwidth requirements, hardware assistance or direct hardware processing is needed. For example, a pipeline mechanism can be added to PCIe DMA to fully utilize PCIe bandwidth.

Because each I/O command can transfer data ranging from 512 B to 256 MB, while the SSD central controller, or FTL management module, usually manages NAND-side data in 4 KB units, data decomposition is required.

![](http://pic.doit.com.cn/2018/01/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE\_20180110104906.png)

If the host-side I/O data block is larger than 4 KB, each I/O data block transfer involves data-block decomposition. The host-side data block must be decomposed into 4 KB granules required by the FTL management module. From a modular layered-design perspective, data segmentation is not suitable for the FTL management module. It should be completed by the host controller module, so the NVMe protocol portion remains transparent to the FTL control module. Similarly, the host-side protocol should also be transparent to the FTL management module.

For 4 KB LBA mode, because the data offset is 4 KB aligned, host-side data decomposition is simple.

For non-4 KB LBA modes such as 512 B, 1 KB, and 2 KB, decomposing host-side data blocks into the 4 KB units required by the FTL control module is more complex, with many cases. The SLBA (Start Logic Address) of each I/O command is not necessarily 4 KB aligned, and the Logic Block Number is not necessarily a multiple of 4 KB. Therefore, the first and last blocks after data decomposition may not be complete 4 KB blocks. The number of decomposed 4 KB blocks on the host side must consider both Logic Block Number and SLBA misalignment. For example, a 12 KB host-side data block, where 12 KB / 4 KB = 3, may be distributed across four 4 KB FTL NAND-side data blocks after decomposition.

![](http://pic.doit.com.cn/2018/01/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE\_20180110105117.png)

Host memory is not 4 KB aligned, SLBA is 4 KB aligned, and Logic Block Number is a multiple of 4 KB.

![](http://pic.doit.com.cn/2018/01/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE\_20180110105435.png)

Host memory is not 4 KB aligned, SLBA is not 4 KB aligned, and Logic Block Number is a multiple of 4 KB.

From the two figures above, whether the data block is 4 KB aligned in host memory, whether SLBA is 4 KB aligned, and whether Logic Block Number is a multiple of 4 KB combine into many complex cases. For example, if the data block is not 4 KB aligned in host memory, then a 4 KB block visible to the FTL side may be distributed across two host-memory blocks.

![](http://pic.doit.com.cn/2018/01/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE\_20180110105758.png)

Interested readers can enumerate all combinations. Each case has a different data-decomposition pattern.

![](http://pic.doit.com.cn/2018/01/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE\_20180110105902.png)

If the decomposed I/O data block is smaller than 4 KB, such as 512 B, 1 KB, 1.5 KB, or 2 KB, then for write I/O it involves stitching the incomplete data portion together with the existing background data in the FTL's 4 KB data block.

![](http://pic.doit.com.cn/2018/01/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE\_20180110110046.png)

NVMe separates the command-control path from the data path. It supports multiple command queues, out-of-order execution and data transfer between commands, and out-of-order execution of decomposed 4 KB data blocks inside a command. Therefore, controlling command-level and intra-command control flow and data flow is an important topic with many technical difficulties. This document does not expand on it further. The next note focuses on that topic.
