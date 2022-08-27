# 😆 Verbs

### **概述** <a href="#h_329198771_0" id="h_329198771_0"></a>

Verbs直译过来是“动词”的意思，它在RDMA领域中有两种含义：

1\) 由IB规范所描述的一组抽象定义，规定了各厂商的软硬件在各种Verbs下应该执行的动作或者表现出的行为，IB规范并未规定如何编程实现这些Verbs，在这种含义下，Verbs是与操作系统无关的。

举个例子，IB规范要求所有RDMA设备必须支持Create QP的行为（IB 规范11.2.5.1）：

> 描述：\
> ​ 为指定的设备创建一个QP。\
> ​ 用户必须指定一组用于初始化QP的属性。\
> ​ 如果创建QP所需的属性有非法值或者缺失，那么应该返回错误，该QP不会被创建；如果成功， 那么返回该QP的指针和QPN。\
> ​ ……\
> 输入：\
> ​ 设备指针；\
> ​ SQ关联到的CQ；\
> ​ RQ关联到的CQ，如果是XRC的INI QP，则可以不携带此参数；\
> ​ ……\
> 输出：\
> ​ 新创建的QP的指针；\
> ​ QP Number;\
> ​ SQ的最大WR容量。\
> ​ ……\
>

可以看出IB规范中的Verbs，就像教科书中对一个概念进行定义，讲的是“需要支持什么，但具体怎么实现我不做规定”。

2\) 由OpenFabrics推动实现的一组RDMA应用编程接口（API）。既然是API，那么必然和运行的操作系统相关。Verbs API有Linux版本以及Windows版本（Windows版很久没有更新了）。

我们还是以Create QP为例，下文引用自Linux用户态Verbs API的帮助文档（[ibv\_create\_qp(3): create/destroy queue pair](https://link.zhihu.com/?target=https%3A//linux.die.net/man/3/ibv\_create\_qp)）：

> 名称：\
> ​ ibv\_create\_qp - create a queue pair (QP)\
> 概要：

```cpp
#include <infiniband/verbs.h>
struct ibv_qp ibv_create_qp(struct ibv_pd pd, struct ibv_qp_init_attr *qp_init_attr); 
```

> 描述：\
> ​ ibv\_create\_qp()通过一个关联的PD创建一个QP，参数qp\_init\_attr是一个ibv\_qp\_init\_attr类型的结构体，其定义在\<infiniband/verbs.h>中。

```cpp
struct ibv_qp_init_attr {
struct ibv_cq       *send_cq;        /* CQ to be associated with the Send Queue (SQ) */
struct ibv_cq       *recv_cq;        /* CQ to be associated with the Receive Queue (RQ) */
struct ibv_srq      *srq;            /* SRQ handle if QP is to be associated with an SRQ, otherwise NULL */
struct ibv_qp_cap    cap;            /* QP capabilities */
enum ibv_qp_type     qp_type;        /* QP Transport Service Type: IBV_QPT_RC, IBV_QPT_UC, or IBV_QPT_UD */
...
};
```

> ​ 函数ibv\_create\_qp()会更新qp\_init\_attr->cap struct的内容，返回创建的QP所真正支持的规格……\
> 返回值：\
> ​ ibv\_create\_qp()返回被创建的QP的指针，或者在失败时返回NULL。QPN将在返回的指针所指向的结构体中。

可见Verbs API即是对IB规范中的Verbs定义的具体软件实现。

Verbs的第一种语义直接查阅IB规范的第11章即可，里面做了非常详细的描述。

本文介绍的是第二种语义，包含Verbs API是什么，如何和硬件产生交互，我们如何通过Verbs API来编写RDMA程序。如无特殊说明，下文中的Verbs均特指Verbs API。

### 相关名词解释 <a href="#h_329198771_1" id="h_329198771_1"></a>

#### rdma-core <a href="#h_329198771_2" id="h_329198771_2"></a>

指**开源RDMA用户态软件协议栈**，包含用户态框架、各厂商用户态驱动、API帮助手册以及开发自测试工具等。

rdma-core在github上维护，我们的用户态Verbs API实际上就是它实现的。

#### kernel RDMA subsystem <a href="#h_329198771_3" id="h_329198771_3"></a>

指**开源的Linux内核中的RDMA子系统**，包含RDMA内核框架及各厂商的驱动。

RDMA子系统跟随Linux维护，是内核的的一部分。一方面提供内核态的Verbs API，一方面负责对接用户态的接口。

#### OFED <a href="#h_329198771_4" id="h_329198771_4"></a>

全称为OpenFabrics Enterprise Distribution，是一个**开源软件包集合**，其中包含内核框架和驱动、用户框架和驱动、以及各种中间件、测试工具和API文档。

开源OFED由OFA组织负责开发、发布和维护，它会定期从rdma-core和内核的RDMA子系统取软件版本，并对各商用OS发行版进行适配。除了协议栈和驱动外，还包含了perftest等测试工具。

下图为OFA给出的OFED的概览：

<figure><img src="https://pic4.zhimg.com/v2-11ed2af97e47dc17339b49e890ef31b3_b.jpg" alt=""><figcaption></figcaption></figure>

除了开源OFED之外，各厂商也会提供定制版本的OFED软件包，比如华为的HW\_OFED和Mellanox的MLNX\_OFED。这些定制版本基于开源OFED开发，由厂商自己测试和维护，会在开源软件包基础上提供私有的增强特性，并附上自己的配置、测试工具等。

以上三者是包含关系。无论是用户态还是内核态，整个RDMA社区非常活跃，框架几乎每天都在变动，都是平均每两个月一个版本。而OFED会定期从两个社区中取得代码，进行功能和兼容性测试后发布版本，时间跨度较大，以年为单位计。

### Verbs API是什么 <a href="#h_329198771_5" id="h_329198771_5"></a>

Verbs API是一组用于使用RDMA服务的最基本的软件接口，也就是说业界的RDMA应用，要么直接基于这组API编写，要么基于在Verbs API上又封装了一层接口的各种中间件编写。

Verbs API向用户提供了有关RDMA的一切功能，典型的包括：注册MR、创建QP、Post Send、Poll CQ等等。

对于Linux系统来说，Verbs的功能由rdma-core和内核中的RDMA子系统提供，分为用户态Verbs接口和内核态Verbs接口，分别用于用户态和内核态的RDMA应用。

结合上一部分的内容，我们给出一个OFED的全景：

<figure><img src="https://pic1.zhimg.com/v2-75fb344eaf89d646650d7d9605d8af6c_b.jpg" alt=""><figcaption></figcaption></figure>

广义的Verbs API主要由两大部分组成：

#### IB\_VERBS <a href="#h_329198771_6" id="h_329198771_6"></a>

接口以ibv\_xx（用户态）或者ib\_xx（内核态）作为前缀，是最基础的编程接口，使用IB\_VERBS就足够编写RDMA应用了。

比如：

* ibv\_create\_qp() 用于创建QP
* ibv\_post\_send() 用于下发Send WR
* ibv\_poll\_cq() 用于从CQ中轮询CQE

#### RDMA\_CM <a href="#h_329198771_7" id="h_329198771_7"></a>

以rdma\_为前缀，主要分为两个功能：

#### CMA（Connection Management Abstraction） <a href="#h_329198771_8" id="h_329198771_8"></a>

在Socket和Verbs API基础上实现的，用于CM建链并交换信息的一组接口。CM建链是在Socket基础上封装为QP实现，从用户的角度来看，是在通过QP交换之后数据交换所需要的QPN，Key等信息。

比如：

* rdma\_listen()用于监听链路上的CM建链请求。
* rdma\_connect()用于确认CM连接。

#### CM VERBS <a href="#h_329198771_9" id="h_329198771_9"></a>

RDMA\_CM也可以用于数据交换，相当于在verbs API上又封装了一套数据交换接口。

比如：

* rdma\_post\_read()可以直接下发RDMA READ操作的WR，而不像ibv\_post\_send()，需要在参数中指定操作类型为READ。
* rdma\_post\_ud\_send()可以直接传入远端QPN，指向远端的AH，本地缓冲区指针等信息触发一次UD SEND操作。

上述接口虽然方便，但是需要配合CMA管理的链路使用，不能配合Verbs API使用。

Verbs API除了IB\_VERBS和RDMA\_CM之外，还有MAD（Management Datagram）接口等。

**需要注意的是，软件栈中的Verbs API具体实现和IB规范中的描述并不是完全一致的。IB规范迭代较慢，而软件栈几乎每天都有变化，所以编写应用或者驱动程序时，应以软件栈API文档中的描述为准。**

狭义的Verbs API专指以ibv\_/ib\_为前缀的用户态Verbs接口，因为RDMA的典型应用是在用户态，下文主要介绍用户态的Verbs API。

### 设计Verbs API的原因 <a href="#h_329198771_10" id="h_329198771_10"></a>

传统以太网的用户，基于Socket API来编写应用程序；**而RDMA的用户，基于Verbs API来编写应用程序。**

Verbs API支持IB/iWARP/RoCE三大RDMA协议，通过统一接口，让同一份RDMA程序程序可以无视底层的硬件和链路差异运行在不同的环境中。

### Verbs API所包含的内容 <a href="#h_329198771_11" id="h_329198771_11"></a>

用户态Verbs API主要包含两个层面的功能：

为方便讲解，下面对各接口的形式做了简化，格式为"_返回值1,返回值2_ **函数名**(_参数1, 参数2_)"

1）控制面：

* 设备管理：\

*
  * _device\_list_**ibv\_get\_device\_list**()

用户获取可用的RDMA设备列表，会返回一组可用设备的指针。

*
  * _device\_context_**ibv\_open\_device**(_device_)

打开一个可用的RDMA设备，返回其上下文指针（这个指针会在以后用来对这个设备进行各种操作）。

*
  * _device\_attr, errno **ibv\_query\_device**(_device\_context\*)

查询一个设备的属性/能力，比如其支持的最大QP，CQ数量等。返回设备的属性结构体指针，以及错误码。

* 资源的创建，查询，修改和销毁：\

*
  * _pd_**ibv\_alloc\_pd**(_device\_context_)

申请PD。该函数会返回一个PD的指针。

*
  * _mr_**ibv\_reg\_mr**(_pd, addr, length, access\_flag_)

注册MR。用户传入要注册的内存的起始地址和长度，以及这个MR将要从属的PD和它的访问权限（本地读/写，远端读/写等），返回一个MR的指针给用户。

*
  * _cq_ **ibv\_create\_cq**(_device\_context, cqe\_depth, ..._)

创建CQ。用户传入CQ的最小深度（驱动实际申请的可能比这个值大），然后该函数返回CQ的指针。

*
  * _qp_**ibv\_create\_qp**(_pd, qp\_init\_attr_)

创建QP。用户传入PD和一组属性（包括RQ和SQ绑定到的CQ、QP绑定的SRQ、QP的能力、QP类型等），向用户返回QP的指针。

*
  * _errno_**ibv\_modiy\_qp**(_qp, attr, attr\_mask_)

修改QP。用户传入QP的指针，以及表示要修改的属性的掩码和要修改值。修改的内容可以是QP状态、对端QPN、QP能力、端口号和重传次数等等。如果失败，该函数会返回错误码。\
Modify QP最重要的作用是让QP在不同的状态间迁移，完成RST-->INIT-->RTR-->RTS的状态机转移后才具备下发Send WR的能力。也可用来将QP切换到ERROR状态。

*
  * _errno_**ibv\_destroy\_qp**(_qp_)

销毁QP。即销毁QP相关的软件资源。其他的资源也都有类似的销毁接口。

* 中断处理：\

*
  * _event\_info, errno_**ibv\_get\_async\_event**(_device\_context_)

从事件队列中获取一个异步事件，返回异步事件的信息（事件来源，事件类型等）以及错误码。

* 连接管理\

*
  * rdma\_xxx()

用于CM建链，不在本文展开讲。

*
  * ...

2）数据面：

* 下发WR\

*
  * _bad\_wr, errno_**ibv\_post\_send**(_qp, wr_)

向一个QP下发一个Send WR，参数_wr_是一个结构体，包含了WR的所有信息。包括wr\_id、sge数量、操作码（SEND/WRITE/READ等以及更细分的类型）。WR的结构会根据服务类型和操作类型有所差异，比如RC服务的WRITE和READ操作的WR会包含远端内存地址和R\_Key，UD服务类型会包含AH，远端QPN和Q\_Key等。

WR经由驱动进一步处理后，会转化成WQE下发给硬件。

出错之后，该接口会返回出错的WR的指针以及错误码。

*
  * _bad\_wr, errno_ **ibv\_post\_recv**(_qp, wr_)

同ibv\_post\_send，只不过是专门用来下发RECV操作的WR的接口。

* 获取WC\

*
  * _num, wc_ **ibv\_poll\_cq**(_cq, max\_num_)

从完成队列CQ中轮询CQE，用户需要提前准备好内存来存放WC，并传入可以接收多少个WC。该接口会返回一组WC结构体（其内容包括wr\_id，状态，操作码，QPN等信息）以及WC的数量。

### 使用Verbs API编写RDMA应用程序 <a href="#h_329198771_12" id="h_329198771_12"></a>

#### 查看接口定义 <a href="#h_329198771_13" id="h_329198771_13"></a>

#### 内核态 <a href="#h_329198771_14" id="h_329198771_14"></a>

内核态Verbs接口没有专门的API手册，编程时需要参考头文件中的函数注释。声明这些接口的头文件位于内核源码目录中的：

.../include/rdma/ib\_verbs.h

比如ib\_post\_send()接口：

<figure><img src="https://pic2.zhimg.com/v2-e21e86b7b2193758b2f4d2f5096825a1_b.jpg" alt=""><figcaption></figcaption></figure>

函数注释中有明确介绍该函数的作用，输入、输出参数以及返回值。

#### 用户态 <a href="#h_329198771_15" id="h_329198771_15"></a>

有多种方法查阅用户态的Verbs API：

#### 在线查阅最新man page <a href="#h_329198771_16" id="h_329198771_16"></a>

用户态的Verbs API手册跟代码在一个仓库维护，手册地址：

[https://github.com/linux-rdma/rdma-core/tree/master/libibverbs/man](https://link.zhihu.com/?target=https%3A//github.com/linux-rdma/rdma-core/tree/master/libibverbs/man)

这里是按照Linux的man page格式编写的源文件，直接看源文件可能不太直观。有很多在线的man page网站可以查阅这些接口的说明，比如官方的连接：

[https://man7.org/linux/man-pages/man3/ibv\_post\_send.3.html](https://link.zhihu.com/?target=https%3A//man7.org/linux/man-pages/man3/ibv\_post\_send.3.html)

也有一些其他非官方网页，支持在线搜索：

[https://linux.die.net/man/3/ibv\_post\_send](https://link.zhihu.com/?target=https%3A//linux.die.net/man/3/ibv\_post\_send)

<figure><img src="https://pic1.zhimg.com/v2-fdcc59e57e76573f74554d74408b6dcc_b.jpg" alt=""><figcaption></figcaption></figure>

#### 查阅系统man page <a href="#h_329198771_17" id="h_329198771_17"></a>

如果你使用的商用OS安装了rdma-core或者libibverbs库，那么可以直接用man命令查询接口：

```
man ibv_post_send
```

<figure><img src="https://pic1.zhimg.com/v2-dc2440d3806dd71dfe729564b7f219a0_b.jpg" alt=""><figcaption></figcaption></figure>

#### 查询Mellanox的编程手册 <a href="#h_329198771_18" id="h_329198771_18"></a>

《RDMA Aware Networks Programming User Manual Rev 1.7》，最新版是2015年更新的。该手册写的比较详尽，并且附有示例程序，但是可能与最新的接口有一些差异。

#### 包含头文件 <a href="#h_329198771_19" id="h_329198771_19"></a>

按需包含以下头文件：

```c
#include <infiniband/verbs.h>   // IB_VERBS 基础头文件
#include <rdma/rdma_cma.h>      // RDMA_CM CMA 头文件 用于CM建链
#include <rdma/rdma_verbs.h>    // RDMA_CM VERBS 头文件 用于使用基于CM的Verbs接口
```

#### 编写应用 <a href="#h_329198771_20" id="h_329198771_20"></a>

下面附上一个简单的RDMA程序的大致接口调用流程，Client端的程序会发送一个SEND请求给Server端的程序，图中的接口上文中都有简单介绍。

需要注意的是图中的建链过程是为了交换对端的GID，QPN等信息，可以通过传统的Socket接口实现，也可以通过本文中介绍的CMA接口实现。

图中特意列出了多次modify QP的流程，一方面是把建链之后交互得到的信息存入QPC中（即QP间建立连接的过程），另一方面是为了使QP处于具备收/发能力状态才能进行下一步的数据交互。具体状态机的内容请回顾[9. RDMA之Queue Pair](https://zhuanlan.zhihu.com/p/195757767)

<figure><img src="https://pic2.zhimg.com/v2-2701af8f6ad58d44faccc4264cff0351_b.jpg" alt=""><figcaption></figcaption></figure>

#### 编译 & 执行 <a href="#h_329198771_21" id="h_329198771_21"></a>

不在本文讨论范围内。

#### 官方示例程序 <a href="#h_329198771_22" id="h_329198771_22"></a>

rdma-core的源码目录下，为libibverbs和librdmacm都提供了简单的示例程序，大家编程时可以参考。

#### libibverbs <a href="#h_329198771_23" id="h_329198771_23"></a>

位于rdma-core/libibverbs/examples/目录下，都使用最基础的IB\_VERBS接口实现，所以建链方式都是基于Socket的。

* asyncwatch.c 查询指定RDMA设备是否有异步事件上报
* device\_list.c 列出本端RDMA设备列表
* devinfo.c 查询并打印本端RDMA设备详细信息，没有双端数据交互
* rc\_pingpong.c 基于RC服务类型双端数据收发示例
* srq\_pingpong.c 基于RC服务类型双端数据收发示例，与上一个示例程序的差异是使用了SRQ而不是普通的RQ。
* ud\_pingpong.c 基于UD服务类型双端数据收发示例
* ud\_pingpong.c 基于UC服务类型双端数据收发示例
* xsrq\_pingpong.c 基于XRC服务类型双端数据收发示例

#### librdmacm <a href="#h_329198771_24" id="h_329198771_24"></a>

位于rdma-core/librdmacm/examples/目录下：

* rdma\_client/server.c 基础示例，通过CM建链并使用CM VERBS进行数据收发。

该目录剩下的程序就没有研究了。

### 参考文献 <a href="#h_329198771_25" id="h_329198771_25"></a>

\[1] RDMA Aware Networks Programming User Manual Rev 1.7

\[2] part1-OFA\_Training\_Sept\_2016

\[3] [https://en.wikipedia.org/wiki/O](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/OpenFabrics\_Alliance)
