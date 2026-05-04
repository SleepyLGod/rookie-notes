# 🥳 Service

我们在“[**Elements**](elements.md)”一文中提到过，**RDMA的基本通信单元是**[**QP**](elements.md#h\_141267386\_2)，而基于QP的通信模型有很多种，我们在RDMA领域称其为“服务类型”。IB协议中通过“可靠”和“连接”两个维度来描述一种服务类型。

### 可靠

通信中的可靠性指的是通过一些机制保证发出去的数据包都能够被正常接收。IB协议中是这样描述可靠服务的：

> **Reliable Service** provides a guarantee that messages are delivered from a requester to a responder at most once, in order and without corruption.

即“可靠服务在发送和接受者之间保证了信息最多只会传递一次，并且能够保证其按照发送顺序完整的被接收”。

IB通过以下三个机制来保证可靠性：

#### 应答机制

假设A给B发了一个数据包，A怎样才能知道B收到了呢，自然是B回复一个“我收到了”消息给A。在通信领域我们一般称这个回复为应答包或者ACK（Acknowledge）。在IB协议的可靠服务类型中，使用了应答机制来保证数据包被对方收到。IB的可靠服务类型中，接收方不是每一个包都必须回复，也可以一次回复多个包的ACK，以后我们再展开讨论。

<figure><img src="https://pic4.zhimg.com/v2-e3d2e6b28c2bb6c445e55977d8f1603b_b.jpg" alt=""><figcaption></figcaption></figure>

#### 数据校验机制

这个比较好理解，发端会对Header和Payload（有效载荷，也就是真正要收发的数据）通过一定的算法得到一个校验值放到数据包的末尾。对端收到数据包后，也会用相同的算法计算出校验值，然后与数据包中的校验值比对，如果不一致，说明数据中包含错误（一般是链路问题导致的），那么接收端就会丢弃这个数据包。IB协议使用的CRC校验，本文对CRC不做展开介绍。

<figure><img src="https://pic1.zhimg.com/v2-ce187eb2837b4b01a078eea51f4432a0_b.jpg" alt=""><figcaption></figcaption></figure>

#### 保序机制

保序指的是，保证先被发送到物理链路上的数据包一定要先于后发送的数据包被接收方收到。有一些业务对数据包的先后顺序是有严格要求的，比如语音或者视频。IB协议中有PSN（Packet Sequence Number，包序号）的概念，即每个包都有一个递增的编号。PSN可以用来检测是否丢包，比如收端收到了1，但是在没收到2的情况下就收到了3，那么其就会认为传输过程中发生了错误，之后会回复一个NAK给发端，让其重发丢失的包。

<figure><img src="https://pic4.zhimg.com/v2-ebb4d1bce1f692e4bd9621886e13be87_b.jpg" alt=""><figcaption></figcaption></figure>

不可靠服务，没有上述这些机制来保证数据包被正确的接收，属于“发出去就行，我不关心有没有被收到”的服务类型。

### 连接与数据报

**连接（Connection）**在这里指的是一个抽象的逻辑概念，需要区别于物理连接，熟悉Socket的读者一定对这个其不陌生。连接是一条通信的“管道”，一旦管道建立好了，管道这端发出的数据一定会沿着这条管道到达另一端。

对于“连接”或者说“面向连接”的定义有很多种，有的侧重于保证消息顺序，有的侧重于消息的传递路径唯一，有的强调需要软硬件开销来维护连接，有的还和可靠性的概念有交集。本专栏既然是介绍RDMA技术，那么我们就看一下IB协议3.2.2节中对其的描述：

> IBA supports both connection oriented and datagram service. For connected service, each QP is associated with exactly one remote consumer. In this case the QP context is configured with the identity of the remote consumer’s queue pair. ... During the communication establishment process, this and other information is exchanged between the two nodes.

即“IBA支持基于连接和数据报的服务。对于基于连接的服务来说，每个QP都和另一个远端节点相关联。在这种情况下，QP Context中包含有远端节点的QP信息。在建立通信的过程中，两个节点会交换包括稍后用于通信的QP在内的对端信息"。

上面这端描述中的Context一般被翻译成上下文，QP Context（简称QPC）可以简单理解为是记录一个QP相关信息的表格。我们知道QP是两个队列，除了这两个队列之外，我们还需要把关于QP的信息记录到一张表里面，这些信息可能包括队列的深度，队列的编号等等，后面我们会展开讲。

可能还是有点抽象，我们用图说话：

<figure><img src="https://pic3.zhimg.com/v2-320b1db2b90c5334cb4200a0784a12ce_b.jpg" alt=""><figcaption></figcaption></figure>

A、B和A、C节点的网卡在物理上是连接在一起的，A上面的QP2和B上面的QP7、A上面的QP4和B上面的QP2建立了逻辑上的连接，或者说“绑定到了一起”。**在连接服务类型中的每个QP，都和唯一的另一个QP建立了连接，也就是说QP下发的每个WQE的目的地都是唯一的**。拿上图来说，对于A的QP2下发的每个WQE，硬件都可以通过QPC得知其目的为B的QP7，就会把组装好的数据包发送给B，然后B会根据QP7下发的RQ WQE来存放数据；同理，对于A的QP4下发的每个WQE，A的硬件都知道应该把数据发给Node C的QP2。

“连接”是如何维护的呢？其实就是在QPC里面的一个记录而已。如果A的QP2想断开与B的QP7的“连接”然后与其他QP相“连接”，只需要修改QPC就可以了。两个节点在建立连接的过程中，会交换稍后用于数据交互的QP Number，然后分别记录在QPC中。

\


**数据报（Datagram）**与连接相反，发端和收端间不需要“建立管道”的步骤，只要发端到收端物理上是可以到达的，那么我就可能从任何路径发给任意的收端节点。IB协议对其的定义是这样的：

> For datagram service, a QP is not tied to a single remote consumer, but rather information in the WQE identifies the destination. A communication setup process similar to the connection setup process needs to occur with each destination to exchange that information.\
> 即“对于数据报服务来说，QP不会跟一个唯一的远端节点绑定，而是通过WQE来指定目的节点。和连接类型的服务一样，建立通信的过程也需要两端交换对端信息，但是数据报服务对于每个目的节点都需要执行一次这个交换过程。”

我们举个例子：

<figure><img src="https://pic3.zhimg.com/v2-4576be474bb2fe5ec748d4df9c2cbaa2_b.jpg" alt=""><figcaption></figcaption></figure>

在数据报类型的QP的Context中，不包含对端信息，即每个QP不跟另一个QP绑定。**QP下发给硬件的每个WQE都可能指向不同的目的地**。比如节点A的QP2下发的第一个WQE，指示给节点C的QP3发数据；而下一个WQE，可以指示硬件发给节点B的QP7。

与连接服务类型一样，本端QP可以和哪个对端QP发送数据，是在准备阶段提前通过某些方式相互告知的。这也是上文“数据报服务对于每个目的节点都需要执行一次这个交换过程”的含义。

#### 服务类型

上面介绍的两个维度两两组合就形成了IB的四种基本服务类型：

<figure><img src="https://pic3.zhimg.com/v2-37597449e3e4290f3bdf40d5ec23cac6_b.jpg" alt=""><figcaption></figcaption></figure>

RC和UD是应用最多也是最基础的两种服务类型，我们可以将他们分别类比成TCP/IP协议栈传输层的TCP和UDP。

RC用于对数据完整性和可靠性要求较高的场景，更TCP一样，因为需要各种机制来保证可靠，所以开销自然会大一些。另外由于RC服务类型和每个节点间需要各自维护一个QP，假设有N个几点需要相互通信，那么需要**N \* (N - 1)**个QP，而QP和QPC本身是需要占用网卡资源或者内存的，当节点数很多时，存储资源消耗将会非常大。

<figure><img src="https://pic4.zhimg.com/v2-956d4da11726ee385308e010d82bc9bf_b.jpg" alt=""><figcaption></figcaption></figure>

UD硬件开销小并且节省存储资源，比如N个节点需要相互通信，只需要创建**N**个QP就可以了，但是可靠性跟UDP一样没法保证。用户如果想基于UD服务类型实现可靠性，那么需要自己基于IB传输层实现应用层的可靠传输机制。

<figure><img src="https://pic2.zhimg.com/v2-c4b783dad1632469091d54594c35fd71_b.jpg" alt=""><figcaption></figcaption></figure>

除此之外，还有RD和UC类型，以及XRC（Extended Reliable Connection），SRD（Scalable Reliable Datagram）等更复杂的服务类型，我们将在协议解析部分对其进行详细的描述。

更多关于QP类型选择的信息可以参考RDMAmojo上的 [**Which Queue Pair type to use?**](https://www.rdmamojo.com/2013/06/01/which-queue-pair-type-to-use/)  ****  这篇文章。
