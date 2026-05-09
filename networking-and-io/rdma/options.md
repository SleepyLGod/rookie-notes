# 😂 Options

The previous RDMA communication-flow notes repeatedly discussed SEND-RECV. However, SEND-RECV is not fully "**RDMA**" in the strictest sense. It is more like an upgraded version of the traditional send/receive model with zero copy and protocol-stack offload. This operation type does not fully use RDMA's capabilities and is often used for scenarios such as exchanging control information between two endpoints. When large amounts of data need to be transmitted, two RDMA-specific operations are used more often: WRITE and READ.

First, review two-sided operations, namely SEND and RECV. Then compare them with one-sided operations, namely WRITE and READ.

### SEND & RECV

SEND and RECV are two different operation types. However, because if one side performs a SEND operation, the peer must perform a RECV operation, they are usually described together.

Why are they called "two-sided operations"? Because **both CPUs must participate to complete one communication process**, and the receiver must explicitly post a WQE in advance. The following figure shows one SEND-RECV operation. The original figure is from [1], with some modifications.

<figure><img src="https://pic3.zhimg.com/v2-0fd98c63ad73838ba55c9fa73917e2c2_b.jpg" alt=""><figcaption></figcaption></figure>

The previous note explained that upper-layer applications post tasks to hardware through WQEs, or WRs. In a SEND-RECV operation, not only the sender but also the receiver must post a WQE. The receiver-side WQE tells hardware where received data should be placed. The sender does not know where the sent data will be placed. Every time data is sent, the receiver must prepare a receive buffer in advance, and the receiver CPU naturally perceives this process.

To compare SEND/RECV with WRITE/READ below, add memory read/write steps to the previous SEND-RECV flow. In the figure below, step 5 is where sender-side hardware fetches data from memory according to the WQE and encapsulates it into packets that can be transmitted on the link. Step 7 is where receiver-side hardware parses the packet and places data into the specified memory region according to the WQE. Other steps are not repeated. Again, the sender and receiver steps do not necessarily happen in exactly the order shown in the figure. For example, the relative order of steps 8, 11, 12 and steps 9, 10 is not fixed.

<figure><img src="https://pic3.zhimg.com/v2-5496d59332aec3c2da73f009859c481e_b.jpg" alt=""><figcaption></figcaption></figure>

The next section introduces WRITE. The comparison should make the differences clearer.

### WRITE

WRITE, fully RDMA WRITE, is the local side actively writing into remote memory. Except for the preparation phase, the remote CPU does not need to participate and is not aware of when data is written or when data reception completes. Therefore, it is a one-sided operation.

The figure below compares WRITE with SEND-RECV. During the preparation phase, the local side obtains the **address** and **key** of an available remote memory region through data exchange. This is equivalent to obtaining read/write permission for that remote memory region. After obtaining permission, the local side can **directly read and write this remote memory region** as if it were accessing its own memory. This is the core meaning of RDMA: remote direct memory access.

How are the destination address and key in WRITE/READ operations obtained? Usually this can be done through the SEND-RECV operation just discussed, because obtaining the key must ultimately be allowed by the controller of the remote memory, namely the remote CPU. Although preparation is somewhat complex, once it is complete, RDMA can show its advantage for reading and writing large amounts of data. Once the remote CPU authorizes the local side to use the memory, it no longer participates in the data send/receive process. This frees the remote CPU and reduces communication latency.

<figure><img src="https://pic4.zhimg.com/v2-5a8bae1c63fa44ab66b2d06d136acdd7_b.jpg" alt=""><figcaption></figcaption></figure>

Note that the local side reads and writes remote memory through a **virtual address**, which is convenient for upper-layer applications. The actual virtual-to-physical address translation is completed by the RDMA NIC. How this translation works is introduced in later notes.

Ignoring the preparation phase where `key` and `addr` are obtained, the flow of one WRITE operation is described below. From this point onward, instead of calling the two sides "sender" and "receiver", use "requester" and "responder", which is more accurate for describing WRITE and READ operations and avoids ambiguity.

<figure><img src="https://pic3.zhimg.com/v2-95cdf471e117f8dccfee7e7115885c46_b.jpg" alt=""><figcaption></figcaption></figure>

1. The requester application posts one WRITE task in the form of a WQE, or WR.
2. Requester hardware takes the WQE from the SQ and parses its information.
3. The requester NIC translates the virtual address in the WQE to a physical address, fetches the data to be sent from memory, and assembles packets.
4. The requester NIC sends the packets over the physical link to the responder NIC.
5. The responder receives the packets, parses the destination virtual address, translates it to a local physical address, parses the data, and places the data into the specified memory region.
6. The responder replies with an ACK packet to the requester.
7. After the requester NIC receives the ACK, it generates a CQE and places it into the CQ.
8. The requester application obtains the task-completion information.

### READ

As the name suggests, READ is the opposite of WRITE. It is the local side actively reading remote memory. Like WRITE, the remote CPU does not need to participate and is not aware of the process in which data is read from memory.

The process of obtaining the key and virtual address is the same as for WRITE. Note that **the data requested by the "read" operation is carried in the response packet from the peer**.

The flow of one READ operation is described below. Compared with WRITE, the difference is only the direction and step order.

<figure><img src="https://pic1.zhimg.com/v2-3e576043a6b1a1e12993d0a813320768_b.jpg" alt=""><figcaption></figcaption></figure>

1. The requester application posts one READ task in the form of a WQE.
2. The requester NIC takes the WQE from the SQ and parses its information.
3. The requester NIC sends the READ request packet over the physical link to the responder NIC.
4. The responder receives the packet, parses the destination virtual address, translates it to a local physical address, parses the data, and fetches data from the specified memory region.
5. Responder hardware assembles the data into a response packet and sends it onto the physical link.
6. Requester hardware receives the packet, parses and extracts the data, and places it into the memory region specified by the READ WQE.
7. The requester NIC generates a CQE and places it into the CQ.
8. The requester application obtains the task-completion information.

#### Summary

Ignoring various details, RDMA WRITE and READ operations use the NIC to perform the memory-copy operation shown on the left below. The difference is that the copy is completed by the RDMA NIC over a network link. A local memory copy, shown on the right, is completed by the CPU over the bus:

<figure><img src="https://pic2.zhimg.com/v2-c9f2e041eac5ac5fdd7b584cfc191e21_b.jpg" alt=""><figcaption></figcaption></figure>

The words used by the RDMA standard to define these operations are very precise. "Send" and "receive" imply active peer participation, while "read" and "write" sound more like the local side operating on a peer that does not actively participate.

By comparing SEND/RECV with WRITE/READ, we can see that WRITE/READ, which does not require responder CPU participation during data transfer, has a greater advantage. The drawback is that the requester needs to obtain read/write permission for a remote memory region during the preparation phase. In actual data transmission, however, the power and time cost of this preparation phase can be ignored. Therefore, RDMA WRITE/READ are the operation types used for bulk data transfer, while SEND/RECV is usually used only to transmit control information.

Besides the operations introduced in this note, there are more complex operation types such as ATOMIC. They will be analyzed in detail in the later protocol-reading section. That is all for this note. The next note introduces basic RDMA service types.
