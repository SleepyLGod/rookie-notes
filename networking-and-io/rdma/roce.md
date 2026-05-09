---
description: RDMA RoCE and Soft-RoCE
---

# 😅 RoCE

RDMA-capable NICs are expensive. Taking Mellanox, now NVIDIA, as an example, the cheapest single-port model of its latest-generation InfiniBand-capable NIC on the official site, [ConnectX-6](https://link.zhihu.com/?target=https%3A//store.mellanox.com/categories/infiniband/infiniband-vpi-adapters/connectx-6-vpi.html%23), costs 795 USD. For students, this is not a small expense.

In real applications, RDMA technology depends on NICs to complete most of the work. Fortunately, we have Soft-RoCE. It uses software instead of hardware to wrap IB transport-layer packets inside ordinary UDP packets, allowing ordinary NICs to send RoCE packets. This provides a very low-cost way to study the IB transport protocol and to write and debug Verbs-based RDMA programs.

This note introduces what RoCE is, where it comes from, and how Soft-RoCE works.

RoCE stands for `RDMA over Converged Ethernet`, meaning RDMA over converged Ethernet.

In plain terms, it implements part of InfiniBand's upper-layer protocols on top of some lower-layer protocols from traditional Ethernet.

### RoCE Protocol Layers <a href="#h_361740115_1" id="h_361740115_1"></a>

The following figure appeared earlier. It clearly shows the relationship between these protocols:

<figure><img src="https://pic2.zhimg.com/v2-106078a152d4926ac8234022bd629c79_b.jpg" alt=""><figcaption></figcaption></figure>

This may still not be intuitive enough, so expand one `RoCE v2` packet, without drawing the physical-layer protocol:\


<figure><img src="https://pic1.zhimg.com/v2-17e04efb14c550ad0be456b7b71209b4_b.jpg" alt=""><figcaption></figcaption></figure>

First comes the Layer 2 Ethernet frame, then the IP header and UDP header, and finally checksums for the protocol layers. The InfiniBand transport-layer packet is actually the UDP payload, shown as the dark-blue part. The UDP header has a Destination Port Number field. For RoCE v2, this value is fixed at 4791. After the peer NIC receives the packet, it uses this field to identify whether the packet is an ordinary Ethernet packet, a RoCE packet, or another protocol packet, and then parses it accordingly. The dark-blue IB transport-layer part is further divided into the IB header, actual user data or Payload, and the checksum portion. The IB transport layer has many header types and corresponding formats, which can be introduced later.

**Why** design RoCE after we already have the InfiniBand protocol? The main reason is cost. InfiniBand defines a completely new layered architecture. From link layer to transport layer, it is incompatible with existing Ethernet devices. In other words, if a data center hits a performance bottleneck and wants to switch its data-exchange method from Ethernet to InfiniBand, it needs to buy a full set of InfiniBand devices, including NICs, cables, switches, routers, and so on. Commercial devices have high reliability requirements, so the full set is very expensive.

The RoCE protocol reduces this cost by reusing Ethernet infrastructure and carrying RDMA semantics over Ethernet. This must not be misunderstood as "only buying RoCE-capable NICs is enough for production." RoCE is sensitive to packet loss and congestion. Production deployments usually also require QoS, PFC/ECN, buffer, MTU, and related configuration on endpoints and switches. RoCE v1 cannot cross Layer 3 routing, while RoCE v2 supports Layer 3 networks through UDP/IP encapsulation.

Therefore, compared with InfiniBand, RoCE mainly saves cost. Of course, it still has some performance loss compared with InfiniBand, because InfiniBand is a full-stack redesign.

As for iWARP, its protocol stack is more complex than RoCE, and because of TCP constraints it only supports reliable transport and cannot support transport types such as UD. Therefore, iWARP has not developed as well as RoCE and InfiniBand.

### Soft-RoCE <a href="#h_361740115_3" id="h_361740115_3"></a>

Although RoCE has compatibility advantages over InfiniBand and is cheaper, **real applications** still **require dedicated NIC support**. Some readers may ask: if the ordinary UDP/IP protocol stack is implemented in software, and RDMA transport is only carried in the UDP payload, why does it depend on hardware?

RoCE itself can indeed be implemented in software, namely Soft-RoCE, which this section introduces. But in commercial deployments, almost nobody uses software-implemented RoCE. One major feature of RDMA is "hardware offload": work that would otherwise be done by software, or CPU, is moved into hardware for acceleration. CPUs are mainly used for computation; asking them to process protocol encapsulation, parsing, and data movement wastes compute resources. Therefore, RoCE NICs mainly offload RDMA transport processing, memory-address translation, data placement, and related paths. RoCEv2 uses UDP/IP encapsulation, but it should not be simplified as putting the TCP/IP protocol stack into hardware.

Returning to Soft-RoCE, it was implemented by the IBTA RoCE working group led by IBM and Mellanox. Its original design goals include:

* Lowering RoCE deployment cost

Soft-RoCE allows hardware without RoCE capability to communicate with RoCE-capable hardware using IB semantics. This avoids replacing older NICs on some non-critical nodes in the network.

* Improving performance compared with TCP?

Although implementing the IB transport layer in software introduces overhead, compared with traditional Socket-TCP/IP communication, Soft-RoCE _may still provide a performance improvement?_ because it reduces system calls, only using a system call when software notifies hardware that a new SQ WQE has been posted, provides zero copy on the sender side, and requires only a single copy on the receiver side.

> Note: According to feedback from others and my own tests, Soft-RoCE performs worse than TCP. The main reasons are:\
> 1. The maximum IB transport-layer MTU is 4096. With a 256-byte header plus a 4096-byte payload, the header ratio is high. TCP's MTU can be much larger, effectively increasing payload efficiency.\
> 2. NICs often provide hardware acceleration for TCP.\
> 3. Soft-RoCE uses the CPU to compute CRC, which is slow.\
> \
> Here are possible performance-improvement ideas I can think of:\
> 1. **Use multiple threads when programming, bind each thread to one core**, and avoid sharing QPs between threads because lock contention can occur.\
> 2. Use a WR List instead of a single WR. That is, post a linked list of multiple WRs each time Post Send is called, reducing the system-call overhead of ringing the Doorbell.\
> 3. Set the NIC MTU to be larger than 4096 + 256, which avoids link-layer packet splitting overhead.\
> 4. If this is RXE-to-RXE testing, one can modify the RXE driver to disable CRC checking and increase the RoCE MTU, but this violates the protocol and seems not very meaningful.\
> \
> Community discussion link: [Soft-RoCE performance - Christian Blume (kernel.org)](https://link.zhihu.com/?target=https%3A//lore.kernel.org/linux-rdma/CAGP7Hd6PAYcX\_gMMh8jbpezeSSWQxqDrYwxEq1N-zjgT7563%2Bg%40mail.gmail.com/)

* Making RDMA program development and testing easier

With Soft-RoCE, programs written with the Verbs API can run without depending on RDMA hardware, and they can also run conveniently in virtual machines.

#### Implementation Principle <a href="#h_361740115_4" id="h_361740115_4"></a>

Soft-RoCE moves packet encapsulation and parsing work that would normally be offloaded to hardware back into software. It is implemented on top of the Linux kernel TCP/IP protocol stack. The NIC itself is not aware that the packets being sent and received are RoCE packets. The driver encapsulates user data into IB transport-layer packets according to the IB specification, then treats the entire packet as data and fills it into the Socket Buffer. The NIC then performs the next send/receive packet-processing step.

The following figure is from the [**IBTA introduction to Soft-RoCE**](https://www.roceinitiative.org/wp-content/uploads/2016/11/SoftRoCE\_Paper\_FINAL.pdf). The left side is ordinary hardware RoCE, and the right side is Soft-RoCE. Ordinary RoCE offloads the protocol stack to the RoCE NIC, while Soft-RoCE implements it in the software protocol stack.

<figure><img src="../../.gitbook/assets/image (24).png" alt=""><figcaption><p>Soft-RoCE implements the packet processing otherwise managed by the RoCE NIC.</p></figcaption></figure>
