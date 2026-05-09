# 😆 Shared Receive Queue

This note introduces more details about SRQ.

### Basic Concepts <a href="#h_279904125_0" id="h_279904125_0"></a>

#### What Is an SRQ? <a href="#h_279904125_1" id="h_279904125_1"></a>

SRQ stands for Shared Receive Queue. As the name suggests, it is a shared receive queue. We know that the basic unit of RDMA communication is the QP, and each QP consists of one Send Queue (SQ) and one Receive Queue (RQ).

SRQ is designed by the IB protocol to save resources on the receiver side. We can share one RQ across all associated QPs, and this shared RQ is called an SRQ. When any associated QP wants to post a receive WQE, the WQE is placed into this SRQ. Whenever hardware receives data, it uses the next WQE in the SRQ to place the data into the specified location.

<figure><img src="https://pic1.zhimg.com/v2-c53be9fe2b3e05db71b808d05a5f7a30_b.jpg" alt=""><figcaption></figcaption></figure>

#### Why Use SRQ? <a href="#h_279904125_2" id="h_279904125_2"></a>

Usually, we post far more tasks to the SQ than to the RQ. Why? First recall which operation types use the SQ and which use the RQ.

SEND, WRITE, and READ all require the communication initiator to post a WR to the SQ. Only the RECV operation paired with SEND requires the communication responder to post a WR to the RQ. A Write with immediate operation also consumes a Receive WR, but that has not been discussed yet. We also know that the SEND-RECV pair is usually used to transmit control information, while WRITE and READ are the main operations for large-scale remote memory reads and writes. Therefore, SQ usage is naturally much higher than RQ usage.

Every queue has a physical representation and consumes both host memory and on-chip storage on the NIC. In commercial scenarios, the number of QPs may reach hundreds of thousands or more, which creates high memory-capacity requirements. Memory costs real money, so SRQ is an IB protocol mechanism designed to save user memory.

Here is the official explanation from the specification about why SRQ is needed, from Section 10.2.9.1:

> Without SRQ, an RC, UC or UD Consumer must post the number of receive WRs necessary to handle incoming receives on a given QP. If the Consumer cannot predict the incoming rate on a given QP, because, for example, the connection has a bursty nature, the Consumer must either: post a sufficient number of RQ WRs to handle the highest incoming rate for each connection, or, for RC, let message flow control cause the remote sender to back off until local Consumer posts more WRs.\
> • Posting sufficient WRs on each QP to hold the possible incoming rate, wastes WQEs, and the associated Data Segments, when the Receive Queue is inactive. Furthermore, the HCA doesn’t provide a way of reclaiming these WQEs for use on other connections.\
> • Letting the RC message flow control cause the remote sender to back off can add unnecessary latencies, specially if the local Consumer is unaware that the RQ is starving.\
>

In simple terms, without SRQ, the receiver of an RC/UC/UD service does not know when the peer will send data or how much data will arrive. Therefore, it must prepare for the worst case and be ready for bursty arrivals by posting enough receive WQEs to the RQ. In addition, the RC service type can use flow control to backpressure the sender, effectively telling the peer: "I do not have enough RQ WQEs here." The sender then slows down or temporarily stops sending.

However, as discussed above, the first method prepares for the worst case. Most of the time, many RQ WQEs remain idle and unused, which wastes a large amount of memory. The second method avoids posting so many RQ WQEs, but flow control has a cost: it increases communication latency.

SRQ solves the problem above by allowing many QPs to share receive WQEs, as well as the memory space used to store data. When any QP receives a message, hardware takes one WQE from the SRQ, stores the received data according to that WQE, and then returns completion information for the receive task to the corresponding upper-layer user through the Completion Queue.

Now compare how much memory SRQ can save compared with ordinary RQ usage [1]:

Assume that the receiving node has N pairs of QPs, and each QP may randomly receive M consecutive messages, with each message consuming one WQE in the RQ.

* Without SRQ, the user needs to post `N * M` RQ WQEs in total.
* With SRQ, the user only needs to post `K * M` RQ WQEs, where K is much smaller than N.

The value of K can be configured by the user according to the workload. If many concurrent receives may happen, K should be set larger. Otherwise, a single-digit K is often enough for ordinary cases.

In total, we save `(N - K) * M` RQ WQEs. The RQ WQE itself is not very large, roughly a few KB, so it may look like it does not save much memory. But as mentioned earlier, what is actually saved also includes **the memory space used to store data**, and that can be a large memory region. The following figure illustrates this:

<figure><img src="https://pic3.zhimg.com/v2-7aa714891aa161db06800440c64d01da_b.jpg" alt=""><figcaption></figcaption></figure>

The SRQ in the figure has two RQ WQEs. Look at the content of the RQ WQEs: they consist of several SGEs (Scatter/Gather Elements). Each SGE consists of a memory address, a length, and a key. With a starting address and length, an SGE can point to a contiguous memory region. Multiple SGEs can represent multiple discontiguous contiguous memory blocks, and we call multiple SGEs an SGL (Scatter/Gather List). SGEs appear throughout the IB software stack, and in fact they are common throughout Linux. They can represent very large memory regions with very little space, and IB users use SGEs to specify send and receive regions.

We can roughly estimate how much memory each SGE can point to. `length` is a 32-bit unsigned integer, so it can represent 4 GB of space. If one RQ WQE can store up to 256 SGEs, then one RQ WQE can represent 1 TB in total. Of course, actual usage is not this large. This example is only meant to show intuitively how much memory may stand behind an RQ WQE.

#### SRQC <a href="#h_279904125_3" id="h_279904125_3"></a>

SRQC means SRQ Context. Like QPC, SRQC tells hardware about SRQ-related attributes, including depth, WQE size, and other information. This note does not repeat those details.

#### SRQN <a href="#h_279904125_4" id="h_279904125_4"></a>

SRQN means SRQ Number. As with QPs, one node may contain multiple SRQs. To identify and distinguish these SRQs, each SRQ has a sequence number called the SRQN.

#### SRQ PD <a href="#h_279904125_5" id="h_279904125_5"></a>

In [7. Protection Domain](https://zhuanlan.zhihu.com/p/159493100), we introduced the concept of Protection Domain, which isolates different RDMA resources. Each SRQ must specify its own PD. It can be the same as the PD of the associated QPs, or it can be different. Multiple SRQs can also use the same PD.

When using SRQ, if a packet is received, the packet can be received normally only if the MR to be accessed and the SRQ are in the same PD. Otherwise, an immediate error is generated.

### Asynchronous Events <a href="#h_279904125_6" id="h_279904125_6"></a>

In [10. Completion Queue](https://zhuanlan.zhihu.com/p/259650980), we introduced that the IB protocol divides errors into immediate errors, completion errors, and asynchronous errors according to how they are reported. Asynchronous errors are similar to interrupts or events, so they are sometimes also called asynchronous events. Each HCA registers an event handler specifically for asynchronous events. After receiving an asynchronous event, the driver performs necessary handling and reports it further to the user.

SRQ has one special asynchronous event used to notify upper-layer users of SRQ status in time: the SRQ Limit Reached event.

#### SRQ Limit <a href="#h_279904125_7" id="h_279904125_7"></a>

An SRQ can set a waterline or threshold. When the number of remaining WQEs in the queue is lower than this waterline, the SRQ reports an asynchronous event. This reminds the user: "The WQEs in the queue are almost used up; please post more WQEs to avoid having nowhere to receive new data." This waterline or threshold is called the SRQ Limit, and the reported event is called SRQ Limit Reached.

<figure><img src="https://pic1.zhimg.com/v2-586d86eaf0d140a19a9b487df6edbb28_b.jpg" alt=""><figcaption></figcaption></figure>

Because SRQ is shared by multiple QPs, if its depth is small, its WQEs may suddenly be consumed. Therefore, the protocol designs this mechanism to ensure that users can intervene in time when WQEs are insufficient.

After the asynchronous event is reported, the SRQ Limit value is reset by hardware to 0, probably to prevent repeated asynchronous events from being reported continuously to upper layers. Of course, users can choose not to use this mechanism by setting the SRQ Limit value to 0.

### User Interfaces <a href="#h_279904125_8" id="h_279904125_8"></a>

#### Control Plane <a href="#h_279904125_9" id="h_279904125_9"></a>

The same four operations appear again: create, destroy, modify, and query.

* Create: Create SRQ

When creating an SRQ, like creating a QP, the driver allocates all related software and hardware resources. For example, it allocates an SRQN, allocates SRQC space, and fills configuration into the SRQC. When creating an SRQ, the user must also specify the depth of the SRQ, meaning how many WQEs it can store, and the maximum number of SGEs per WQE.

* Destroy: Destroy SRQ

Destroy all related software and hardware resources of the SRQ.

* Modify: Modify SRQ

In addition to attributes such as SRQ depth, the SRQ Limit value is also set through this interface. Because the waterline value is cleared every time an SRQ Limit Reached event is generated, users need to call Modify SRQ each time to set the waterline again.

* Query: Query SRQ

This is usually used to query the waterline configuration.

#### Data Plane <a href="#h_279904125_10" id="h_279904125_10"></a>

#### Post SRQ Receive <a href="#h_279904125_11" id="h_279904125_11"></a>

This is similar to Post Receive. It posts receive WQEs to the SRQ, and each WQE contains information about memory blocks used as receive buffers. Note that **the subject is the SRQ, and it has nothing to do with a particular QP**. At this point, the user does not care which QPs are associated with this SRQ.

### Differences Between SRQ and RQ <a href="#h_279904125_12" id="h_279904125_12"></a>

Functionally, SRQ and RQ are both used to store receive task descriptors. However, because SRQ is shared, it differs from RQ in several ways.

#### State Machine <a href="#h_279904125_13" id="h_279904125_13"></a>

In [9. Queue Pair](https://zhuanlan.zhihu.com/p/195757767), we introduced that QP has a complex state machine, and QP send/receive capability differs by state. SRQ only has two states: non-error and error.

In either state, the user can post WQEs to the SRQ. However, in the error state, associated QPs cannot obtain received data from this SRQ. In addition, in the error state, users cannot query or modify SRQ attributes.

When a QP enters the error state, it can return to the RESET state through Modify QP. For SRQ, however, the only way to leave the error state is to destroy it.

#### Receive Flow <a href="#h_279904125_14" id="h_279904125_14"></a>

For one QP, RQ and SRQ cannot be used at the same time. One of them must be chosen. If a WQE is posted to the RQ of a QP that is already associated with an SRQ, an immediate error is returned.

Now compare the receive flows of SRQ and RQ. This subsection is the focus of the note. After reading it, readers should have a more complete understanding of the SRQ mechanism.

#### RQ Receive Flow <a href="#h_279904125_15" id="h_279904125_15"></a>

First, review the receive flow of a normal RQ. For the complete flow including the sender side, see [4. Operation Types](https://zhuanlan.zhihu.com/p/142175657):

0. Create QPs.

1. Through the Post Recv interface, the user posts receive WQEs to the RQs of QP2 and QP3 respectively. Each WQE contains information about which memory region should store received data.

2. Hardware receives data.

3. Hardware finds that the data is sent to QP3, so it takes WQE1 from QP3's RQ and places the received data into the memory region specified by WQE1.

4. After hardware completes data placement, it generates a CQE in CQ3, which is associated with QP3's RQ, to report task completion information.

5. The user takes the WC, namely the CQE, from CQ3, and then takes the data from the specified memory region.

6. Hardware receives data.

7. Hardware finds that the data is sent to QP2, so it takes WQE1 from QP2's RQ and places the received data into the memory region specified by WQE1.

8. After hardware completes data placement, it generates a CQE in CQ2, which is associated with QP2's RQ, to report task completion information.

9. The user takes the WC, namely the CQE, from CQ2, and then takes the data from the specified memory region.

<figure><img src="https://pic4.zhimg.com/v2-c755d526e37757d5a70c4cc5262ca68f_b.jpg" alt=""><figcaption></figcaption></figure>

#### SRQ Receive Flow <a href="#h_279904125_16" id="h_279904125_16"></a>

The SRQ receive flow is different:

0. Create SRQ1, and create QP2 and QP3, both associated with SRQ1.

1. Through the Post SRQ Recv interface, the user posts two receive WQEs to SRQ1. Each WQE contains information about which memory region should store received data.

2. Hardware receives data.

3. Hardware finds that the data is sent to QP3. It takes the first WQE from SRQ1, now WQE1, and places the received data according to the WQE content.

> Each WQE in the SRQ is "ownerless" and is not associated with any specific QP. Hardware takes WQEs from the queue in order and places data into them.\
>

4. Hardware finds that **the CQ associated with QP3's RQ** is CQ3, so it generates a CQE there.

5. The user takes the CQE from CQ3 and takes the data from the specified memory region.

> Careful readers may ask: when users post WRs, each WR specifies memory regions that will later be used to store data. But SRQ is a pool, and each WQE in it points to different memory regions. After the user receives a WC from the CQ corresponding to some QP, how does the user know where the received data was stored?\
> The WC contains `wr_id`, which tells the user which WR, or WQE, specified the memory region that received the data. Since the WR was posted by the user, the user naturally knows the exact location it points to.\
>

6. Hardware receives data.

7. Hardware finds that the data is sent to QP2. It takes the first WQE from SRQ1, now WQE2, and places the received data according to the WQE content.

8. Hardware finds that the CQ associated with QP2's RQ is CQ2, so it generates a CQE there.

9. The user takes the CQE from CQ2 and takes the data from the specified memory region.

<figure><img src="https://pic3.zhimg.com/v2-6219e911fbeb4242a569524438eee816_b.jpg" alt=""><figcaption></figcaption></figure>

### Summary <a href="#h_279904125_17" id="h_279904125_17"></a>

This note first introduced the basic concept of SRQ, then explained its design motivation, related mechanisms, and user interfaces, and finally compared SRQ and RQ receive flows. In real workloads, SRQ is used quite often, so it is worth understanding in depth.

That is all for this note. Thanks for reading. The next note introduces Memory Window.

### Protocol Sections <a href="#h_279904125_18" id="h_279904125_18"></a>

10.2.9 SRQ design motivation and related operations

10.2.3 PDs of SRQ and QP

10.8.2 Relationship between QPs associated with SRQ and QPs not using SRQ

10.8.5 WCs returned for SRQ-related operations

11.5.2.4 Asynchronous events

### Other References <a href="#h_279904125_19" id="h_279904125_19"></a>

\[1] Linux Kernel Networking - Implement and Theory. Chapter 13. Shared Receive Queue
