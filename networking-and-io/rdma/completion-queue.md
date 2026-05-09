# 😂 Completion Queue

### Basic Concepts <a href="#h_259650980_0" id="h_259650980_0"></a>

First, recall the role of a CQ. CQ means Completion Queue. Its role is the opposite of a WQ, meaning SQ and RQ: hardware uses CQEs/WCs in the CQ to tell software the completion status of a WQE/WR. As a reminder, upper-layer users usually call this object a WC, while drivers usually call it a CQE. This note does not distinguish between the two.

<figure><img src="https://pic2.zhimg.com/v2-017a227c8b78fab94a938a6870f00f1d_b.png" alt=""><figcaption></figcaption></figure>

A CQE can be understood as a "report" that describes the execution status of a task. It includes:

* Which WQE in which QP has completed, identified by QP Number and WR ID.
* What operation this task executed, namely the Opcode operation type.
* Whether the task succeeded or failed, and if it failed, the status and error code.
* ...

Every time hardware finishes processing one WQE, it generates a CQE and places it into the CQ. If the CQE corresponding to a WQE has not been generated, that WQE must still be considered unfinished. What does this mean?

* Operations that fetch data from memory, such as SEND and WRITE

Before the CQE is generated, hardware may not have sent the message yet, may be sending the message, or the peer may have received the correct message. Because the memory region is allocated before sending, upper-layer software must consider that memory region still in use until it receives the corresponding CQE. It must not release any related memory resources.

* Operations that place data into memory, such as RECV and READ

Before the CQE is generated, hardware may not have started writing data, may have written only half of the data, or may have detected a data-check error. Therefore, before upper-layer software obtains the CQE, the content of the memory region used to store received data is not trustworthy.

In short, the user can only consider a message send/receive task complete after obtaining the CQE and checking its contents.

#### When CQEs Are Generated <a href="#h_259650980_1" id="h_259650980_1"></a>

The timing and meaning of CQE generation differ by service type and operation type. This section only covers RC and UD. It is useful to review notes 4 and 5 before reading this part.

* Reliable service type (RC)

As discussed earlier, **reliable means the local side cares that the sent message is received correctly by the remote side**. This is guaranteed through mechanisms such as ACKs, checksums, and retransmission.

* SEND\
  A SEND operation requires hardware to fetch data from memory, assemble packets, and send them to the peer through the physical link. For SEND, CQE generation on the client side means **the peer has received the data correctly**. After peer hardware receives and verifies the data, it replies with an ACK packet to the sender. The sender generates a CQE only after receiving this ACK, thereby telling the user that the task completed successfully. As shown in the figure, the left client side generates the CQE for this task at the red dot.

<figure><img src="https://pic1.zhimg.com/v2-af0a569fdf5b78066a455b1da6037b6c_b.jpg" alt=""><figcaption></figcaption></figure>

* RECV\
  A RECV operation requires hardware to place received data into the memory region specified by the user's WQE. After completing validation and data placement, hardware generates a CQE, as shown on the server side in the figure above.\

* WRITE\
  For the client side, WRITE is similar to SEND: hardware fetches data from memory and waits for an ACK from the peer before generating a CQE. The difference is that WRITE is an RDMA operation. The peer CPU is unaware of it, and naturally the peer user is also unaware. Therefore, the previous figure becomes:

<figure><img src="https://pic1.zhimg.com/v2-8354562f3bd3eeba9deb0a7acb6b38d4_b.jpg" alt=""><figcaption></figcaption></figure>

* READ\
  READ is somewhat similar to RECV. After the client initiates a READ operation, the peer replies with the data we want to read. The local side validates the data and places it into the location specified by the WQE. After those actions complete, the local side generates a CQE. READ is also an RDMA operation, so the peer user is unaware of it and no CQE is generated on the peer side. In this case, the figure becomes:

<figure><img src="https://pic3.zhimg.com/v2-41245a836ef08d02be3df5c535a0f6aa_b.jpg" alt=""><figcaption></figcaption></figure>

* Unreliable service type (UD)

Because an unreliable service type has no retransmission or acknowledgment mechanism, CQE generation means hardware **has already sent out the data specified by the corresponding WQE**. As noted earlier, UD only supports SEND-RECV operations and does not support RDMA operations. Therefore, CQE generation timing on both sides of a UD service is shown below:

<figure><img src="https://pic3.zhimg.com/v2-4de94036dff67d4755705886a770182e_b.jpg" alt=""><figcaption></figcaption></figure>

#### Relationship Between WQ and CQ <a href="#h_259650980_2" id="h_259650980_2"></a>

**Every WQ must be associated with one CQ, while each CQ can be associated with multiple SQs and RQs**.

Here, "associated" means that all CQEs corresponding to WQEs in a WQ are placed by hardware into the bound CQ. Note that the SQ and RQ belonging to the same QP can each be associated with different CQs. As shown below, both the SQ and RQ of QP1 are associated with CQ1, while QP2's RQ is associated with CQ1 and its SQ is associated with CQ2.

<figure><img src="https://pic2.zhimg.com/v2-53f2a12d262db6813b6f85e462acc045_b.jpg" alt=""><figcaption></figcaption></figure>

Because every WQ must be associated with a CQ, the user needs to create CQs before creating QPs, and then specify which CQ the SQ and RQ will use.

**For WQEs in the same WQ, the corresponding CQEs are ordered**

> This subsection has been modified. The original text said "WQEs and their corresponding CQEs are not ordered", which could mislead readers. Thanks to the commenters [@old-biology-building-rabbit](https://www.zhihu.com/people/c7fd0f8e0a53fa9f5a27639d9fa3459a) and [@river-boating](https://www.zhihu.com/people/3f1ae153e0f53a4609df5749e72a2d30) for the correction. Topics such as sender-side WR processing order, transmission order, and receiver-side processing order will be clarified in a separate article later.

Hardware fetches WQEs from a WQ, either SQ or RQ, in FIFO order. When it places CQEs corresponding to WRs into the associated CQ, it also follows the order in which those WQEs were placed into the WQ. In simple terms, whoever enters the queue first completes first. The process is shown below:

<figure><img src="https://pic3.zhimg.com/v2-833c50d2a96e9e9158fe58176c8372da_b.jpg" alt=""><figcaption></figcaption></figure>

Note that SRQ usage and the RQ of the RD service type are unordered cases. This note does not expand on them.

**For WQEs in different WQs, the corresponding CQEs are not ordered**

As mentioned above, one CQ may be shared by multiple WQs. In this case, the CQE generation order corresponding to those WQEs cannot be guaranteed. The following figure illustrates this situation. The WQE numbers indicate posting order: 1 is posted first, and 6 is posted last.

<figure><img src="https://pic1.zhimg.com/v2-7135324c6de191ff9f510299144f7a28_b.jpg" alt=""><figcaption></figcaption></figure>

The description above also covers the case where "WQEs in the SQ and RQ of the same QP do not have ordered corresponding CQEs." This is easy to understand. SQ and RQ are two different directions: one handles actively initiated tasks, and the other handles passive receive tasks. They should not affect each other. Suppose a user first posts a Receive WQE and then posts a Send WQE to the same QP. It would make no sense for the local side to be unable to send a message to the peer just because the peer has not sent a message to the local side.

Since CQE generation order in this case is unrelated to WQE posting order, how do upper-layer applications and drivers know which WQE a received CQE corresponds to? The answer is simple: **the CQE indicates the identifier of its corresponding WQE**.

Also note that even when multiple WQs share one CQ, "for WQEs within the same WQ, their corresponding CQEs are ordered" is still guaranteed. In the figure above, CQEs corresponding to WQ1's WQEs 1, 3, and 4 must be generated in order, and the same is true for WQ2's WQEs 2, 5, and 6.

#### CQC <a href="#h_259650980_3" id="h_259650980_3"></a>

Like a QP, a CQ is only a memory queue used to store CQEs. Aside from the base address, hardware knows almost nothing about this memory region. Therefore, software and hardware must agree on the format in advance. The driver allocates memory and fills basic CQ information into this memory according to the agreed format, allowing hardware to read it. This memory is the CQC. The CQC contains information such as CQ capacity and the current CQE sequence number being processed. By slightly modifying the QPC figure, we can show the relationship between CQC and CQ:

<figure><img src="https://pic4.zhimg.com/v2-b1241e57f7ca7f556c1b5a040859fe3b_b.jpg" alt=""><figcaption></figcaption></figure>

#### CQN <a href="#h_259650980_4" id="h_259650980_4"></a>

CQ Number, or CQN, is the identifier of a CQ and is used to distinguish different CQs. CQ does not have special reserved numbers like QP0 and QP1, so this note does not discuss it further.

### Completion Errors <a href="#h_259650980_5" id="h_259650980_5"></a>

The IB protocol has three error types: immediate errors, completion errors, and asynchronous errors.

An immediate error means "stop the current operation immediately and return an error to the upper-layer user." A completion error means "return error information to the upper-layer user through a CQE." An asynchronous error means "report the error to the upper-layer user through an interrupt event." This may still sound abstract, so the following examples show when these error types are generated:

* The user passes an illegal opcode when posting Send, such as trying to use RDMA WRITE in UD mode.

Result: an immediate error is generated. Some vendors may generate a completion error in this situation.

In this case, the driver usually exits the post-send flow directly and returns an error code to the upper-layer user. Note that the WQE has not yet been issued to hardware at this point.

* The user posts a WQE whose operation type is SEND, but no ACK is received from the peer for a long time.

Result: a completion error is generated.

Because the WQE has already reached hardware, hardware generates the corresponding CQE. The CQE contains timeout and no-response error details.

* The user posts multiple WQEs, so hardware generates multiple CQEs, but software never removes CQEs from the CQ, causing CQ overflow.

Result: an asynchronous error is generated.

Because software never takes CQEs from the CQ, it naturally cannot obtain information from CQEs. At this point, the IB framework calls the event handler registered by software to notify the user to handle the current error.

These are all ways for the lower layer to report errors to upper-layer users; they differ only in when they are generated. The IB protocol specifies which reporting method should be used in different situations. For example, in the figure below, modifying an illegal parameter during Modify QP should return an immediate error.

<figure><img src="https://pic1.zhimg.com/v2-f5f415adf5938600b34fbc051124cbe4_b.png" alt=""><figcaption></figcaption></figure>

This note focuses on CQs. After introducing error types, we now focus on completion errors. Completion errors are reported by hardware by filling an error code into the CQE. One communication process involves both the requester and the responder, and the specific error cause can be local or remote. First, look at the stages where error detection occurs. The following figure redraws Figure 118 from the IB protocol:

<figure><img src="https://pic2.zhimg.com/v2-af9c2d762baaba1e97858d333e948209_b.jpg" alt=""><figcaption></figcaption></figure>

The requester has two error-detection points:

1. Local error detection

This checks the WQE in the SQ. If an error is detected, a CQE is generated directly from the local error-checking module into the CQ, and no data is sent to the responder. If there is no error, data is sent to the peer.

2. Remote error detection

This checks whether the responder's ACK is abnormal. ACK/NAK is generated by the peer after its local error-checking module performs detection. It contains whether the responder has an error and the specific error type. Regardless of whether the remote error-detection result has a problem, a CQE is generated into the CQ.

The responder has only one error-detection point:

1. Local error detection

In practice, this checks whether the peer packet has a problem. The IB protocol also calls it "local" error detection. If an error is detected, it is reflected in the ACK/NAK packet returned to the peer, and a local CQE is generated.

Note that ACK generation and remote error detection are only valid for connected service types. For unconnected service types, such as UD, the requester does not care whether the peer has received the packet, and the receiver does not generate ACKs. Therefore, after requester's local error detection, a CQE is always generated, regardless of whether there is a local error.

Several common completion errors are briefly introduced below:

* RC SQ completion errors
* Local Protection Error\
  Local protection-domain error. The MR for the data memory address specified in the local WQE is invalid, meaning the user is trying to use data from unregistered memory.\

* Remote Access Error\
  Remote permission error. The local side does not have permission to read or write the specified remote memory address.\

* Transport Retry Counter Exceeded Error\
  Retry-count-exceeded error. The peer never replies with a correct ACK, causing the local side to retransmit multiple times until the preset retry count is exceeded.\

* RC RQ completion errors\

* Local Access Error\
  Local access error. This means the peer tried to write into a memory region that it does not have permission to write.\

* Local Length Error\
  Local length error. The local RQ does not have enough space to receive the data sent by the peer.

For a full list of completion error types, see Section 10.10.3 of the IB specification.

### User Interfaces <a href="#h_259650980_6" id="h_259650980_6"></a>

As with QP, this section introduces the IB protocol's CQ-related interfaces exposed to upper layers from two perspectives: communication preparation, or control plane, and communication execution, or data plane.

#### Control Plane <a href="#h_259650980_7" id="h_259650980_7"></a>

As with QP, the control plane still follows the four operations: create, delete, modify, and query. However, for CQs, upper-layer users are resource users rather than resource managers. Users can only read data from a CQ, not write data to it. Therefore, the only configurable parameter exposed to users is the "CQ specification".

* Create: Create CQ

When creating a CQ, the user must specify the CQ specification, meaning how many CQEs it can store. The user can also provide a callback function pointer that is invoked after CQE generation, which is discussed below. The kernel-mode driver configures other related parameters and fills them into the CQC agreed with hardware.

* Destroy: Destroy CQ

This releases CQ software and hardware resources, including the CQ itself and the CQC. The CQN naturally becomes invalid as well.

* Modify: Resize CQ

The name is slightly different here. Because users can only modify the CQ size, the operation is called Resize rather than Modify.

* Query: Query CQ

This queries the current CQ specification and the callback function pointer used for notification.

> By comparing RDMA specifications with software protocol stacks, one can see that many verbs interfaces are not implemented exactly according to the specification. Therefore, readers should not be surprised if software APIs differ from the protocol. RDMA technology is still evolving, and software frameworks are also actively changing. If you care more about programming implementation, use the software protocol-stack API documentation as the source of truth. If you care more about academic research, use the RDMA specification as the source of truth.

#### Data Plane <a href="#h_259650980_8" id="h_259650980_8"></a>

CQE is the medium through which hardware passes information to software. Although software knows under what conditions CQEs are generated, it does not know exactly when hardware will place a CQE into the CQ. In communication and computer systems, this pattern, where the receiver does not know when the sender will send, is called "asynchronous". First consider a NIC example, and then look at how users obtain CQEs, or WCs, through data-plane interfaces.

When a NIC receives packets, how does it tell the CPU and trigger packet processing? There are two common modes:

* Interrupt mode

When the amount of data is small, or when occasional data exchange is common, interrupt mode is suitable. The CPU usually does other work. When the NIC receives a packet, it raises an interrupt and interrupts the CPU's current task. The CPU then processes the packet, for example by parsing each layer of the TCP/IP stack. After processing the data, the CPU returns to the task it was executing before the interrupt.

Each interrupt must save context, meaning values of registers, local variables, and other state are saved to the stack and restored later. This itself has overhead. If the workload is heavy and the NIC continuously receives packets, the CPU keeps receiving interrupts and remains busy with interrupt switching, causing other tasks not to be scheduled.

* Polling mode

In addition to interrupt mode, NICs also support polling mode. After packets are received, they are placed into a buffer first. The CPU checks periodically whether the NIC has received data. If there is data, it takes a batch of data from the buffer for processing. If not, it continues processing other tasks.

By comparing with interrupt mode, we can see that polling mode requires the CPU to check periodically, which introduces some overhead. However, when the workload is busy, polling mode can greatly reduce interrupt-context switches and therefore reduce CPU burden.

Modern NICs generally use interrupt plus polling, dynamically switching according to workload.

In RDMA, a CQE is analogous to a packet received by a NIC: RDMA hardware passes it to the CPU for processing. The RDMA framework defines two upper-layer interfaces, poll and notify, corresponding to polling and interrupt modes.

#### Poll Completion Queue <a href="#h_259650980_9" id="h_259650980_9"></a>

The meaning of poll is straightforward: polling. After the user calls this interface, the CPU periodically checks whether the CQ contains fresh CQEs. If so, it takes out the CQE. Note that once the CQE is taken out, it is consumed. The software then parses its information and returns it to the upper-layer user.

#### Request Completion Notification <a href="#h_259650980_10" id="h_259650980_10"></a>

Request completion notification means that after the user calls this interface, it effectively registers an interrupt with the system. When hardware places a CQE into the CQ, it immediately triggers an interrupt to the CPU. The CPU stops its current work, takes out the CQE, processes it, and returns the result to the user.

Which of these two interfaces should be used depends on the user's real-time requirements and the actual workload intensity.

Thanks for reading. This is the end of the CQ introduction. The next note will discuss SRQ in detail.

### Protocol Sections <a href="#h_259650980_11" id="h_259650980_11"></a>

9.9 CQ error detection and recovery

10.2.6 Relationship between CQ and WQ

10.10 Error types and handling

11.2.8 CQ-related control-plane interfaces

11.4.2 CQ-related data-plane interfaces

### Other References <a href="#h_259650980_12" id="h_259650980_12"></a>

\[1] Linux Kernel Networking - Implement and Theory. Chapter 13. Completion Queue
