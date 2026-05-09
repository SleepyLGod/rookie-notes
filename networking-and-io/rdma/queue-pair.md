# 😅 Queue Pair

This note explains more details about QP.

### Basic Concept Review

First, briefly review the basic knowledge of QP.

According to the IB protocol, a QP is a virtual interface between hardware and software. A QP is a queue structure that sequentially stores tasks, or WQEs, posted by software to hardware. A WQE contains information such as where to fetch data from, how much data to fetch, and which destination to send it to.

<figure><img src="https://pic1.zhimg.com/v2-1e8db2b2499ca6c838f3b5d91b440aac_b.jpg" alt=""><figcaption></figcaption></figure>

Each QP is independent from other QPs and is isolated through PDs. Therefore, a QP can be viewed as a resource exclusively owned by a user, and one user can also use multiple QPs at the same time.

QPs have many service types, including RC, UD, RD, UC, and others. The source QP and destination QP must be of the same type before they can exchange data.

Although the IB protocol calls QP a "virtual interface", it has concrete representations:

* In hardware, a QP is a storage region containing several WQEs. The IB NIC reads WQE content from this region and accesses memory according to the user's expectations. The IB protocol does not restrict whether this storage region is host memory or on-chip storage inside the IB NIC; each vendor can implement it differently.\

* In software, a QP is a data structure maintained by the IB NIC driver. It contains the QP address pointer and related software attributes.

### QPC

In [5. RDMA Basic Service Types](https://zhuanlan.zhihu.com/p/144099636), we mentioned that QPC stands for Queue Pair Context and is used to store QP-related attributes. The driver already stores QP software attributes, so why do we still need QPC?

The reason is that **QPC is mainly for hardware to read, and it is also used to synchronize QP information between software and hardware**.

As discussed above, the hardware representation of a QP is only a storage region. Aside from the starting address and size of this region, hardware knows nothing about it. It may not even know the service type of this QP. There is also other important information. For example, if a QP contains several WQEs, how does hardware know how many there are and which one should be processed now?

Software can design data structures and allocate memory for all of this information, but software sees virtual addresses, and these memory regions are physically scattered. Hardware does not know where the data is stored. Therefore, software needs the operating system to allocate a large contiguous region in advance, namely the QPC, to carry this information for hardware. The NIC and its driver agree in advance on what fields are contained in the QPC, how much space each field occupies, and in what order the fields are stored. In this way, the driver and hardware can read and write information such as QP state through the QPC region.

<figure><img src="https://pic2.zhimg.com/v2-d010221f5c05a699969184a197b251a5_b.jpg" alt=""><figcaption></figcaption></figure>

As shown in the figure, hardware only needs to know the QPC address `0x12350000`, because it can parse the QPC content and learn the QP location, QP number, QP size, and other information. It can then find the QP and know which WQE should be fetched for processing. Vendor implementations may differ, but the general principle is like this.

The IB software stack has many other Context concepts besides QPC, such as Device Context, SRQC, CQC, and EQC (Event Queue Context). Their roles are similar to QPC: they record and synchronize attributes of a particular resource.

### QP Number

QPN is short for QP Number, meaning the identifier of each QP. The IB protocol specifies that QPN is represented by 24 bits. That means each node can use up to `2^24` QPs at the same time, which is already a very large number and is almost impossible to exhaust. Each node maintains its own QPN set independently, so different nodes may have QPs with the same number.

The QPN concept itself is simple, but two special reserved numbers require extra attention.

#### QP0

QP number 0 is used for the Subnet Management Interface (SMI), which manages all nodes in a subnet. To be honest, I have not fully understood the role of this interface yet, so leave it aside for now.

#### QP1

QP number 1 is used for the General Service Interface (GSI). GSI is a set of management services, the best-known of which is CM (Communication Management). CM is a way for two communicating nodes to exchange required information before formally establishing a connection. Its details will be introduced in a later note.

This is why QP0 and QP1 never appeared in the QP diagrams in previous notes. All QPs other than these two are ordinary QPs. When the user creates a QP, the driver or hardware allocates a QPN to the new QP. Ordinary QPNs are usually assigned sequentially, such as 2, 3, 4, and so on. After a QP is destroyed, its QPN is recycled and assigned to another newly created QP when appropriate.

### User Interfaces

We introduce user interfaces from the control plane and data plane. The control plane refers to user configuration of a resource, usually before formal data transmission. The data plane refers to operations performed during real data send and receive.

#### Control Plane

Readers familiar with algorithms know that linked-list nodes involve four operations: create, delete, modify, and query. A linked-list node is a memory region and is a software resource.

"Create" means requesting a memory region from the operating system to store data. The system allocates a region in memory and marks it as "used by process XX", so other unauthorized processes cannot overwrite or even read this memory region.

"Delete" means notifying the operating system that this region is no longer used, so it can be marked as unused and made available to other processes.

"Modify" means writing, or changing the content of this memory region.

"Query" means reading, or obtaining the content of this memory region.

As one of the most important resources in RDMA, QP has a lifecycle similar to a linked-list node:

<figure><img src="https://pic3.zhimg.com/v2-62f578aee30bb5dd47249b9faaa3d7be_b.jpg" alt=""><figcaption></figcaption></figure>

These four operations are the control-plane interfaces provided by Verbs, the RDMA API exposed to upper-layer applications.

#### Create QP

Create the software and hardware resources of a QP, including the QP itself and the QPC. When creating a QP, the user passes a series of initialization attributes, including the QP service type and the number of WQEs it can store.

#### Destroy QP

Release all software and hardware resources of a QP, including the QP itself and the QPC. After destroying a QP, the user can no longer locate this QP through its QPN.

#### Modify QP

Modify certain attributes of a QP, such as QP state and path MTU. This modification process includes both software data-structure updates and QPC updates.

#### Query QP

Query the current state and attributes of a QP. The queried data comes from the driver and the QPC content.

These four operations all have corresponding Verbs interfaces, in forms such as `ibv_create_qp()`. When writing applications, we can call them directly. More details about upper-layer APIs will be introduced separately later.

#### Data Plane

On the data plane, a QP exposes only two kinds of interfaces to upper layers. They are used to fill send and receive requests into the QP. **Here, "send" and "receive" do not mean sending and receiving data directly. They refer to the initiator, or Requestor, and receiver, or Responder, of one communication process.**

In behavior, software fills a WQE into the QP. At the application layer this is called a WR. The request asks hardware to execute an action. Therefore, both behaviors are called "Post XXX Request", meaning posting an XXX request.

#### Post Send Request

To emphasize again, Post Send itself does not mean the WQE operation type is Send. It means the WQE belongs to the communication initiator. The WQE/WR filled into the QP through this flow may be a Send operation, RDMA Write operation, RDMA Read operation, and so on.

The user needs to prepare data buffers, destination addresses, and other information in advance, then call the interface to pass the WR to the driver. The driver then fills the WQE into the QP.

#### Post Receive Request

Post Recv is used less frequently. It is generally executed only by the receiver side of Send-Recv operations. The receiver needs to prepare the receive-data buffer in advance and provide information such as buffer address to hardware in the form of a WQE.

### QP State Machine

When discussing QP state, the following figure must be shown. It is taken from IB protocol Section 10.3.1:

<figure><img src="https://pic1.zhimg.com/v2-9368ea8d0bbb0b9a74bf8f3dcf4402d8_b.jpg" alt=""><figcaption></figcaption></figure>

A state machine describes the different states of an object and the conditions that trigger transitions between those states. Designing a state machine for an object makes its lifecycle very clear and also makes implementation logic clearer.

For QP, the IB specification also defines several states. QPs in different states have different capabilities. For example, only after entering Ready to Send can a QP perform Post Send data operations. Transitions between normal states, shown in green, are actively triggered by the user through the Modify QP interface introduced above. Error states, shown in red, are usually entered automatically after errors occur. Once a QP is in an error state, it can no longer execute normal business operations. Upper layers must reconfigure it back into a normal state through Modify QP.

In the figure above, we only focus on the QP portion. EE (End-to-End Context) is a concept used specifically for the RD service type, and it is not covered here. We enter this state diagram through the Create QP interface and leave it through the Destroy QP interface.

QP has the following states. Only the important points are introduced here:

#### RST (Reset)

Reset state. After a QP is created through Create QP, it is in this state. The related resources have already been allocated, but the QP cannot do anything yet. It cannot accept WQEs posted by the user, and it cannot accept messages from a peer QP.

#### INIT (Initialized)

Initialized state. In this state, the user can post Receive WRs to this QP through Post Receive, but received messages are not processed and are silently discarded. If the user posts a Post Send WR, an error is reported.

#### RTR (Ready to Receive)

Ready to Receive state. On top of INIT, the RQ can work normally. For received messages, data can be moved to the specified memory location according to the WQE. In this state, the SQ still cannot work.

#### RTS (Ready to Send)

Ready to Send state. On top of RTR, the SQ can work normally. The user can execute Post Send, and hardware sends data according to the SQ content. Before entering this state, the QP must have established a connection with the peer.

#### SQD (Send Queue Drain)

Send Queue Drain state. As the name suggests, this state drains all existing unprocessed WQEs in the SQ. At this time, the user can still post new WQEs, but these WQEs will not be processed until all old WQEs have been processed.

#### SQEr (Send Queue Error)

Send Queue Error state. When a completion error occurs for some Send WR, meaning hardware reports the error to the driver through a CQE, the QP enters this state.

#### ERR (Error)

Error state. If an error occurs in other states, the QP may enter this state. In Error state, the QP stops processing WQEs, and WQEs that are already halfway through processing also stop. Upper layers need to fix the error and then switch the QP back to the initial RST state.

### Summary

This note first reviewed several important basic concepts about QP, then explained closely related concepts such as QPC and QPN, and finally introduced common user interfaces for operating QPs and the QP state machine. After this note, readers should have a deeper understanding of QP.

As the core concept of RDMA, QP contains a lot of material, and this note cannot cover everything. Later notes will gradually fill in related topics. For example, the QKey concept will be explained in a future note dedicated to different keys.

That is all for this note. Thanks for reading. The next note will explain CQ in detail.

### Protocol Sections

3.5.1 and 10.2.4 Basic QP concepts

10.3 QP state machine

10.2.5 QP-related software interfaces

11.4 Post Send and Post Recv
