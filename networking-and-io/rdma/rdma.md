---
description: Introductory notes on RDMA
---

# 😍 Introduction

RDMA (Remote Direct Memory Access) is a networking technology designed to reduce server-side data-processing latency during network transmission. It transfers data directly from the memory of one computer to the memory of another computer without involving the operating systems on either side in the data path. This enables high-throughput and low-latency network communication, especially in large-scale parallel computing clusters.

RDMA sends data directly into a computer's memory region over the network, quickly moving data from one system into the memory of a remote system with minimal operating-system involvement. As a result, it consumes much less host CPU processing power. It removes the overhead of extra memory copies and context switches, freeing memory bandwidth and CPU cycles to improve application performance.

This note introduces RDMA from three perspectives: RDMA background, related work, and RDMA technical details.

### 1. Background

![](https://tjcug.github.io/blog/images/pasted-59.png)

#### 1.1 Traditional TCP/IP Communication Model

In traditional TCP/IP network communication, data is transferred from the sender's user space to the receiver's user space through the kernel and the network protocol stack.

**Sender side:**

Data is copied from the user-application buffer into the kernel-space socket buffer.

The kernel then adds packet headers and encapsulates the data.

After a series of packet-processing steps through multiple network protocols, including TCP, UDP, IP, ICMP, and others, the data is pushed into the NIC buffer for network transmission.

**Receiver side:**

After receiving packets from the remote machine, the receiver copies data from the NIC buffer into the socket buffer.

The kernel then parses the packets through the multi-layer network protocol stack.

The parsed data is copied into the corresponding user-application buffer.

Only after that does the system perform a context switch and invoke the user application.

This is the traditional TCP/IP protocol-layer workflow.

![](https://tjcug.github.io/blog/images/pasted-60.png)

As systems evolve, we increasingly need faster and lighter-weight network communication.

#### 1.2 Network Communication Metrics

The two most important metrics in computer network communication are **high bandwidth** and **low latency**.

Communication latency mainly consists of processing latency and network transmission latency.

Processing latency is the time spent processing a message on the send and receive sides.

Network transmission latency is the time spent transmitting the message over the network between sender and receiver. When network conditions are good, the network itself can provide high bandwidth and low latency.

#### 1.3 Current Network Workloads

Message communication can be roughly divided into two categories:

The first category is `large messages`. In this type of communication, **network transmission latency** dominates the total communication cost.

The second category is `small messages`. In this type of communication, sender-side and receiver-side processing overhead dominates the total communication cost.

In real computer-network workloads, however, communication is mostly dominated by small messages. Therefore, message send and receive processing overhead becomes the dominant factor.

More specifically, **processing overhead** includes buffer management, message copying across different memory spaces, and system interrupts after message transmission completes.

#### 1.4 Problems in Traditional TCP/IP

The main problem with traditional TCP/IP is the `I/O bottleneck`. Under high-speed network conditions, high host-side processing overhead related to network I/O limits the bandwidth that can be delivered between machines.

This high overhead mainly comes from data movement and data copying.

More specifically, traditional TCP/IP network communication sends messages through the **kernel**.

`Message passing through the kernel` leads to low performance and low flexibility:

Performance is low because kernel-mediated network communication introduces high data-movement and data-copying overhead. Modern memory bandwidth is also very different from CPU bandwidth and network bandwidth, making these copies especially expensive.

Flexibility is low because all network communication protocols pass through the kernel. This makes it difficult to support new network protocols, new messaging protocols, and new send/receive interfaces.

### 2. Related Work

The history of high-performance network communication mainly includes four directions: `TCP Offloading Engine (TOE)`, `User-Net Networking (U-Net)`, `Virtual Interface Architecture (VIA)`, and `Remote Direct Memory Access (RDMA)`. U-Net was one of the first models to bypass the kernel in network communication. VIA first proposed a standardized user-level networking model, combining the U-Net interface with remote DMA devices. RDMA is the modern high-performance networking technology that follows this line of work.

#### 2.1 TCP Offloading Engine

When hosts communicate over a network, the host processor spends substantial resources processing packets through multiple network protocols, including TCP, UDP, IP, ICMP, and others. Because the CPU performs heavy packet encapsulation and protocol processing, TOE (TCP/IP Offloading Engine) was introduced to move this work from the host processor to the network card, freeing CPU resources for other applications.

This technology requires a specific network interface card that supports offloading. **Such a NIC can encapsulate packets for multiple network protocol layers**. This capability is common on high-speed Ethernet interfaces, such as Gigabit Ethernet (GbE) and 10 Gigabit Ethernet (10GbE).

#### 2.2 User-Net Networking (U-Net)

The design goal of U-Net is to **move protocol processing into user space**. This avoids the cost of moving and copying data from user space into kernel space. Its core idea is to move the entire protocol stack into user space and remove the kernel completely from the data-communication path. This design improves both performance and flexibility.

![](https://tjcug.github.io/blog/images/pasted-61.png)

U-Net's virtual NI gives each process the illusion that it owns a network interface. The kernel interface is only involved during connection setup. In traditional networking, the kernel controls all network communication, and every communication operation must pass through the kernel. A U-Net application can access the network directly through the MUX, without moving and copying data into kernel space.

### 3. RDMA Details

RDMA (Remote Direct Memory Access) is a technology designed to reduce server-side data-processing latency during network transmission.

RDMA sends data directly into a computer's **memory region** over the network, rapidly moving data from one system into the **memory of a remote system** without materially affecting the operating system. This greatly reduces the amount of host processing required. It removes the overhead of extra memory copies and context switches, freeing memory bandwidth and CPU cycles to improve application performance.

![](https://tjcug.github.io/blog/images/pasted-62.png)

RDMA mainly has the following three characteristics:

1. Low latency
2. Low CPU overhead
3. High bandwidth

#### 3.1 RDMA Overview

Remote: data is transferred over the network between local and remote machines.

Direct: there is no kernel involvement in the data path; **the work related to send and transfer is offloaded to the NIC**.

Memory: data is transferred **directly** between user-space virtual memory and the RNIC without involving the system kernel or performing additional data movement and copying.

Access: send, receive, read, write, and atomic operations.

#### 3.2 RDMA Concepts and Terms

**3.2.1 Basic Terms**

<mark style="color:purple;">1</mark> **Fabric**

A fabric is a local-area network that supports RDMA.

```
A local-area RDMA network is usually referred to as a fabric.
```

<mark style="color:purple;">2</mark> **CA (Channel Adapter)**

CA stands for Channel Adapter. A CA is the hardware component that connects a system to the fabric.

In IBTA terminology, a CA is an end node in an InfiniBand subnet.

There are two types: `HCA` and `TCA`. Together they are often called `xCA`.

An `HCA (Host Channel Adapter)` is a CA that supports the verbs interface. A `TCA (Target Channel Adapter)` can be understood as a "weak CA" and does not need to support as many functions as an HCA.

In IEEE/IETF terminology, the CA concept is materialized as an `RNIC (RDMA Network Interface Card)`. In iWARP, a CA is called an RNIC.

**In short, in the IBTA ecosystem, a CA is an `HCA` or `TCA`; in the iWARP ecosystem, a CA is an `RNIC`. Whether it is an `HCA`, `TCA`, or `RNIC`, it is still a CA. Its basic function is essentially to produce or consume packets.**

```
A channel adapter is the hardware component that connects a system to the fabric.
```

<mark style="color:purple;">3</mark> **Verbs**

During RDMA's evolution, the OpenFabrics Alliance made important contributions. The word "verbs" is difficult to translate directly. It can be roughly understood as a standard set of actions for accessing RDMA hardware. Each verb can be understood as a function.

**3.2.2 Core Concepts**

<mark style="color:purple;">1</mark> **Memory Registration (MR)**

RDMA is used to transfer data between memory regions. How can memory be used for such transfers?

The answer is simple: **registration**.

RDMA hardware has special requirements for memory used in data transfer.

* During data transfer, the application must not modify the **memory** being used by the RDMA operation.
* The operating system must not `page out` the **memory** containing the data. The mapping between virtual addresses and physical pages must remain stable.

More precisely, DMA and RDMA engines require stable physical-address translations, and hardware usually consumes either physical addresses or scatter-gather descriptors. Memory registration pins the memory and records the address-translation information required by the RNIC.

How is memory registered?

* Create two **`key`s**, `local` and `remote`, that refer to the memory region to be operated on.
* The registered `key`s become part of the data-transfer request.

After a `Memory Region` is registered, it has its own attributes:

* `context`: RDMA operation context
* `addr`: registered buffer address of the MR
* `length`: registered buffer length of the MR
* `lkey`: local key of the registered MR
* `rkey`: remote key of the registered MR

About `Memory Registration`:

`Memory Registration` is a memory-protection mechanism in RDMA. Only after a memory region is registered as an RDMA `Memory Region` is the memory handed over to the RDMA protection domain for RDMA operations.

At this point, operations can be performed on the memory region. The starting address and buffer length are determined by the program's specific needs. The main requirement is that the receiver's buffer length must be **greater than or equal to** the sender's buffer length.

<mark style="color:purple;">2</mark> **Queues**

RDMA provides **message-queue-based point-to-point communication**. Each application can directly obtain its own messages without operating-system or protocol-stack intervention.

RDMA supports three queues. These queues manage different types of messages.

The three queues are the Send Queue (**SQ**), Receive Queue (**RQ**), and Completion Queue (**CQ**). SQ and RQ are usually created as a pair called a Queue Pair (**QP**):

1. Each application can have many QPs and CQs.
2. Each QP contains one SQ and one RQ.
3. Each CQ can be associated with multiple SQs or RQs.

The messaging service is built on a **Channel-IO** connection created between the local and remote applications. When an application needs to communicate, it creates a channel connection. The two endpoints of each channel are two QPs. QPs are mapped into the application's virtual address space, allowing the application to access the RNIC directly through them.

RDMA is a message-based transport protocol, and all data transfers are asynchronous. An RDMA operation can be understood as follows:

1.  The host submits a Work Request (**WR**) to a Work Queue (**WQ**). The WR describes the message that the application wants to transfer to the remote endpoint of the channel, and it targets one work queue in the QP. The WQ can be either an SQ or an RQ.

    Each element in a work queue is called a Work Queue Element (**WQE**), which is the format into which a WR is converted inside the WQ.

    The WQE waits for asynchronous scheduling and parsing by the RNIC. The RNIC then obtains the actual message from the buffer pointed to by the WQE and sends it to the remote endpoint of the channel.
2. The host obtains Work Completion (**WC**) from the Completion Queue (CQ). The CQ notifies the user that messages on the WQ have been processed. Each element in the CQ is called a Completion Queue Element (**CQE**), which is the same object represented to user space as a WC.
3.  Hardware with an RDMA engine is a queue-element processor. RDMA hardware continuously fetches Work Requests (WRs) from Work Queues (WQs), executes them, and places Work Completions (WCs) into Completion Queues (CQs).

    From a **producer-consumer** perspective:

    1. The host produces WRs and places them into the WQ.
    2. RDMA hardware consumes WRs.
    3. RDMA hardware produces WCs and places them into the CQ.
    4. The host consumes WCs.

![img](https://s2.loli.net/2022/08/25/RvNoKOmFsAif4J6.jpg)

<figure><img src="../../.gitbook/assets/image (2) (2).png" alt=""><figcaption></figcaption></figure>

#### **3.2.3 Other Concepts**

RDMA has two basic operation classes:

* `Memory verbs`: RDMA read, write, and atomic operations. These operations **operate on specified remote addresses and bypass the receiver's CPU**.
* `Messaging verbs`: RDMA send and receive operations. These operations **involve the responder's CPU; the sent data is written to an address previously specified by a receive posted by the responder's CPU**.

RDMA transports can be reliable or unreliable, and they can be connected or unconnected datagram transports.

With reliable transport, the NIC uses acknowledgments to guarantee in-order message delivery. Unreliable transport does not provide this guarantee.

However, modern RDMA implementations such as InfiniBand use a lossless link layer. Link-level flow control prevents congestion-based loss, and link-level retransmission handles bit-error-based loss. Therefore, unreliable transports rarely drop packets in practice.

**Current RDMA hardware provides one datagram transport: Unreliable Datagram (UD), and UD does not support memory verbs.**

![](https://tjcug.github.io/blog/images/pasted-63.png)

Detailed **data transfer** operations:

<mark style="color:purple;">1</mark> **RDMA Send / Receive (Send/Recv)**

This is similar to TCP/IP send/recv. The difference is that RDMA is a **message-based data-transfer protocol**, not a byte-stream protocol. Packet assembly is completed by RDMA hardware. In other words, the lower four layers of the OSI model, transport, network, data link, and physical layers, are handled by RDMA hardware.

<mark style="color:purple;">2</mark> **RDMA Read (Pull)**

An RDMA read is essentially a pull operation. It pulls data from remote system memory back into local system memory.

<mark style="color:purple;">3</mark> **RDMA Write (Push)**

An RDMA write is essentially a push operation. It pushes data from local system memory into remote system memory.

<mark style="color:purple;">4</mark> **RDMA Write with Immediate Data**

RDMA write with immediate data is essentially a push of out-of-band (OOB) data to the remote system, similar to out-of-band data in TCP.

Optionally, a 4-byte immediate value can be sent together with the data buffer.

This value is presented to the receiver as part of the receive notification and is not included in the data buffer itself.

#### 3.3 Three RDMA Hardware Implementations

There are currently three different RDMA **hardware implementations**: `InfiniBand`, `iWARP (Internet Wide Area RDMA Protocol)`, and `RoCE (RDMA over Converged Ethernet)`.

![](https://tjcug.github.io/blog/images/pasted-64.png)

Broadly speaking, there are three types of RDMA **networks**: `InfiniBand`, `RoCE`, and `iWARP`.

`InfiniBand` is a network designed specifically for RDMA. It guarantees reliable transmission at the **hardware level** and supports a next-generation network protocol for RDMA. Because it is a separate network technology, it requires NICs and switches that support InfiniBand.

`RoCE` and `iWARP` are Ethernet-based RDMA technologies that support the corresponding verbs interface, as shown in the figure above.

<mark style="color:blue;background-color:blue;">**`RoCE`**</mark> is a network protocol that allows RDMA over Ethernet. Its lower network header is an Ethernet header, while its higher network header, including the data, is an InfiniBand header. This allows RDMA to run over standard Ethernet infrastructure such as switches. Only the NIC needs to be special and support RoCE. As the figure shows, the `RoCE` protocol has two versions: `RoCEv1` and `RoCEv2`. The main difference is that `RoCEv1` implements RDMA at the Ethernet **link layer**. In practice it relies on lossless Ethernet features such as PFC to avoid packet loss. `RoCEv2` runs over UDP/IP, so it is routable at Layer 3, but production deployments still typically require a carefully configured low-loss or lossless Ethernet fabric.

`iWARP` is a network protocol that allows RDMA over TCP. Some functions available in InfiniBand and RoCE are not supported in iWARP. It supports RDMA over standard Ethernet infrastructure such as switches. Only the NIC needs to be special and support iWARP if CPU offload is used. Otherwise, the whole iWARP stack can be implemented in software, but that loses most of RDMA's performance advantages.

In terms of performance, `InfiniBand` is clearly the strongest network, but its NICs and switches are expensive. `RoCEv2` and `iWARP` require only special NICs and are therefore much cheaper.

![](https://tjcug.github.io/blog/images/pasted-65.png)

![](https://tjcug.github.io/blog/images/pasted-66.png)

#### 3.4 RDMA Technology

![](https://tjcug.github.io/blog/images/pasted-67.png)

Traditional network communication requires the kernel to encapsulate multiple network protocol layers and participate in data transfer. RDMA uses a dedicated RDMA NIC, the **RNIC**, to bypass the kernel and access an RDMA-enabled NIC directly from user space. RDMA exposes a dedicated verbs interface rather than the traditional TCP/IP socket interface.

To use RDMA, an application must **first establish a data path between the RDMA hardware and application memory**. These data paths can be established through the RDMA verbs interface. Once the data path is established, the hardware can directly access user-space buffers.

#### 3.5 Overall RDMA System Architecture

![RDMA overall architecture](https://tjcug.github.io/blog/images/pasted-68.png)

The figure shows that RDMA provides a series of verbs interfaces in application user space for operating RDMA hardware.

RDMA bypasses the kernel and accesses the RDMA NIC (RNIC) directly from user space.

The RNIC contains cached page table entries. Page tables are used to map virtual pages to corresponding physical pages.

#### 3.6 RDMA Workflow

RDMA works as follows:

1.  When an application issues an RDMA read or write request, no data copy is performed.

    Without requiring any kernel-memory participation, the RDMA request is sent from the user-space application to the **local NIC**.
2. The NIC reads the buffer contents and transmits them over the network to the **remote NIC**.
3.  The RDMA message transmitted over the network **contains** the target virtual address, memory key, and the data itself.

    The request can either be processed completely in user space through polling a user-level completion queue, or be handled through a system interrupt when the application sleeps until the request completes.

    RDMA operations allow an application to read data from, or write data to, the memory of a remote application.
4.  The target NIC validates the memory key and writes the data directly into the application buffer.

    The remote virtual memory address used for the operation is included in the RDMA message.

#### 3.7 RDMA Operation Details

**One-sided operations** are the largest difference between RDMA and traditional network transfer. They only require direct access to a remote virtual address and do not require participation from the remote application. This mode is suitable for bulk data transfer.

READ and WRITE are one-sided operations. The local side only needs clear information about the **source and destination addresses**. The remote application does not need to be aware of the communication. Data reads or writes are completed by RDMA between the RNIC and the application buffer, and the remote RNIC packages the result message back to the local side.

**Two-sided operations** are similar to the lower-level buffer-pool model in traditional networking. The participation process of sender and receiver is not fundamentally different. The difference lies in zero copy and kernel bypass. In RDMA, this is a relatively complex message-transfer mode and is often used for short control messages.

**3.7.1 RDMA One-Sided Operation (RDMA READ)**

For a one-sided operation, using storage in a storage-network environment as an example, the data flow is:

1. First, A and B establish a connection. The QP has already been created and initialized.
2. The data is stored at buffer address `VB` on B. Note that `VB` should be registered with B's RNIC in advance, and it is a `Memory Region`. Registration returns a `local key`, which is equivalent to permission for RDMA to operate on this buffer.
3. B packages data address `VB` and the `key` into a dedicated message and sends it to A. This is equivalent to B handing the operation rights of its data buffer to A. At the same time, B registers a WR in its WQ to receive the status returned by A after the data transfer.
4. After A receives `VB` and the `key` from B, A's RNIC packages them together with A's own storage address `VA` into an `RDMA READ` request and sends the request to B. During this process, neither A nor B needs software participation. The data from B can be stored into A's virtual address `VA`.
5. After storage completes, B returns the status information of the entire data transfer to A.

**3.7.2 RDMA One-Sided Operation (RDMA WRITE)**

For a one-sided operation, using storage in a storage-network environment as an example, the data flow is:

1. First, A and B establish a connection. The QP has already been created and initialized.
2. The remote target storage buffer address is `VB`. Note that `VB` should be registered with B's RNIC in advance, and it is a `Memory Region`. Registration returns a `local key`, which is equivalent to permission for RDMA to operate on this buffer.
3. B packages data address `VB` and the `key` into a dedicated message and sends it to A. This is equivalent to B handing the operation rights of its data buffer to A. At the same time, B registers a WR in its WQ to receive the status returned by A after the data transfer.
4. After A receives `VB` and the `key` from B, A's RNIC packages them together with A's own send address `VA` into an `RDMA WRITE` request. During this process, neither A nor B needs software participation. A's data can be sent to B's virtual address `VB`.
5. After A finishes sending the data, it returns the status information of the entire data transfer to B.

**3.7.3 RDMA Two-Sided Operation (RDMA SEND/RECEIVE)**

In RDMA, SEND/RECEIVE is a two-sided operation. The remote application must be aware of and participate in the operation for send and receive to complete.

In practice, SEND/RECEIVE is often used for connection **control messages**, while **data messages** are usually transferred through READ/WRITE.\
For a two-sided operation, using host A sending data to host B as an example:

1. First, both A and B create and initialize their own QPs and CQs.
2. A and B each post WQEs to their own WQs. For A, WQ = SQ, and the WQE describes data waiting to be sent. For B, WQ = RQ, and the WQE describes a buffer used to store received data.
3. A's RNIC asynchronously schedules A's WQE, parses it as a SEND message, and sends data directly from the buffer to B. After the data flow reaches B's RNIC, B's WQE is consumed, and the data is stored directly into the memory location pointed to by that WQE.
4. After A and B complete communication, A's CQ receives a CQE indicating send completion. At the same time, B's CQ also receives a CQE indicating receive completion. Every completed WQE in a WQ produces a CQE.
