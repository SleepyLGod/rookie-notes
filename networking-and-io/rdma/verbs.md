# 😆 Verbs

### **Overview** <a href="#h_329198771_0" id="h_329198771_0"></a>

The word "verbs" literally means actions. In the RDMA domain, it has two meanings:

1. A set of abstract definitions described by the IB specification. These definitions specify the actions or behaviors that vendors' software and hardware should perform under different Verbs. The IB specification does not prescribe how to implement these Verbs in software. Under this meaning, Verbs are independent of any operating system.

For example, the IB specification requires all RDMA devices to support the Create QP behavior, in IB specification Section 11.2.5.1:

> Description:\
> Create a QP for the specified device.\
> The user must specify a set of attributes used to initialize the QP.\
> If the attributes required to create the QP contain illegal values or are missing, an error should be returned and the QP should not be created. If creation succeeds, the pointer to the QP and the QPN are returned.\
> ...\
> Input:\
> Device pointer;\
> CQ associated with SQ;\
> CQ associated with RQ. If this is an XRC INI QP, this parameter may be omitted;\
> ...\
> Output:\
> Pointer to the newly created QP;\
> QP Number;\
> Maximum WR capacity of the SQ;\
> ...\
>

From this, we can see that Verbs in the IB specification are like textbook definitions of concepts. They describe "what must be supported", but do not specify exactly how to implement it.

2. A set of RDMA application programming interfaces, or APIs, promoted and implemented by OpenFabrics. Since these are APIs, they are necessarily related to the operating system on which they run. Verbs APIs have Linux and Windows versions, although the Windows version has not been updated for a long time.

Again using Create QP as an example, the following is quoted from the Linux userspace Verbs API help document, [ibv_create_qp(3): create/destroy queue pair](https://link.zhihu.com/?target=https%3A//linux.die.net/man/3/ibv\_create\_qp):

> Name:\
> ibv_create_qp - create a queue pair (QP)\
> Synopsis:

```cpp
#include <infiniband/verbs.h>
struct ibv_qp *ibv_create_qp(struct ibv_pd *pd, struct ibv_qp_init_attr *qp_init_attr);
```

> Description:\
> `ibv_create_qp()` creates a QP associated with a PD. The `qp_init_attr` parameter is a structure of type `ibv_qp_init_attr`, defined in `<infiniband/verbs.h>`.

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

> The function `ibv_create_qp()` updates the content of the `qp_init_attr->cap` structure and returns the actual capabilities supported by the created QP.\
> Return value:\
> `ibv_create_qp()` returns a pointer to the created QP, or `NULL` on failure. The QPN is available in the structure pointed to by the returned pointer.

Therefore, the Verbs API is the concrete software implementation of the Verbs definitions in the IB specification.

For the first meaning of Verbs, directly read Chapter 11 of the IB specification. It provides a very detailed description.

This note discusses the second meaning: what the Verbs API is, how it interacts with hardware, and how to write RDMA programs through Verbs APIs. Unless otherwise stated, "Verbs" below specifically means Verbs API.

### Terminology <a href="#h_329198771_1" id="h_329198771_1"></a>

#### rdma-core <a href="#h_329198771_2" id="h_329198771_2"></a>

`rdma-core` is the **open-source RDMA userspace software stack**. It includes the userspace framework, vendor userspace drivers, API manuals, development tests, and self-test tools.

`rdma-core` is maintained on GitHub. The userspace Verbs API we use is implemented by this project.

#### kernel RDMA subsystem <a href="#h_329198771_3" id="h_329198771_3"></a>

The kernel RDMA subsystem is the **open-source RDMA subsystem in the Linux kernel**. It includes the RDMA kernel framework and vendor drivers.

The RDMA subsystem is maintained as part of Linux. It provides kernel-space Verbs APIs on one side, and interfaces with userspace on the other.

#### OFED <a href="#h_329198771_4" id="h_329198771_4"></a>

OFED stands for OpenFabrics Enterprise Distribution. It is an **open-source software package collection** that includes kernel frameworks and drivers, userspace frameworks and drivers, middleware, test tools, and API documentation.

Open-source OFED is developed, released, and maintained by the OFA organization. It periodically takes software versions from `rdma-core` and the kernel RDMA subsystem, adapts them to commercial OS distributions, and releases them. In addition to the protocol stack and drivers, it includes test tools such as `perftest`.

The following figure is an OFED overview provided by OFA:

<figure><img src="https://pic4.zhimg.com/v2-11ed2af97e47dc17339b49e890ef31b3_b.jpg" alt=""><figcaption></figcaption></figure>

In addition to open-source OFED, vendors also provide customized OFED packages, such as Huawei's HW_OFED and Mellanox's MLNX_OFED. These customized versions are developed from open-source OFED and are tested and maintained by the vendors themselves. They provide private enhancements on top of the open-source package and include vendor-specific configuration and test tools.

These three components are in a containment relationship. Whether in userspace or kernel space, the RDMA community is very active. The frameworks change almost every day and are released roughly every two months on average. OFED periodically takes code from the two communities, performs functionality and compatibility testing, and then releases versions on a much longer schedule, usually measured in years.

### What Is the Verbs API? <a href="#h_329198771_5" id="h_329198771_5"></a>

The Verbs API is the most basic software interface set for using RDMA services. In the industry, RDMA applications are either written directly on this API set, or written on top of middleware that wraps another interface layer around the Verbs API.

The Verbs API exposes all RDMA functionality to users. Typical operations include registering MRs, creating QPs, posting sends, polling CQs, and so on.

On Linux, Verbs functionality is provided by `rdma-core` and the kernel RDMA subsystem. It is divided into userspace Verbs interfaces and kernel-space Verbs interfaces, used by userspace and kernel-space RDMA applications respectively.

Combining this with the previous section, the whole OFED picture is:

<figure><img src="https://pic1.zhimg.com/v2-75fb344eaf89d646650d7d9605d8af6c_b.jpg" alt=""><figcaption></figcaption></figure>

Broadly speaking, the Verbs API mainly consists of two parts:

#### IB_VERBS <a href="#h_329198771_6" id="h_329198771_6"></a>

Interfaces use the `ibv_xx` prefix in userspace and the `ib_xx` prefix in kernel space. These are the most fundamental programming interfaces, and `IB_VERBS` alone is enough to write RDMA applications.

Examples:

* `ibv_create_qp()` creates a QP.
* `ibv_post_send()` posts a Send WR.
* `ibv_poll_cq()` polls CQEs from a CQ.

#### RDMA_CM <a href="#h_329198771_7" id="h_329198771_7"></a>

Interfaces with the `rdma_` prefix mainly provide two functions:

#### CMA (Connection Management Abstraction) <a href="#h_329198771_8" id="h_329198771_8"></a>

CMA is a set of interfaces implemented on top of sockets and Verbs APIs. It is used for CM connection setup and information exchange. CM connection setup is implemented by encapsulating QP setup on top of sockets. From the user's perspective, after QP exchange it exchanges the information required for data exchange, such as QPN and keys.

Examples:

* `rdma_listen()` listens for CM connection requests on a link.
* `rdma_connect()` confirms a CM connection.

#### CM VERBS <a href="#h_329198771_9" id="h_329198771_9"></a>

`RDMA_CM` can also be used for data exchange. It is equivalent to wrapping another data-exchange interface layer around the Verbs API.

Examples:

* `rdma_post_read()` can directly post a WR for an RDMA READ operation. By contrast, `ibv_post_send()` requires the operation type READ to be specified in the parameters.
* `rdma_post_ud_send()` can directly pass information such as the remote QPN, the AH pointing to the remote endpoint, and the local buffer pointer to trigger a UD SEND operation.

These interfaces are convenient, but they must be used together with links managed by CMA and cannot be mixed freely with raw Verbs APIs.

Besides `IB_VERBS` and `RDMA_CM`, the Verbs API also includes interfaces such as MAD (Management Datagram).

**Note that the concrete implementation of Verbs APIs in the software stack is not exactly the same as the description in the IB specification. The IB specification evolves slowly, while the software stack changes almost every day. Therefore, when writing applications or drivers, use the software-stack API documentation as the source of truth.**

Narrowly speaking, the Verbs API specifically refers to userspace Verbs interfaces prefixed with `ibv_` or kernel-space interfaces prefixed with `ib_`. Because typical RDMA applications run in userspace, the following sections mainly introduce the userspace Verbs API.

### Why Design the Verbs API? <a href="#h_329198771_10" id="h_329198771_10"></a>

Traditional Ethernet users write applications based on the Socket API. **RDMA users write applications based on the Verbs API.**

The Verbs API supports the three major RDMA protocols: IB, iWARP, and RoCE. Through a unified interface, the same RDMA program can run in different environments without caring about differences in underlying hardware and links.

### What the Verbs API Contains <a href="#h_329198771_11" id="h_329198771_11"></a>

The userspace Verbs API mainly contains two layers of functionality.

For easier explanation, the following interface forms are simplified into the format "_return value 1, return value 2_ **function_name**(_parameter 1, parameter 2_)".

1. Control plane:

* Device management:\

*
  * _device_list_ **ibv_get_device_list**()

The user obtains the list of available RDMA devices. It returns a set of pointers to available devices.

*
  * _device_context_ **ibv_open_device**(_device_)

Open an available RDMA device and return its context pointer. This pointer will later be used for various operations on the device.

*
  * _device_attr, errno_ **ibv_query_device**(_device_context_)

Query the attributes or capabilities of a device, such as the maximum number of QPs and CQs it supports. It returns a pointer to the device attribute structure and an error code.

* Resource creation, query, modification, and destruction:\

*
  * _pd_ **ibv_alloc_pd**(_device_context_)

Allocate a PD. This function returns a pointer to the PD.

*
  * _mr_ **ibv_reg_mr**(_pd, addr, length, access_flag_)

Register an MR. The user passes the starting address and length of the memory to be registered, the PD to which this MR belongs, and its access permissions, such as local read/write and remote read/write. The function returns a pointer to the MR.

*
  * _cq_ **ibv_create_cq**(_device_context, cqe_depth, ..._)

Create a CQ. The user passes the minimum CQ depth, although the driver may actually allocate a larger one. The function returns a pointer to the CQ.

*
  * _qp_ **ibv_create_qp**(_pd, qp_init_attr_)

Create a QP. The user passes the PD and a set of attributes, including the CQs bound to RQ and SQ, the SRQ bound to the QP, QP capabilities, QP type, and other information. The function returns a pointer to the QP.

*
  * _errno_ **ibv_modify_qp**(_qp, attr, attr_mask_)

Modify a QP. The user passes a pointer to the QP, a mask indicating which attributes to modify, and the new values. Modifiable content includes QP state, peer QPN, QP capability, port number, retry count, and so on. If it fails, the function returns an error code.\
The most important role of Modify QP is to move the QP between different states. Only after completing the RST -> INIT -> RTR -> RTS state-machine transition does the QP gain the ability to post Send WRs. It can also be used to switch the QP into the ERROR state.

*
  * _errno_ **ibv_destroy_qp**(_qp_)

Destroy a QP, meaning destroy QP-related software resources. Other resources also have similar destroy interfaces.

* Interrupt handling:\

*
  * _event_info, errno_ **ibv_get_async_event**(_device_context_)

Obtain an asynchronous event from the event queue. It returns asynchronous-event information, such as event source and event type, and an error code.

* Connection management\

*
  * rdma_xxx()

Used for CM connection setup. This note does not expand on it.

*
  * ...

2. Data plane:

* Posting WRs\

*
  * _bad_wr, errno_ **ibv_post_send**(_qp, wr_)

Post a Send WR to a QP. The parameter `wr` is a structure that contains all information for the WR, including `wr_id`, SGE count, opcode such as SEND/WRITE/READ and more specific operation types. The WR structure differs by service type and operation type. For example, WRITE and READ operations under the RC service include the remote memory address and R_Key, while UD service WRs include AH, remote QPN, Q_Key, and other information.

After further driver processing, the WR is converted into a WQE and posted to hardware.

On error, this interface returns a pointer to the failed WR and an error code.

*
  * _bad_wr, errno_ **ibv_post_recv**(_qp, wr_)

This is similar to `ibv_post_send`, except that it is specifically used to post RECV WRs.

* Obtaining WCs\

*
  * _num, wc_ **ibv_poll_cq**(_cq, max_num_)

Poll CQEs from the Completion Queue. The user must prepare memory in advance to store WCs and pass in how many WCs can be received. This interface returns a set of WC structures, whose contents include `wr_id`, status, opcode, QPN, and other information, as well as the number of WCs returned.

### Writing RDMA Applications with the Verbs API <a href="#h_329198771_12" id="h_329198771_12"></a>

#### Checking Interface Definitions <a href="#h_329198771_13" id="h_329198771_13"></a>

#### Kernel Space <a href="#h_329198771_14" id="h_329198771_14"></a>

Kernel-space Verbs interfaces do not have a dedicated API manual. When programming, refer to function comments in the header files. The header that declares these interfaces is located in the kernel source tree:

.../include/rdma/ib_verbs.h

For example, the `ib_post_send()` interface:

<figure><img src="https://pic2.zhimg.com/v2-e21e86b7b2193758b2f4d2f5096825a1_b.jpg" alt=""><figcaption></figcaption></figure>

The function comments clearly describe the function's purpose, input parameters, output parameters, and return value.

#### Userspace <a href="#h_329198771_15" id="h_329198771_15"></a>

There are several ways to consult userspace Verbs APIs:

#### Read the Latest Man Pages Online <a href="#h_329198771_16" id="h_329198771_16"></a>

The userspace Verbs API manuals are maintained in the same repository as the code. The manual path is:

[https://github.com/linux-rdma/rdma-core/tree/master/libibverbs/man](https://link.zhihu.com/?target=https%3A//github.com/linux-rdma/rdma-core/tree/master/libibverbs/man)

These files are written in Linux man-page source format, so reading the source directly may not be intuitive. Many online man-page websites can be used to read these interface descriptions. For example, the official link:

[https://man7.org/linux/man-pages/man3/ibv_post_send.3.html](https://link.zhihu.com/?target=https%3A//man7.org/linux/man-pages/man3/ibv\_post\_send.3.html)

There are also other unofficial websites that support online search:

[https://linux.die.net/man/3/ibv_post_send](https://link.zhihu.com/?target=https%3A//linux.die.net/man/3/ibv\_post\_send)

<figure><img src="https://pic1.zhimg.com/v2-fdcc59e57e76573f74554d74408b6dcc_b.jpg" alt=""><figcaption></figcaption></figure>

#### Read System Man Pages <a href="#h_329198771_17" id="h_329198771_17"></a>

If your commercial OS has installed `rdma-core` or the `libibverbs` library, you can query interfaces directly with the `man` command:

```
man ibv_post_send
```

<figure><img src="https://pic1.zhimg.com/v2-dc2440d3806dd71dfe729564b7f219a0_b.jpg" alt=""><figcaption></figcaption></figure>

#### Read the Mellanox Programming Manual <a href="#h_329198771_18" id="h_329198771_18"></a>

The manual is _RDMA Aware Networks Programming User Manual Rev 1.7_. Its latest version was updated in 2015. The manual is detailed and includes example programs, but some content may differ from the latest interfaces.

#### Include Header Files <a href="#h_329198771_19" id="h_329198771_19"></a>

Include the following headers as needed:

```c
#include <infiniband/verbs.h>   // Basic IB_VERBS header
#include <rdma/rdma_cma.h>      // RDMA_CM CMA header for CM connection setup
#include <rdma/rdma_verbs.h>    // RDMA_CM VERBS header for CM-based Verbs interfaces
```

#### Writing an Application <a href="#h_329198771_20" id="h_329198771_20"></a>

The following figure shows the rough interface-call flow for a simple RDMA program. The client program sends a SEND request to the server program, and the interfaces in the figure have all been briefly introduced above.

Note that the connection setup process in the figure is used to exchange peer information such as GID and QPN. It can be implemented through traditional socket interfaces or through the CMA interface introduced in this note.

The figure intentionally lists multiple Modify QP steps. One purpose is to store the information exchanged after connection setup into the QPC, meaning the process of establishing a connection between QPs. Another purpose is to move the QP into a state with send/receive capability before the next data-exchange step. For state-machine details, review [9. RDMA Queue Pair](https://zhuanlan.zhihu.com/p/195757767).

<figure><img src="https://pic2.zhimg.com/v2-2701af8f6ad58d44faccc4264cff0351_b.jpg" alt=""><figcaption></figcaption></figure>

#### Compilation and Execution <a href="#h_329198771_21" id="h_329198771_21"></a>

This is outside the scope of this note.

#### Official Example Programs <a href="#h_329198771_22" id="h_329198771_22"></a>

The `rdma-core` source tree provides simple example programs for both `libibverbs` and `librdmacm`. They are useful references when writing programs.

#### libibverbs <a href="#h_329198771_23" id="h_329198771_23"></a>

These examples are located under `rdma-core/libibverbs/examples/`. They all use the most basic `IB_VERBS` interfaces, so their connection setup is socket-based.

* `asyncwatch.c`: query whether the specified RDMA device reports asynchronous events.
* `device_list.c`: list local RDMA devices.
* `devinfo.c`: query and print detailed local RDMA device information; it performs no two-sided data exchange.
* `rc_pingpong.c`: two-sided data send/receive example based on the RC service type.
* `srq_pingpong.c`: two-sided data send/receive example based on the RC service type. The difference from the previous example is that it uses SRQ instead of ordinary RQ.
* `ud_pingpong.c`: two-sided data send/receive example based on the UD service type.
* `ud_pingpong.c`: two-sided data send/receive example based on the UC service type.
* `xsrq_pingpong.c`: two-sided data send/receive example based on the XRC service type.

#### librdmacm <a href="#h_329198771_24" id="h_329198771_24"></a>

These examples are located under `rdma-core/librdmacm/examples/`:

* `rdma_client/server.c`: basic example that sets up a connection through CM and uses CM VERBS for data send and receive.

I have not studied the remaining programs in this directory.

### References <a href="#h_329198771_25" id="h_329198771_25"></a>

\[1] RDMA Aware Networks Programming User Manual Rev 1.7

\[2] part1-OFA_Training_Sept_2016

\[3] [https://en.wikipedia.org/wiki/O](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/OpenFabrics\_Alliance)
