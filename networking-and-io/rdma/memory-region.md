# 😃 Memory Region

Assume the following scenario, which also reviews the RDMA WRITE flow:

As shown below, node A wants to write a piece of data into the memory of node B through the IB protocol. The upper-layer application posts a WQE to the local RDMA NIC. The WQE contains information such as source memory address, destination memory address, data length, and key. Hardware then fetches data from memory, assembles packets, and sends them to the peer NIC. After node B's NIC receives the data, it parses the destination memory address and writes the data into the memory of node B.

<figure><img src="https://pic4.zhimg.com/v2-5a8bae1c63fa44ab66b2d06d136acdd7_b.jpg" alt=""><figcaption></figcaption></figure>

The question is: addresses provided by applications are virtual addresses, or VAs below. They must be translated by the MMU before real physical addresses, or PAs below, are obtained. **How does the RDMA NIC obtain the PA so it can fetch data from memory?** Even if the NIC knows where to fetch data, **if a malicious user specifies an illegal VA, could the NIC be "instructed" to read or write critical memory?**

To solve these problems, the IB protocol introduces the concept of MR.

### What Is MR?

MR stands for Memory Region. It refers to a region planned out in memory by the RDMA software layer and used to store sent and received data. In the IB protocol, after users allocate a memory region for storing data, they must call the API provided by the IB framework to register the MR before the RDMA NIC can access this memory region. As the following figure shows, an MR is simply a special memory region:

<figure><img src="https://pic4.zhimg.com/v2-55517804b2d8dcb43bfa13578638e4eb_b.jpg" alt=""><figcaption></figcaption></figure>

When describing the IB protocol, RDMA hardware is usually called **HCA (Host Channel Adapter)**. The IB protocol defines it as an IB device in a processor or I/O unit that can produce and consume packets. To stay consistent with the protocol, this note and later notes call the hardware component HCA.

### Why Register MR?

Now look at how MR solves the two problems raised at the beginning of this note.

#### 1. Register MR to Translate Virtual Addresses to Physical Addresses

Applications can only see virtual addresses, and they directly pass VAs to the HCA inside WQEs. This includes the local source VA and the remote destination VA. Modern CPUs have MMUs and page tables to translate between VA and PA. An HCA either connects directly to the bus or connects to the bus after address translation through an IOMMU/SMMU. It cannot directly understand the real physical memory address corresponding to the VA provided by the application.

Therefore, during MR registration, hardware creates and fills a VA-to-PA mapping table in memory. When needed, it can translate VA to PA by looking up this table. Here is a concrete example:

<figure><img src="https://pic3.zhimg.com/v2-f0c015985e54c7d0420882698b7f8702_b.jpg" alt=""><figcaption></figcaption></figure>

Assume the node on the left initiates an RDMA WRITE operation to the node on the right, meaning it directly writes data into the memory region of the right node. Assume both endpoints in the figure have already registered MRs. The MRs correspond to the "data Buffer" in the figure, and the VA->PA mapping tables have also been created.

* First, the local application posts a WQE to the HCA, telling the HCA the virtual address of the local buffer that stores the data to be sent, and the virtual address of the peer data buffer to be written.
* The local HCA queries the VA->PA mapping table, obtains the physical address of the data to be sent, fetches the data from memory, assembles packets, and sends them out.
* The peer HCA receives the packets and parses the destination VA from them.
* The peer HCA uses the VA->PA mapping table stored in local memory to find the real physical address. After verifying permissions, it places the data into memory.

Emphasize again: for the node on the right, **both address translation and memory writing are completed without CPU participation**.

#### 2. MR Controls HCA Memory-Access Permissions

Because memory addresses accessed by the HCA come from users, if a user passes an illegal address, such as system memory or memory used by another process, HCA reads or writes may cause information leakage or memory corruption. Therefore, a mechanism is needed to ensure that the HCA can only access authorized and safe memory addresses. In the IB protocol, the application must register MRs during the preparation phase for data exchange.

Registering an MR produces two keys: L_Key (Local Key) and R_Key (Remote Key). They are called keys, but their concrete representation is just a sequence of bits. They are used to protect access permissions for local and remote memory regions respectively. The following two figures describe the roles of L_Key and R_Key:

<figure><img src="https://pic3.zhimg.com/v2-a3ed0705600b6303396229bd1e81db7e_b.jpg" alt=""><figcaption></figcaption></figure>

<figure><img src="https://pic1.zhimg.com/v2-3ceba9b81100b02569caf8f8f4a87624_b.jpg" alt=""><figcaption></figcaption></figure>

Readers may ask: how does the local side know the available VA and corresponding R_Key of the peer node? In practice, before real RDMA communication, the two endpoint nodes establish a link through some method, such as a socket connection or CM connection, and exchange information required for RDMA communication over that link, including VA, Key, QPN, and so on. This process is called connection setup and handshake. Later notes will explain it in detail.

Besides the two points above, registering an MR has another important role.

#### 3. MR Prevents Paging

Physical memory is limited, so the operating system uses paging to temporarily save memory contents that a process is not using to disk. When the process needs them again, a page-fault interrupt brings the contents back from disk to memory. This process almost inevitably changes the VA-PA mapping relationship.

Because the HCA often bypasses the CPU and reads or writes the physical memory region pointed to by the user-provided VA, if the VA-PA mapping relationship changes, the VA->PA mapping table described above becomes meaningless. The HCA can no longer find the correct physical address.

To prevent paging from changing the VA-PA mapping relationship, MR registration "pins" this memory region, also called page locking, which locks the VA-PA mapping relationship. In other words, the memory region corresponding to the MR stays in physical memory and is not paged out until communication is complete and the user actively deregisters this MR.

At this point, the concept and role of MR have been introduced. The next note introduces PD (Protection Domain).
