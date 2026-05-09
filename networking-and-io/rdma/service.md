# 🥳 Service

In "[**Elements**](elements.md)", we mentioned that **the basic RDMA communication unit is the** [**QP**](elements.md#h\_141267386\_2). There are many QP-based communication models, and in RDMA terminology these are called "service types". The IB protocol describes service types along two dimensions: reliability and connection.

### Reliability

Reliability in communication means using certain mechanisms to ensure that transmitted packets can be received correctly. The IB protocol describes reliable service as follows:

> **Reliable Service** provides a guarantee that messages are delivered from a requester to a responder at most once, in order and without corruption.

In other words, a reliable service guarantees that a message is delivered from the requester to the responder at most once, and that it is received completely and in send order.

IB uses the following three mechanisms to provide reliability.

#### Acknowledgment Mechanism

Suppose A sends a packet to B. How can A know that B received it? Naturally, B replies to A with an "I received it" message. In communication systems, this reply is usually called an acknowledgment packet, or ACK. In reliable IB service types, acknowledgments are used to ensure that packets are received by the peer. In IB reliable service types, the receiver does not have to reply to every packet individually; it may acknowledge multiple packets at once. This can be discussed in more detail later.

<figure><img src="https://pic4.zhimg.com/v2-e3d2e6b28c2bb6c445e55977d8f1603b_b.jpg" alt=""><figcaption></figcaption></figure>

#### Data-Checking Mechanism

This mechanism is easy to understand. The sender computes a check value over the header and payload, meaning the actual data to be transmitted, using a certain algorithm, and appends that value to the end of the packet. After the peer receives the packet, it computes the check value using the same algorithm and compares it with the value in the packet. If they differ, the data contains an error, usually caused by a link problem, so the receiver discards the packet. The IB protocol uses CRC checking. This note does not expand on CRC.

<figure><img src="https://pic1.zhimg.com/v2-ce187eb2837b4b01a078eea51f4432a0_b.jpg" alt=""><figcaption></figcaption></figure>

#### Ordering Mechanism

Ordering means ensuring that packets sent earlier onto the physical link are received by the receiver before packets sent later. Some workloads have strict ordering requirements, such as voice or video. The IB protocol has the concept of PSN (Packet Sequence Number), meaning each packet has an increasing sequence number. PSN can be used to detect packet loss. For example, if the receiver receives packet 1 and then receives packet 3 without receiving packet 2, it considers an error to have occurred during transmission and replies with a NAK to the sender, asking it to retransmit the missing packet.

<figure><img src="https://pic4.zhimg.com/v2-ebb4d1bce1f692e4bd9621886e13be87_b.jpg" alt=""><figcaption></figcaption></figure>

Unreliable service does not have the mechanisms above to ensure that packets are received correctly. It is a "send it out; I do not care whether it is received" service type.

### Connection and Datagram

**Connection** here is an abstract logical concept and should be distinguished from a physical connection. Readers familiar with sockets should already know this concept. A connection is a communication "pipe". Once the pipe is established, data sent from one end of the pipe will travel along that pipe to the other end.

There are many definitions of "connection" or "connection-oriented". Some emphasize message ordering, some emphasize a unique delivery path, some emphasize the software and hardware overhead required to maintain a connection, and some overlap with the concept of reliability. Since this series introduces RDMA, look at the description in Section 3.2.2 of the IB protocol:

> IBA supports both connection oriented and datagram service. For connected service, each QP is associated with exactly one remote consumer. In this case the QP context is configured with the identity of the remote consumer’s queue pair. ... During the communication establishment process, this and other information is exchanged between the two nodes.

That is, IBA supports both connection-oriented and datagram services. For connected service, each QP is associated with exactly one remote node. In this case, the QP Context contains information about the remote node's QP. During communication establishment, the two nodes exchange peer information, including the QP that will later be used for communication.

The word Context in the description above is usually translated as context. QP Context, or QPC, can be simply understood as a table that records information related to a QP. We know that a QP consists of two queues, but in addition to those two queues, information about the QP must also be recorded in a table. This information may include queue depth, queue number, and so on. Later notes will expand on this.

This may still feel abstract, so use a figure:

<figure><img src="https://pic3.zhimg.com/v2-320b1db2b90c5334cb4200a0784a12ce_b.jpg" alt=""><figcaption></figcaption></figure>

The NICs of nodes A, B, and C are physically connected. QP2 on A establishes a logical connection with QP7 on B, and QP4 on A establishes a logical connection with QP2 on C. In other words, they are "bound together". **In a connected service type, each QP establishes a connection with exactly one other QP, which means every WQE posted to that QP has a unique destination**. In the figure above, for every WQE posted to QP2 on A, hardware can use the QPC to know that the destination is QP7 on B. It sends the assembled packet to B, and B places the data according to the RQ WQE posted to QP7. Similarly, for every WQE posted to QP4 on A, A's hardware knows that the data should be sent to QP2 on node C.

How is a "connection" maintained? It is essentially just a record inside the QPC. If QP2 on A wants to disconnect from QP7 on B and connect to another QP, it only needs to modify the QPC. During connection setup between two nodes, they exchange the QP Numbers that will later be used for data exchange and record them in their respective QPCs.

\


**Datagram** is the opposite of connection. There is no need to "build a pipe" between sender and receiver. As long as the sender can physically reach the receiver, it may send to any receiving node through any path. The IB protocol defines it as follows:

> For datagram service, a QP is not tied to a single remote consumer, but rather information in the WQE identifies the destination. A communication setup process similar to the connection setup process needs to occur with each destination to exchange that information.\
> In other words, for datagram service, a QP is not bound to one unique remote node. Instead, the WQE specifies the destination node. As with connected service, the communication setup process requires both endpoints to exchange peer information, but datagram service must perform this exchange for each destination node.

Here is an example:

<figure><img src="https://pic3.zhimg.com/v2-4576be474bb2fe5ec748d4df9c2cbaa2_b.jpg" alt=""><figcaption></figcaption></figure>

In the Context of a datagram-type QP, peer information is not stored, meaning each QP is not bound to another QP. **Each WQE posted by the QP to hardware may point to a different destination**. For example, the first WQE posted to QP2 on node A instructs hardware to send data to QP3 on node C. The next WQE may instruct hardware to send data to QP7 on node B.

As with connected service types, which remote QP a local QP can send data to is communicated in advance during the preparation phase. This is the meaning of the statement above that "datagram service must perform this exchange for each destination node."

#### Service Types

The two dimensions above combine pairwise into the four basic IB service types:

<figure><img src="https://pic3.zhimg.com/v2-37597449e3e4290f3bdf40d5ec23cac6_b.jpg" alt=""><figcaption></figcaption></figure>

RC and UD are the most widely used and most fundamental service types. They can be roughly compared to TCP and UDP at the transport layer of the TCP/IP protocol stack.

RC is used in scenarios with high requirements for data integrity and reliability. Like TCP, it needs various mechanisms to guarantee reliability, so its overhead is naturally higher. In addition, RC service requires each pair of nodes to maintain its own QP. If N nodes need to communicate with each other, **N * (N - 1)** QPs are required. QPs and QPCs consume NIC resources or memory, so when the number of nodes is large, storage-resource consumption becomes very high.

<figure><img src="https://pic4.zhimg.com/v2-956d4da11726ee385308e010d82bc9bf_b.jpg" alt=""><figcaption></figcaption></figure>

UD has lower hardware overhead and saves storage resources. For example, if N nodes need to communicate with each other, only **N** QPs need to be created. However, reliability cannot be guaranteed, just like UDP. If users want to implement reliability on top of the UD service type, they need to implement application-layer reliable transmission mechanisms above the IB transport layer.

<figure><img src="https://pic2.zhimg.com/v2-c4b783dad1632469091d54594c35fd71_b.jpg" alt=""><figcaption></figcaption></figure>

Besides these, there are RD and UC service types, as well as more complex service types such as XRC (Extended Reliable Connection) and SRD (Scalable Reliable Datagram). These will be described in detail in the protocol-analysis section.

For more information about choosing QP types, see the RDMAmojo article [**Which Queue Pair type to use?**](https://www.rdmamojo.com/2013/06/01/which-queue-pair-type-to-use/)  ****.
