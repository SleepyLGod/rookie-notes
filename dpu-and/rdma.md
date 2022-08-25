# 😍 RDMA

RDMA(RemoteDirect Memory Access)技术全称**远程直接内存访问**，就是为了解决网络传输中服务器端数据处理的延迟而产生的。它将数据直接从一台计算机的内存传输到另一台计算机，无需双方操作系统的介入。这允许高吞吐、低延迟的网络通信，尤其适合在大规模并行计算机集群中使用。

RDMA通过网络把资料直接传入计算机的存储区，将数据从一个系统快速移动到远程系统存储器中，而不对操作系统造成任何影响，这样就不需要用到多少计算机的处理能力。它消除了外部存储器复制和上下文切换的开销，因而能解放内存带宽和CPU周期用于改进应用系统性能。

本次详解我们从三个方面详细介绍RDMA：RDMA背景、RDMA相关工作、RDMA技术详解。

### 一、背景介绍

![](https://tjcug.github.io/blog/images/pasted-59.png)

#### 1.1 传统TCP/IP通信模式

传统的TCP/IP网络通信，数据需要通过用户空间发送到远程机器的用户空间。

**数据发送方：**

将数据从用户应用空间Buffer复制到内核空间的Socket Buffer中。

然后Kernel空间中添加数据包头，进行数据封装。

通过一系列多层网络协议（包括传输控制协议（TCP）、用户数据报协议（UDP）、互联网协议（IP）以及互联网控制消息协议（ICMP）等）的数据包处理工作，数据才被push到NIC网卡中的Buffer进行网络传输。

**消息接受方：**

接受从远程机器发送的数据包后，要将数据包从NIC buffer中复制数据到Socket Buffer。

然后经过一些列的多层网络协议进行数据包的解析工作。

解析后的数据被复制到相应位置的用户应用空间Buffer。

这个时候再进行系统上下文切换，用户应用程序才被调用。

以上就是传统的TCP/IP协议层的工作。

![](https://tjcug.github.io/blog/images/pasted-60.png)

如今随着社会的发展，我们希望更快和更轻量级的网络通信。

#### 1.2 通信网络定义

计算机网络通信中最重要两个衡量指标主要是指**高带宽和低延迟**。

通信延迟主要是指：处理延迟和网络传输延迟。

处理延迟开销指的就是消息在发送和接收阶段的处理时间。

网络传输延迟指的就是消息在发送和接收方的网络传输时延。如果网络通信状况很好的情况下，网络基本上可以达到高带宽和低延迟。

#### 1.3 当今网络现状

消息通信主要分为两类消息：

一类是`Large messages`，在这类消息通信中，**网络传输延迟**占整个通信中的主导位置。

一类消息是`Small messages`，在这类消息通信中，消息发送端和接受端的处理开销占整个通信的主导地位。

然而在现实计算机网络中的通信场景中，主要是以发送小消息为主。所有说发送消息和接受消息的处理开销占整个通信的主导的地位。

具体来说，**处理开销**指的是buffer管理、在不同内存空间中消息复制、以及消息发送完成后的系统中断。

#### 1.4 传统TCP/IP存在的问题

传统的TPC/IP存在的问题主要是指`I/O bottleneck`瓶颈问题。在高速网络条件下与网络I/O相关的主机处理的高开销限制了可以在机器之间发送的带宽。

这里的高额开销是数据移动操作和复制操作。

具体来讲，主要是传统的TCP/IP网络通信是通过**内核**发送消息。

`Messaging passing through kernel` 这种方式会导致很低的性能和很低的灵活性：

性能低下的原因主要是由于网络通信通过内核传递，这种通信方式存在的很高的数据移动和数据复制的开销。并且现如今内存带宽性相较如CPU带宽和网络带宽有着很大的差异。

很低的灵活性的原因主要是所有网络通信协议通过内核传递，这种方式很难去支持新的网络协议和新的消息通信协议以及发送和接收接口。

### 二、相关工作

高性能网络通信历史发展主要有以下四个方面：`TCP Offloading Engine（TOE）`、`User-Net Networking(U-Net)`、`Virtual interface Architecture（VIA）`、`Remote Direct Memroy Access(RDMA)`。U-Net是第一个跨过内核网络通信的模式之一。VIA首次提出了标准化user-level的网络通信模式，其次它组合了U-Net接口和远程DMA设备。RDMA就是现代化高性能网络通信技术。

#### 2.1 TCP Offloading Engine

在主机通过网络进行通信的过程中，主机处理器需要耗费大量资源进行多层网络协议的数据包处理工作，这些协议包括传输控制协议（TCP）、用户数据报协议（UDP）、互联网协议（IP）以及互联网控制消息协议（ICMP）等。由于CPU需要进行繁重的封装网络数据包协议，为了将占用的这部分主机处理器资源解放出来专注于其他应用，人们发明了TOE（TCP/IP Offloading Engine）技术，将上述主机处理器的工作转移到网卡上。

这种技术需要特定网络接口-网卡支持这种Offloading操作。**这种特定网卡能够支持封装多层网络协议的数据包**，这个功能常见于高速以太网接口上，如吉比特以太网（GbE）或10吉比特以太网（10GbE）。

#### 2.2 User-Net Networking(U-Net)

U-Net的设计目标是将**协议处理部分移动到用户空间去处理**。这种方式避免了用户空间将数据移动和复制到内核空间的开销。它的设计宗旨就是移动整个协议栈到用户空间中去，并且从数据通信路径中彻底删除内核。这种设计带来了高性能的提升和高 灵活性的提升。

![](https://tjcug.github.io/blog/images/pasted-61.png)

U-Net的virtual NI 为每个进程提供了一种拥有网络接口的错觉，内核接口只涉及到连接步骤。传统上的网络，内核控制整个网络通信，所有的通信都需要通过内核来传递。U-Net应用程序可以通过MUX直接访问网络，应用程序通过MUX直接访问内核，而不需要将数据移动和复制到内核空间中去。

### 三、RDMA详解

RDMA(Remote Direct Memory Access)技术全称远程直接内存访问，就是为了解决网络传输中服务器端数据处理的延迟而产生的。

RDMA通过网络把资料直接传入计算机的**存储区**，将数据从一个系统快速移动到**远程系统存储器**中，而不对操作系统造成任何影响，这样就不需要用到多少计算机的处理功能。它消除了外部存储器复制和上下文切换的开销，因而能解放内存带宽和CPU周期用于改进应用系统性能。

![](https://tjcug.github.io/blog/images/pasted-62.png)

RDMA主要有以下三个特性：

1. Low-Latency&#x20;
2. Low CPU overhead&#x20;
3. high bandwidth

#### 3.1 RDMA 简介

Remote：数据通过网络与远程机器间进行数据传输

Direct：没有内核的参与，**有关发送传输的所有内容都卸载到网卡上**

Memory：在用户空间虚拟内存与RNIC网卡**直接**进行数据传输不涉及到系统内核，没有额外的数据移动和复制

Access:  send、receive、read、write、atomic操作

#### 3.2 RDMA基本概念与术语

#### **3.2.1 基本术语**

**Fabric：**所谓Fabric，就是支持RDMA的局域网(LAN)。

```
A local-area RDMA network is usually referred to as a fabric.
```

**CA(Channel Adapter)：**CA是Channel Adapter(通道适配器)的缩写。那么，CA就是将系统连接到Fabric的硬件组件。&#x20;

在IBTA中，一个CA就是IB子网中的一个终端结点(End Node)。

分为两种类型，一种是HCA, 另一种叫做TCA, 它们合称为xCA。

其中， `HCA(Host Channel Adapter)`是支持"verbs"接口的CA, T`CA(Target Channel Adapter)`可以理解为"weak CA", 不需要像HCA一样支持很多功能。

而在IEEE/IETF中，CA的概念被实体化为`RNIC（RDMA Network Interface Card）`, iWARP就把一个CA称之为一个RNIC。

**简言之，在IBTA阵营中，CA即HCA或TCA； 而在iWARP阵营中，CA就是RNIC。 总之，无论是HCA、 TCA还是RNIC，它们都是CA, 它们的基本功能本质上都是生产或消费数据包(packet)。**

```
A channel adapter is the hardware component that connects a system to the fabric.
```

**Verbs**：在RDMA的持续演进中，有一个组织叫做OpenFabric Alliance所做的贡献可谓功不可没。 Verbs这个词不好翻译，大致可以理解为访问RDMA硬件的“一组标准动作”。 每一个Verb可以理解为一个Function。

#### **3.2.2 核心概念**

****

#### **3.2.3 其他**

RDMA有两种基本操作：

* Memory verbs:  包括RDMA read、write和atomic操作。这些操作**指定远程地址进行操作并且绕过接收者的CPU**。
* Messaging verbs:  包括RDMA send、receive操作。这些动作**涉及响应者的CPU，发送的数据被写入由响应者的CPU先前发布的接受所指定的地址**。

RDMA传输分为可靠和不可靠的，并且可以连接和不连接的（数据报）。

凭借可靠的传输，NIC使用确认来保证消息的按序传送。不可靠的传输不提供这样的保证。

然而，像InfiniBand这样的现代RDMA实现使用了一个无损链路层，它可以防止使用链路层流量控制的基于拥塞的损失\[1]，以及使用链路层重传的基于位错误的损失\[8]。因此，不可靠的传输很少会丢弃数据包。

**目前的RDMA硬件提供一种数据报传输：不可靠的数据报（UD），并且不支持memory verbs。**

![](https://tjcug.github.io/blog/images/pasted-63.png)

#### 3.3 RDMA三种不同的硬件实现

目前RDMA有三种不同的**硬件实现**。分别是 `InfiniBand`、`iWarp(internet Wide Area RDMA Protocol)`、`RoCE(RDMA over Converged Ethernet)`。

![](https://tjcug.github.io/blog/images/pasted-64.png)

目前，大致有三类RDMA**网络**，分别是 `Infiniband`、`RoCE`、`iWARP`。

其中，`Infiniband` 是一种专为RDMA设计的网络，从**硬件级别**保证可靠传输 ；`Infiniband` 支持RDMA的新一代网络协议。 由于这是一种新的网络技术，因此需要支持该技术的NIC和交换机。

而`RoCE`和 `iWARP`都是基于以太网的RDMA技术，支持相应的verbs接口，如上图所示。

<mark style="color:blue;background-color:blue;">**`RoCE`**</mark>，一个允许在以太网上执行RDMA的网络协议。 其较低的网络标头是以太网标头，其较高的网络标头（包括数据）是InfiniBand标头。 这支持在标准以太网基础设施（交换机）上使用RDMA。 只有网卡应该是特殊的，支持RoCE。从图中不难发现，`RoCE`协议存在 `RoCEv1` 和 `RoCEv2` **两个版本**，主要区别`RoCEv1`是基于以太网**链路层**实现的RDMA协议(交换机需要支持PFC等流控技术，在物理层保证可靠传输)，而`RoCEv2`是以太网TCP/IP协议中UDP层实现。

`iWARP`，一个允许在TCP上执行RDMA的网络协议。 IB和RoCE中存在的功能在iWARP中不受支持。 这支持在标准以太网基础设施（交换机）上使用RDMA。 只有网卡应该是特殊的，并且支持iWARP（如果使用CPU卸载），否则所有iWARP堆栈都可以在SW中实现，并且丧失了大部分RDMA性能优势。

从性能上，很明显`Infiniband`网络最好，但网卡和交换机是价格也很高，然而`RoCEv2`和`iWARP`仅需使用特殊的网卡就可以了，价格也相对便宜很多。

![](https://tjcug.github.io/blog/images/pasted-65.png)

![](https://tjcug.github.io/blog/images/pasted-66.png)

#### 3.4 RDMA技术

![](https://tjcug.github.io/blog/images/pasted-67.png)

传统上的RDMA技术设计内核封装多层网络协议并且涉及内核数据传输。RDMA通过专有的RDMA网卡**RNIC**，绕过内核直接从用户空间访问RDMA enabled NIC网卡。RDMA提供一个专有的verbs interface而不是传统的TCP/IP Socket interface。

要使用RDMA**首先要建立从RDMA到应用程序内存的数据路径** ，可以通过RDMA专有的verbs interface接口来建立这些数据路径，一旦数据路径建立后，就可以直接访问用户空间buffer。

#### 3.5 RDMA整体系统架构图

![RDMA整体框架架构图](https://tjcug.github.io/blog/images/pasted-68.png)

从图中可以看出，RDMA在应用程序用户空间，提供了一系列verbs interface接口操作RDMA硬件。

RDMA绕过内核直接从用户空间访问RDMA 网卡(RNIC)。

RNIC网卡中包括Cached Page Table Entry，页表就是用来将虚拟页面映射到相应的物理页面。

#### 3.6 RDMA技术详解

RDMA 的工作过程如下:

1.  当一个应用执行RDMA 读或写请求时，不执行任何数据复制。

    在不需要任何内核内存参与的条件下，RDMA 请求从运行在用户空间中的应用中发送到**本地NIC**( 网卡)。
2. NIC 读取缓冲的内容，并通过网络传送到**远程NIC**。
3.  在网络上传输的RDMA 信息**包含**目标虚拟地址、内存钥匙和数据本身。

    请求既可以完全在用户空间中处理(通过轮询用户级完成排列) ，又或者在应用一直睡眠到请求完成时的情况下通过系统中断处理。

    RDMA 操作使应用可以从一个远程应用的内存中读数据或向这个内存写数据。
4.  目标NIC 确认内存钥匙，直接将数据写人应用缓存中。

    用于操作的远程虚拟内存地址包含在RDMA 信息中。

#### 3.7 RDMA操作细节

RDMA提供了**基于消息队列的点对点通信**，每个应用都可以直接获取自己的消息，无需操作系统和协议栈的介入。

消息服务建立在通信双方本端和远端应用之间创建的**Channel-IO**连接之上。

当应用需要通信时，就会创建一条Channel连接，每条Channel的首尾端点是两对**Queue Pairs**（**QP**）。每对QP由Send Queue（**SQ**）和Receive Queue（**RQ**）构成，这些队列中管理着各种类型的消息。

QP会被映射到应用的虚拟地址空间，使得应用直接通过它访问RNIC网卡。

除了QP描述的两种基本队列之外，RDMA还提供一种队列Complete Queue（**CQ**），CQ用来知会用户**WQ**上的消息已经被处理完。

RDMA提供了一套软件传输接口，方便用户创建传输请求Work Request(**WR**），WR中描述了应用希望传输到Channel对端的消息内容，WR通知QP中的某个队列Work Queue(**WQ)**。

在WQ中，用户的WR被转化为Work Queue Element（**WQE**）的格式，等待RNIC的异步调度解析，并从WQE指向的Buffer中拿到真正的消息发送到Channel对端。

RDMA是一种host-offload, host-bypass技术，允许应用程序(包括存储)在它们的内存空间之间直接做数据传输。具有RDMA引擎的以太网卡(RNIC)--而不是host--负责管理源和目标之间的可靠连接。使用RNIC的应用程序之间使用专注的QP和CQ进行通讯：

1. 每一个应用程序可以有很多QP和CQ
2. 每一个QP包括一个SQ和RQ
3. 每一个CQ可以跟多个SQ或者RQ相关联

![](https://tjcug.github.io/blog/images/pasted-69.png)

READ和WRITE是单边操作，只需要本端明确信息的**源和目的地址**，远端应用不必感知此次通信，数据的读或写都通过RDMA在RNIC与应用Buffer之间完成，再由远端RNIC封装成消息返回到本端。

单边操作传输方式是RDMA与传统网络传输的最大不同，只需提供直接访问远程的虚拟地址，无须远程应用的参与其中，这种方式适用于批量数据传输。

**3.7.1 RDAM单边操作 (RDMA READ)**

对于单边操作，以存储网络环境下的存储为例，数据的流程如下：

1. 首先A、B建立连接，QP已经创建并且初始化。
2. 数据被存档在B的buffer地址VB，注意VB应该提前注册到B的RNIC (并且它是一个Memory Region) ，并拿到返回的local key，相当于RDMA操作这块buffer的权限。
3. B把数据地址VB，key封装到专用的报文传送到A，这相当于B把数据buffer的操作权交给了A。同时B在它的WQ中注册进一个WR，以用于接收数据传输的A返回的状态。
4. A在收到B的送过来的数据VB和R\_key后，RNIC会把它们连同自身存储地址VA到封装RDMA READ请求，将这个消息请求发送给B，这个过程A、B两端不需要任何软件参与，就可以将B的数据存储到B的VA虚拟地址。
5. B在存储完成后，会向A返回整个数据传输的状态信息。

**3.7.2 RDMA 单边操作 (RDMA WRITE)**

对于单边操作，以存储网络环境下的存储为例，数据的流程如下：

1. 首先A、B建立连接，QP已经创建并且初始化。
2. 数据remote目标存储buffer地址VB，注意VB应该提前注册到B的RNIC(并且它是一个Memory Region)，并拿到返回的local key，相当于RDMA操作这块buffer的权限。
3. B把数据地址VB，key封装到专用的报文传送到A，这相当于B把数据buffer的操作权交给了A。同时B在它的WQ中注册进一个WR，以用于接收数据传输的A返回的状态。
4. A在收到B的送过来的数据VB和R\_key后，RNIC会把它们连同自身发送地址VA到封装RDMA WRITE请求，这个过程A、B两端不需要任何软件参与，就可以将A的数据发送到B的VB虚拟地址。
5. A在发送数据完成后，会向B返回整个数据传输的状态信息。\
   单边操作传输方式是RDMA与传统网络传输的最大不同，只需提供直接访问远程的虚拟地址，无须远程应用的参与其中，这种方式适用于批量数据传输。

**3.7.3 RDMA 双边操作 (RDMA SEND/RECEIVE)**

RDMA中SEND/RECEIVE是双边操作，即必须要远端的应用感知参与才能完成收发。在实际中，SEND/RECEIVE多用于连接控制类报文，而数据报文多是通过READ/WRITE来完成的。\
对于双边操作为例，主机A向主机B(下面简称A、B)发送数据的流程如下：

1. 首先，A和B都要创建并初始化好各自的QP，CQ
2. A和B分别向自己的WQ中注册WQE，对于A，WQ=SQ，WQE描述指向一个等到被发送的数据；对于B，WQ=RQ，WQE描述指向一块用于存储数据的Buffer。
3. A的RNIC异步调度轮到A的WQE，解析到这是一个SEND消息，从Buffer中直接向B发出数据。数据流到达B的RNIC后，B的WQE被消耗，并把数据直接存储到WQE指向的存储位置。
4. AB通信完成后，A的CQ中会产生一个完成消息CQE表示发送完成。与此同时，B的CQ中也会产生一个完成消息表示接收完成。每个WQ中WQE的处理完成都会产生一个CQE。\
   双边操作与传统网络的底层Buffer Pool类似，收发双方的参与过程并无差别，区别在零拷贝、Kernel Bypass，实际上对于RDMA，这是一种复杂的消息传输模式，多用于传输短的控制消息。

