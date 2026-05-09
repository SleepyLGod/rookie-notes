# 😋 Elements

RDMA uses many abbreviations, which can easily confuse newcomers. This note explains the most basic RDMA elements and their meanings.

I place the abbreviation table at the beginning. If you forget a term while reading, you can come back here and check it.

<figure><img src="https://pic1.zhimg.com/v2-b6723caa5b291ee161d94fd8fd8ce09c_b.jpg" alt=""><figcaption></figcaption></figure>

### WQ <a href="#h_141267386_0" id="h_141267386_0"></a>

WQ stands for Work Queue and is one of the most important concepts in RDMA. A WQ is a queue that stores work requests. To explain what WQ means, first introduce the element inside this queue: WQE (Work Queue Element).

#### WQE <a href="#h_141267386_1" id="h_141267386_1"></a>

A WQE can be understood as a "task description". This work request is posted by software to hardware. The description contains the task that software wants hardware to execute and the detailed information related to that task. For example, one task may say: "I want to send 10 bytes of data located at address `0x12345678` to the peer node." After hardware receives this task, it fetches data from memory through DMA, assembles a packet, and sends it.

The meaning of WQE should now be clear. So what is the WQ mentioned at the beginning? It is the "folder" used to store these task descriptions. A WQ can contain many WQEs. Readers with data-structure background know that a queue is a first-in-first-out data structure and is very common in computer systems. The relationship between WQ and WQE can be shown as follows:

<figure><img src="https://pic3.zhimg.com/v2-40c7e57f2760323c6b6665306e8f8896_b.png" alt=""><figcaption></figcaption></figure>

Software always adds WQEs into the WQ, meaning enqueue, and hardware removes WQEs from it. This is the process by which software "posts tasks" to hardware. Why use a queue rather than a stack? Because software and hardware are responsible for storing and fetching respectively, and user requests need to be processed in order. In RDMA, all communication requests are delivered to hardware in this way. This action is usually called "Post".

#### QP <a href="#h_141267386_2" id="h_141267386_2"></a>

QP stands for Queue Pair, meaning a "pair" of WQs.

#### SQ and RQ <a href="#h_141267386_3" id="h_141267386_3"></a>

Every communication process has a sender and a receiver. A QP is a combination of one send work queue and one receive work queue. These two queues are called SQ (Send Queue) and RQ (Receive Queue). Enriching the previous figure, the sender is on the left and the receiver is on the right:

<figure><img src="https://pic2.zhimg.com/v2-0dafce0772299d6930905373e0664929_b.jpg" alt=""><figcaption></figcaption></figure>

Where did WQ go? SQ and RQ are both WQs. WQ only represents a kind of unit that can store WQEs, while SQ and RQ are concrete instances.

SQ is used specifically to store send tasks, and RQ is used specifically to store receive tasks. In one SEND-RECV flow, the sender needs to place a WQE representing one send task into the SQ. Similarly, receiver-side software needs to post a WQE representing one receive task to hardware, so hardware knows where in memory to place the data after it is received. The Post operation mentioned above is called Post Send for SQ and Post Receive for RQ.

One important point is that in RDMA, **the basic communication unit is the QP**, not the node. As shown below, for each node, each process can use several QPs, and each local QP can be associated with a remote QP. Saying "node A sends data to node B" is not enough to describe one RDMA communication precisely. A more accurate statement is like "QP3 on node A sends data to QP4 on node C".

<figure><img src="https://pic2.zhimg.com/v2-71b3b17ef8aec45d74ef9e4a42a69201_b.jpg" alt=""><figcaption></figcaption></figure>

Each QP on each node has a unique identifier called QPN (Queue Pair Number). A QPN uniquely identifies a QP on one node.

#### SRQ <a href="#h_141267386_4" id="h_141267386_4"></a>

SRQ stands for Shared Receive Queue. The concept is easy to understand: when several QPs share the same RQ, we call that queue an SRQ. Later notes will show that RQs are used much less frequently than SQs, while every queue consumes memory resources. When many QPs are needed, SRQ can save memory. As shown below, QP2 through QP4 use the same RQ:

<figure><img src="https://pic3.zhimg.com/v2-4a21f2b1333877b4b0d97a1ca91d4096_b.jpg" alt=""><figcaption></figcaption></figure>

### CQ <a href="#h_141267386_5" id="h_141267386_5"></a>

CQ stands for Completion Queue. As with WQ, first introduce the element inside this queue: CQE (Completion Queue Element). CQE can be understood as the opposite of WQE. If a WQE is a "task description" posted by software to hardware, then a CQE is a "task report" returned by hardware to software after completing the task. A CQE describes whether a task was executed correctly or encountered an error, and if there was an error, what caused it.

CQ is the container that carries CQEs: a first-in-first-out queue. If we reverse the previous WQ/WQE diagram, we get the relationship between CQ and CQE:

<figure><img src="https://pic4.zhimg.com/v2-31f9a407ab66381fbc557d8acc5573cb_b.png" alt=""><figcaption></figcaption></figure>

Each CQE contains completion information for a certain WQE. Their relationship is shown below:

<figure><img src="https://pic2.zhimg.com/v2-701fa8eacb10c90c45b0241c75254a01_b.jpg" alt=""><figcaption></figcaption></figure>

Now put CQ and WQ, or QP, together and look at the software/hardware interaction during one SEND-RECV operation. The sequence numbers in the figure do not necessarily represent the actual timing:

> 2022/5/23: The figure below and the following list order were modified. The original second item, "receiver-side hardware obtains the task description from RQ and prepares to receive data", was moved after "the receiver receives data, verifies it, and replies to the sender with an ACK packet", and the description was updated. It is now item 6.\
> My mistake was treating RQ like SQ. RQ is a "passive receive" process: hardware consumes an RQ WQE only after receiving a Send packet, or a Write packet with immediate data. Thanks to [@connection-changes-the-world](https://www.zhihu.com/people/67ff85d690d09e9f6741c579512fe9a9) for the correction.

<figure><img src="https://pic4.zhimg.com/v2-a8d38721903672037b27cc7e49ecee03_b.jpg" alt=""><figcaption></figcaption></figure>

1. The receiver-side application posts one RECV task to the RQ in the form of a WQE.
2. The sender-side application posts one SEND task to the SQ in the form of a WQE.
3. Sender-side hardware obtains the task description from the SQ, fetches the data to be sent from memory, and assembles a packet.
4. The sender-side NIC sends the packet over the physical link to the receiver-side NIC.
5. The receiver receives the data, verifies it, and replies to the sender with an ACK packet.
6. Receiver-side hardware takes one task description, or WQE, from the RQ.
7. Receiver-side hardware places the data into the location specified by the WQE, then generates a "task report", or CQE, and places it into the CQ.
8. The receiver-side application obtains the task-completion information.
9. After the sender-side NIC receives the ACK, it generates a CQE and places it into the CQ.
10. The sender-side application obtains the task-completion information.

**NOTE: The example in the figure above is an interaction flow for a reliable service type. If it is an unreliable service, there is no ACK reply in step 6, and step 9 and later steps are triggered immediately after step 5. Service types and reliable/unreliable behavior are explained in** [**RDMA Basic Service Types**](https://zhuanlan.zhihu.com/p/144099636)**.**

At this point, through the two media WQ and CQ, the software and hardware on both sides jointly complete one send/receive process.

### WR and WC <a href="#h_141267386_6" id="h_141267386_6"></a>

After explaining several queues, two concepts mentioned at the beginning remain: WR and WC. WC here is not short for Water Closet.

WR stands for Work Request, and WC stands for Work Completion. They are essentially the user-layer "mappings" of WQE and CQE. Applications complete RDMA communication by calling protocol-stack interfaces. WQE and CQE themselves are not visible to the user; they are driver-level concepts. What users actually post through APIs is WR, and what users receive is WC.

WR/WC and WQE/CQE are the same concepts represented at different layers. They are still "task descriptions" and "task reports". Therefore, add some content to the previous two figures:

<figure><img src="https://pic2.zhimg.com/v2-00b87c111a8e1701f96fbfb78e078b29_b.jpg" alt=""><figcaption></figcaption></figure>

### Summary <a href="#h_141267386_7" id="h_141267386_7"></a>

Finally, summarize this note with Figure 11 from Section 3.2.1 of the IB protocol:

<figure><img src="https://pic4.zhimg.com/v2-2107a9bf8230c45ad73aa5ff0b8626ff_b.jpg" alt=""><figcaption></figcaption></figure>

The userspace WR is converted by the driver into a WQE and filled into a WQ. The WQ can be an SQ responsible for sending or an RQ responsible for receiving. Hardware takes WQEs from different WQs and completes send or receive tasks according to the requirements in the WQEs. After a task is complete, hardware generates a CQE for the task and fills it into the CQ. The driver takes the CQE from the CQ, converts it into a WC, and returns it to the user.

That is all for the basic concepts. The next note introduces several common RDMA operation types.
