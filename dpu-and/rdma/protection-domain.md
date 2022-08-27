# ☺ Protection Domain

前文我们简单介绍了RDMA中最常见的一些资源，包括各种Queue，以及MR的概念等等。MR用于控制和管理HCA对于本端和远端内存的访问权限，确保HCA只有拿到正确Key之后才能读写用户已经注册了的内存区域。为了更好的保障安全性，IB协议又提出了Protection Domain（PD）的概念，用于保证RDMA资源间的相互隔离，本文就介绍一下PD的概念。

### PD是什么

PD全称是Protection Domain，意为"保护域"。域的概念我们经常见到，从数学上的“实数域”、“复数域”，到地理上的“空域”、“海域”等等，表示一个空间/范围。在RDMA中，PD像是一个容纳了各种资源（QP、MR等）的“容器”，将这些资源纳入自己的保护范围内，避免他们被未经授权的访问。一个节点中可以定义多个保护域，各个PD所容纳的资源彼此隔离，无法一起使用。

概念还是有些抽象，下面我们来看一下PD有什么作用，具体解决了什么问题。

### PD的作用

一个用户可能创建多个QP和多个MR，每个QP可能和不同的远端QP建立了连接，比如下图这样（灰色箭头表示QP间的连接关系）：

<figure><img src="https://pic4.zhimg.com/v2-2743ce566dd7634b25c7bbcab579fa5f_b.jpg" alt=""><figcaption></figcaption></figure>

由于MR和QP之间并没有绑定关系，这就意味着一旦某个远端的QP与本端的一个QP建立了连接，具备了通信的条件，那么理论上远端节点只要知道VA和R\_key（甚至可以靠不断的猜测直到得到一对有效的值），就可以访问本端节点某个MR的内容。

其实一般情况下，MR的虚拟地址VA和秘钥R\_Key是很难猜到的，已经可以保证一定的安全性了。但是为了更好的保护内存中的数据，把各种资源的权限做进一步的隔离和划分，我们在又在每个节点中定义了PD，如下图所示：

<figure><img src="https://pic3.zhimg.com/v2-ad65c4e7f0a2b1b7b4deb4ad2426f29e_b.jpg" alt=""><figcaption></figcaption></figure>

图中Node 0上有两个PD，将3个QP和2个MR分为了两组，此外Node 1和Node 2中各有一个PD包含了所有QP和MR。Node 0上的两个PD中的资源不可以一起使用，也就是说QP3和QP9不能访问MR1的数据，QP6也不可以访问MR0的数据。如果我们在数据收发时，指定硬件使用QP3和MR1，那么硬件校验他们不属于同一个PD后，会返回错误。

对于远端节点来说，Node1只能通过QP8相连的QP3来访问Node0的内存，但是因为Node 0的QP3被“圈”到了PD0这个保护域中，所以Node 1的QP8也只能访问MR0对应的内存，**无论如何都无法访问MR1中的数据**，这是从两个方面限制的：

1. Node 1的QP8只跟Node 0的QP3有连接关系，无法通过Node 0的QP6进行内存访问。
2. Node 0的MR1和QP3属于不同的PD，就算Node 1的QP8拿到了MR1的VA和R\_key，硬件也会因为PD不同而拒绝提供服务。

所以就如本文一开始所说的，PD就像是一个容器，将一些RDMA资源保护起来，彼此隔离，以提高安全性。其实RDMA中不止有QP、MR这些资源，后文即将介绍的Address Handle，Memory Window等也是由PD进行隔离保护的。

### 如何使用PD

还是看上面的图，我们注意到Node 0为了隔离资源，存在两个PD；而Node 1和Node 2只有一个PD包含了所有资源。

我之所以这样画，是为了说明一个节点上划分多少个PD完全是由用户决定的，**如果想提高安全性，那么对每个连接到远端节点的QP和供远端访问的MR都应该尽量通过划分PD做到隔离；如果不追求更高的安全性，那么创建一个PD，囊括所有的资源也是可以的**。

IB协议中规定：**每个节点都至少要有一个PD，每个QP都必须属于一个PD，每个MR也必须属于一个PD。**

那么PD的包含关系在软件上是如何体现的呢？它本身是有一个软件实体的（结构体），记录了这个保护域的一些信息。用户在创建QP和MR等资源之前，必须先通过IB框架的接口创建一个PD，拿到它的指针/句柄。接下来在创建QP和MR的时候，需要传入这个PD的指针/句柄，PD信息就会包含在QP和MR中。硬件收发包时，会对QP和MR的PD进行校验。更多的软件协议栈的内容，我会在后面的文章中介绍。

另外需要强调的是，**PD是本地概念，仅存在于节点内部**，对其他节点是不可见的；而MR是对本端和对端都可见的。

为了方便大家查阅和学习，以后我会列出文章涉及的协议章节，前面的内容有时间的时候我也会补充一下。

### PD相关协议章节

* 3.5.5 PD的基本概念和作用\

* 10.2.3 介绍了PD和其他一些RDMA资源的关系，以及PD相关的软件接口。\

* 10.6.3.5 再次强调PD和MR及QP的关系。
* 11.2.1.5 详细介绍PD的Verbs接口，包括作用、入参、出参和返回值等。

好了，关于PD的介绍就到这里。下文我会介绍用于UD服务类型的Address Handle的概念。

### 参考

[infinibandta.org/ibta-s](http://link.zhihu.com/?target=https%3A//www.infinibandta.org/ibta-specifications-download/)\
是IB协议，可以从IBTA的官网下载PDF，大部分内容都在volume 1里面

除了IB协议之外，推荐看一下OFA官网的培训PPT：[openfabrics.org/trainin](http://link.zhihu.com/?target=https%3A//www.openfabrics.org/training/)。另外还有Mellanox提供的编程手册《RDMA Aware Networks Programming User Manual》，以及这个老外的写的RDMA博客[rdmamojo.com/](http://link.zhihu.com/?target=https%3A//www.rdmamojo.com/)
