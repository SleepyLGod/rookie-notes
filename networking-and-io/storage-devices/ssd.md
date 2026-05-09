---
description: Overview of SSD structure and basic operating principles
---

# 😁 SSD

### 1. What Is an SSD?

A solid-state drive (SSD) is a storage device built from arrays of solid-state electronic memory chips. It consists of a **controller** and **storage units** such as flash chips and, in some designs, DRAM chips. From the operating system's perspective, SSDs use the same interface specifications, functional model, and usage patterns as ordinary disks. Their product form factors and dimensions can also match those of conventional hard drives. SSDs are widely used in military systems, vehicles, industrial control, video surveillance, network monitoring, network terminals, power systems, medical equipment, aviation, navigation devices, and many other domains.

### 2. SSD Categories

SSD storage media are usually divided into two categories. One uses **flash memory** as the storage medium, and the other uses **DRAM** as the storage medium.

**Flash-based SSDs**: Flash-based SSDs, such as IDE Flash Disk and Serial ATA Flash Disk products, use flash chips as the storage medium. This is what people usually mean by SSD. They can be manufactured in many physical forms, such as laptop drives, micro drives, memory cards, and USB flash drives. Their main advantage is mobility. Data protection is not dependent on external power, so they can adapt to many environments and are suitable for personal users.

**DRAM-based SSDs**: DRAM-based SSDs use DRAM as the storage medium, so their application scope is narrower. They imitate traditional hard-disk design, can be configured and managed by most operating-system file-system tools, and provide industry-standard PCI and FC interfaces for host or server connections. Their deployment modes include single SSD drives and SSD arrays. DRAM SSDs provide high performance and long lifetime, but the drawback is that they require an **independent power supply** to protect data safety. DRAM SSDs are relatively niche devices.

### 3. SSD Components

An SSD is mainly composed of a main controller, storage units, optional cache, and a host interface such as SATA, SAS, or PCIe.

#### 3.1 Main Controller

Every SSD has a controller that connects the storage units to the computer. The controller can operate multiple flash packages in parallel through several **channels**, similar to RAID 0, which greatly improves low-level bandwidth.

The controller is an embedded processor that runs firmware code. Its main responsibilities include:

1. Error checking and correction (ECC)
2. Wear leveling
3. Bad-block mapping
4. Read-disturb management, where reading one block may affect data in adjacent blocks
5. Cache control
6. Garbage collection
7. Encryption

#### 3.2 **Storage Units**

Although some vendors have shipped products based on faster DRAM memory, **NAND** flash is still the most common medium and dominates the SSD market.

Low-end products usually use MLC (multi-level cell) or even TLC (triple-level cell) flash. Their typical characteristics are larger capacity, lower speed, lower reliability, lower program/erase endurance, and lower price.

High-end products usually use SLC (single-level cell) flash. Its typical characteristics are mature technology, smaller capacity, higher speed, higher reliability, higher program/erase endurance, and higher price. In practice, however, speed and reliability differences can be mitigated or even reversed by different internal product architectures.

**3.2.1 Internal Structure of Storage Units**

A flash package consists of two or more dies, also called chips. These dies can share an I/O data bus and some control signal lines.

A die can be divided into multiple planes. Each plane contains multiple blocks, and each block contains multiple pages.

Take a Samsung 4 GB flash package as an example. A 4 GB flash package consists of two 2 GB dies, sharing an 8-bit I/O data bus and several control signal lines. Each die contains four planes. Each plane contains 2,048 blocks, and each block contains 64 pages of 4 KB each.

**3.2.2 Storage Cell Types**

* **SLC (Single-Level Cell)**: Each storage cell stores 1 bit of data. Whether the stored data is `0` or `1` is determined by a voltage threshold. For NAND flash programming, the control gate is charged by applying voltage. If enough charge is stored in the floating gate and the voltage exceeds 4 V, the cell represents `0` (programmed). If it is not charged or the voltage threshold is below 4 V, it represents `1` (erased).
* **MLC (Multi-Level Cell)**: Each storage cell stores 2 bits of data. The possible values are `00`, `01`, `10`, and `11`, also determined by voltage thresholds. When the stored charge is below 3.5 V, the value is `11`; between 3.5 V and 4.0 V, it is `10`; between 4.0 V and 5.5 V, it is `01`; above 5.5 V, it is `00`. Compared with SLC, MLC uses the same voltage range but divides it into four threshold regions, which directly affects performance and stability.
* **TLC (Triple-Level Cell)**: TLC is more complex because each storage cell stores 3 bits of data. Its voltage-threshold boundaries are finer, so each storage cell is less reliable.

### 4. How the Host Accesses an SSD

The host accesses an SSD through **LBA** (Logical Block Addressing). Each LBA represents a sector, usually 512 B. Operating systems usually access SSDs in 4 KB units. We call the basic unit used by the host to access the SSD a **host page**.

Inside the SSD, the SSD controller accesses flash using the **flash page** as the basic unit. We call this a **physical page**.

Whenever the host writes one host page, the SSD controller selects one physical page and writes the host data into it. The SSD also records a corresponding **mapping** internally. With this mapping, when the host later needs to **read** a host page, the SSD knows which flash location contains the data.

The SSD maintains a **map table** internally. Each time the host writes a host page, a new mapping is generated. This mapping is **added** to the map table for the first write, or **updated** for an overwrite. When reading a host page, the SSD first looks up the physical page corresponding to that host page in the map table, and then accesses flash to read the corresponding host data.

Most SSDs include **on-board DRAM**. Its main purpose is to store this map table.

There are exceptions. For example, some SandForce-controller SSDs do not support on-board DRAM. Where is their map table stored? During SSD operation, most mappings are stored in flash, and a smaller portion is stored in on-chip RAM.

When the host reads data from an SSD with on-board DRAM, the SSD only needs to look up the map table in DRAM, obtain the physical address, and then access flash to retrieve the host data. This path needs only one flash access. For a SandForce SSD, the controller first checks whether the mapping for the host page is in RAM. If it is, the controller reads flash directly according to the mapping. If the mapping is not in RAM, the controller first reads the mapping from flash, and then reads the host data using that mapping. Compared with an SSD with DRAM, this requires two flash reads before returning the data, effectively halving the low-level bandwidth. For random host reads, because on-chip RAM is limited, the probability of a mapping-cache hit is low. Therefore, these SSDs usually need two flash accesses for each read, which explains why SandForce-controller SSDs often have weaker random-read performance.

### 5. SSD Concepts and Techniques

#### 5.1 Multi-Plane NAND

Multi-plane NAND is a design that can improve performance effectively. For example, one die may be divided into two planes, and the block numbers in the two planes may be interleaved as odd and even numbers. During operation, the controller can also execute **interleaved operations** across the planes, such as one odd block and one even block, to improve performance.

#### 5.2 FTL

Operating systems usually treat a disk as **a sequence of 512 B sectors**. Note that the minimum unit for one disk read or write by an operating system is not necessarily the sector; it is usually the file-system **block**, commonly 512 B, 1 KB, or 4 KB, and sometimes larger, depending on the format-time configuration. However, **the flash read/write unit is a 4 KB or 8 KB page**, while **flash erase, also called programming in this note, operates on blocks made of 128 or 256 pages**. More importantly, data cannot be overwritten in place. The whole block must be erased before new data can be written. This is completely incompatible with file systems designed for traditional hard drives. Clearly, a more advanced file system designed specifically for SSDs would be needed to fit this operating model.

Unfortunately, such file systems are not yet the default mainstream solution. To remain compatible with existing file systems, SSDs introduce the **FTL (Flash Translation Layer)**. It sits between the file system and the physical medium, virtualizing flash behavior into the traditional 512 B sector model used by hard drives. With FTL, the operating system can operate as if it were using ordinary sectors, without worrying about the erase/read/write constraints mentioned above. All **logical-to-physical** translation is handled by the FTL.

An FTL algorithm is essentially a logical-to-physical mapping. When the file system sends a command to write or update a specific logical sector, the FTL actually writes the data to another free physical page, updates the map table, and marks the old page containing the previous data as invalid. The new data has already been written to a new address, so the old address naturally becomes invalid.

#### 5.3 Wear Leveling

Simply put, wear leveling is a mechanism that ensures **each flash block is written approximately the same number of times**.

In normal usage, data in NAND blocks is updated at different frequencies. Some data is updated often, while some data is rarely updated. Obviously, blocks containing frequently updated data wear out quickly, while blocks containing rarely updated data wear much more slowly.

To solve this problem, the SSD needs to keep the program/erase count of every block as even as possible. This requires monitoring read/program operations for each page. In the most optimistic case, this technique makes the physical wear of all flash cells across the drive roughly equal, so the drive ages uniformly. Wear-leveling algorithms are divided into **dynamic** and **static** approaches.

Dynamic wear leveling is the basic wear-leveling approach. It only balances physical page addresses occupied by files that are updated during user workloads.

Static wear leveling is a more advanced approach. On top of dynamic wear leveling, it also balances physical addresses occupied by files that are rarely updated. This is true **whole-drive wear leveling**. In simple terms, dynamic wear leveling always tries to choose the youngest NAND blocks for new writes and avoids older NAND blocks when possible. Static wear leveling moves long-lived cold data out of young NAND blocks and places it into older NAND blocks, allowing the younger blocks to return to the frequently used pool. Although wear leveling prevents data from being repeatedly written to the same physical area and helps keep wear roughly uniform across storage regions, thereby extending SSD lifetime, it can also hurt SSD performance.

#### 5.4 Garbage Collection

When an SSD is completely full, from the user's perspective, writing new data requires deleting some data first and freeing space. During deletion and writing, data in some blocks becomes invalid or stale. Data in a block becomes stale or invalid when **no mapping points to it anymore**. Users will no longer access those flash locations because they have been replaced by new mappings.

For example, suppose host page A is initially stored at flash location X, and the mapping is `A -> X`. Later, the host rewrites this host page. Because **flash cannot overwrite in place**, the SSD must find an unwritten location for the new data. Suppose it chooses location Y. A new mapping is established: `A -> Y`. The previous mapping is removed, and the data at location X becomes stale and invalid. We call this garbage data. As the host continues writing, available flash space gradually decreases until it is exhausted. If this garbage data is not reclaimed in time, the host will no longer be able to write.

SSDs include an internal garbage-collection mechanism. Its **basic principle** is to collect valid data, meaning non-garbage data, from several blocks into a new block, and then erase those old blocks. This creates new available blocks.

From the wear-leveling discussion above, wear leveling requires "blank blocks" for updated data. **When the number of spare blank blocks that can be written directly falls below a threshold**, the SSD controller gathers all valid data from blocks containing invalid data, writes that valid data into new blank blocks, and then erases the old blocks to increase the number of spare blank blocks.

There are three garbage-collection strategies:

* **Idle garbage collection**: Garbage collection obviously consumes a large amount of controller processing power and bandwidth, reducing performance for user requests. The SSD controller can perform proactive garbage collection while the **system is idle**, preserving a certain number of spare blank blocks so the SSD can maintain higher performance during active operation.

  The drawback of idle garbage collection is that it increases extra write amplification. The "valid data" just moved by garbage collection may soon be replaced by updated data and become invalid, making the previous garbage-collection work wasted.
* **Passive garbage collection**: This is supported by every SSD, but it places high demands on **controller performance** and is suitable for server use. SandForce controllers are an example of this category. User operations are processed while garbage collection consumes bandwidth and compute capacity. If the controller is not strong enough, performance drops significantly. This explains why many SSDs slow down after the whole drive has been filled once: continued writing requires garbage collection and writes to proceed at the same time.
* **Manual garbage collection**: The user manually chooses an appropriate time to run garbage-collection software and execute garbage collection.

It is easy to imagine that if the system performs garbage collection too often and repeatedly erases blocks, SSD lifetime can decrease further. Therefore, controlling garbage-collection frequency while ensuring longer flash-chip lifetime requires a careful balance. This is why SSDs need to support **TRIM**. Without TRIM, garbage collection cannot fully show its advantages.

#### 5.5 TRIM

TRIM is an ATA command. When the operating system deletes a file or formats a region, it sends the corresponding file address to the SSD controller, letting the controller know that the data at that address is invalid.

When a file is deleted, the file system does not actually erase it immediately. It only marks the **file address as deleted**, meaning it can be reused. This means the address occupied by the file is now invalid. This creates a problem: the disk itself does not know that the operating system has marked the address as deleted. For mechanical disks this is not a problem, because the same address can simply be overwritten. For SSDs, however, NAND must be erased before data can be written again. To obtain free NAND space, the SSD must copy all valid pages into a new free block and erase the old block through garbage collection. Without TRIM, the SSD controller does not know that a page is invalid unless the operating system later asks to overwrite it. TRIM is simply a command that lets the operating system tell the SSD controller that the page is invalid. TRIM reduces write amplification because the controller no longer needs to copy invalid pages, which would otherwise still look valid, into blank blocks. This also reduces the number of valid pages that must be copied, improving garbage-collection efficiency and SSD performance. TRIM can greatly reduce the number of pseudo-valid pages and significantly improve garbage collection. Supporting TRIM requires three components:

* **System**: The operating system must send TRIM commands. Examples include Windows 7, Windows Server 2008 R2, and Linux 2.6.33 or later.
* **Firmware**: The SSD vendor must implement a TRIM algorithm in firmware, meaning the SSD controller must recognize the TRIM command.
* **Driver**: The controller driver must support TRIM command transport, meaning it can pass TRIM commands to the SSD controller. Microsoft's driver and Intel's AHCI driver support it; support in other drivers depends on later updates.

At present, disks in RAID arrays clearly do not support TRIM, although RAID arrays can still support garbage collection.

#### 5.6 Over-Provisioning

Over-provisioning refers to capacity that users cannot operate directly. It is **the actual physical flash capacity minus the user-visible capacity**. This region is generally used for optimization, including wear leveling, garbage collection, and bad-block mapping.

The first layer is a fixed 7.37%. Where does this number come from? Mechanical disk and SSD manufacturers calculate capacity using 1 GB = 1,000,000,000 bytes, or 10^9 bytes. **However, the actual binary capacity of flash is 1 GiB = 1,073,741,824 bytes, or 2^30 bytes**. The difference is 7.37%. Therefore, for a 128 GB SSD, the user receives 128,000,000,000 bytes, and the extra 7.37% is used by controller firmware as over-provisioning.

The second layer comes from manufacturer configuration, usually 0%, 7%, or 28%. For example, for an SSD using 128 GB of flash with a SandForce controller, the market may offer 120 GB and 100 GB models. This depends on the vendor's firmware settings and does not include the first 7.37% layer.

The third layer is over-provisioning that users can allocate during daily use. Users can intentionally avoid partitioning the full SSD capacity. However, a secure erase should be performed first to ensure that this space has truly not been used.

Specific uses of over-provisioning:

* **Garbage collection**: Garbage collection moves data around, so it needs free space to place moved data. The more free space available, the faster the movement. Some SSDs take additional user capacity to become faster.
* **Map table and other internal metadata**: The SSD contains a large map table that translates user addresses into physical flash addresses. This table must be saved to prevent loss after power failure. It consumes roughly 0.3% of the capacity.
* **Bad-block replacement**: As writes accumulate, bad blocks gradually increase and need to be replaced by good blocks. As flash process nodes shrink from 32 nm to 14 nm, flash quality declines and bad blocks increase. This portion may reach 3% or more.

#### 5.7 Write Amplification

Because flash must be erased, also called programmed in this note, before it can be written, user data and metadata may be moved or overwritten more than once during these operations. These extra operations increase the amount of data written, reduce SSD lifetime, consume flash bandwidth, and indirectly affect random-write performance. This **effect** is called write amplification. The quality of a **controller** is largely reflected in its write-amplification behavior.

For example, suppose I want to write 4 KB of data. In the worst case, a block has no clean space left, but it contains invalid data that can be erased. The controller then reads all data into cache, erases the block, updates the whole block's data in cache, and writes the new data back. The write amplification caused by this operation is: I logically write 4 KB of data, but the system performs an entire block write of 1,024 KB, which is 256x amplification. The simple 4 KB write becomes: <mark style="color:red;">**<**</mark> flash read (1,024 KB) <mark style="color:red;">**->**</mark> cache update (4 KB) <mark style="color:red;">**->**</mark> flash erase (1,024 KB) <mark style="color:red;">**->**</mark> flash write (1,024 KB) <mark style="color:red;">**>**</mark>. The resulting latency increase and performance drop are natural consequences. Therefore, write amplification is a key factor affecting SSD random-write performance and lifetime.

When writing 100% random 4 KB data to an SSD, for most current SSD controllers, the actual write amplification in the worst case may reach or exceed 20x. Users can reduce write amplification by configuring over-provisioning. For example, if a 128 GB SSD is partitioned with only 64 GB for use, worst-case write amplification can be reduced by about three times.

Many factors affect SSD write amplification. The main factors and their effects are listed below:

* Garbage collection increases write amplification, although passive garbage collection does not have the same effect as **idle garbage collection**, and it can improve speed.
* Over-provisioning reduces write amplification. More over-provisioning means lower write amplification.
* Enabling TRIM reduces write amplification.
* More unused space during user workloads reduces write amplification, assuming TRIM support exists.
* **Sequential writes** reduce write amplification. In theory, write amplification for sequential writes is 1, although some factors can still affect the actual value.
* **Random writes** greatly increase write amplification because they write many non-contiguous LBAs.
* The **wear-leveling** mechanism directly increases write amplification.

#### 5.8 Interleaved Operations

Interleaved operations can **multiply NAND transfer rate**, because NAND packages may contain multiple dies and multiple planes, with each plane having a 4 KB register. Plane operations can be interleaved: after the first plane receives an instruction and starts operating, the second instruction can already be sent to the second plane, and so on. This can achieve nearly 2x or even 4x transfer capability, depending on flash support.

#### 5.9 ECC

ECC stands for Error Checking and Correction. It is an error-detection and correction algorithm used for NAND.

The NAND flash manufacturing process cannot guarantee that NAND remains reliable throughout its lifecycle, so **bad blocks** appear during NAND production and use. To verify data reliability, systems using NAND flash usually adopt a bad-block management mechanism. Reliable bad-block detection is the prerequisite for bad-block management.

If there are no timing or circuit-stability problems, NAND flash errors usually do not make an entire block or page unreadable or completely incorrect. Instead, only one or several bits in a page are wrong. This is where ECC becomes useful. Different flash particles have different basic ECC requirements, and different controllers support different ECC capabilities. In theory, stronger controllers provide stronger ECC capability.



References: [http://www.jinbuguo.com/storage/ssd\_intro.html](http://www.jinbuguo.com/storage/ssd\_intro.html)

&#x20;         [http://www.ssdfans.com/?p=131](http://www.ssdfans.com/?p=131)
