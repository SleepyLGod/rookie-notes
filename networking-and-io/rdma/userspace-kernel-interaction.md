# 🥲 Userspace and Kernel Interaction

In the note "[**Verbs**](verbs.md)", we discussed that Verbs APIs are divided into userspace and kernel-space APIs, using the `ibv_` and `ib_` prefixes respectively. The biggest advantage of RDMA is that userspace can bypass the kernel and directly control hardware for data transmission, reducing system calls and memory copies. Therefore, most RDMA applications are userspace applications that use userspace Verbs APIs prefixed with `ibv_`.

However, not every userspace Verbs API can completely bypass the kernel. This note explains which APIs depend on the kernel RDMA subsystem, including drivers, why they need to depend on the kernel, and how userspace and the kernel interact.

### Verbs Categories <a href="#h_346708569_0" id="h_346708569_0"></a>

Section 11.1.2.3 of the IB specification divides Verbs users into two categories. One category is privileged users who can directly access OS internal data and control RDMA hardware; they can use all Verbs. The other category is user-level users who must rely on an agent to access OS data structures; they can only use a small subset of Verbs.

In plain terms, kernel-space Verbs users have the highest privileges, so they can directly access all RDMA resources. Userspace Verbs users can use only some interfaces to interact directly with hardware, while most Verbs APIs must enter kernel space through system calls or similar mechanisms.

Table 95 in the IB specification lists the implementation requirements and required user privileges for all Verbs. For implementation requirements, Mandatory means the software must support it, while other values mean the software can choose whether to support it. For user privileges, Privileged means special privileges are required, and User-Level means ordinary privileges are sufficient.

<figure><img src="https://pic2.zhimg.com/v2-633af4500c805132e8f40eb7c5a36511_b.jpg" alt=""><figcaption></figcaption></figure>

By observing the table, we can see that aside from interfaces used for data exchange, such as posting WRs (Post Send and Post Recv) and obtaining WCs (Poll CQ and Request Completion Notification), as well as operations related to Bind MW and AH, all other operations require privilege. In other words, calling the corresponding Verbs APIs requires entering kernel space.

### RDMA Software Stack <a href="#h_346708569_1" id="h_346708569_1"></a>

To make the following explanation easier, this section uses the Mellanox driver as an example and gives a rough architecture of the RDMA software stack. Later notes can describe this part in more detail.

<figure><img src="https://pic3.zhimg.com/v2-ce501eb019ef6e034b487470efae41a6_b.jpg" alt=""><figcaption></figcaption></figure>

First, read the diagram from top to bottom:

#### Userspace <a href="#h_346708569_2" id="h_346708569_2"></a>

* Application\
  RDMA applications, such as `perftest`, middleware such as UCX, and other user programs.\

* libibverbs.so\
  The core userspace dynamic library of the RDMA software stack. Its responsibilities are:

1. Implement and expose various Verbs APIs to upper-layer applications.
2. Call hook functions registered by vendor drivers inside the logic of different Verbs APIs.
3. Provide interfaces for entering kernel space.

* libmlx5.so\
  The userspace driver for Mellanox ConnectX-5 NICs. It is also a dynamic library and implements vendor-specific driver logic.

#### Kernel <a href="#h_346708569_3" id="h_346708569_3"></a>

* Intermediate interaction module\
  This module handles userspace system-call requests through the ABI. When userspace verbs enter the kernel, the `ib_uverbs` module in this layer parses the command. The `xxx.ko` on the right refers to a custom module used by upper-layer programs that use kernel Verbs interfaces, such as `ib_post_send`, to handle system calls.\

* ib_core.ko\
  The core module of the kernel RDMA subsystem. Its responsibilities are:

1. Provide kernel-space Verbs APIs to applications using kernel-space Verbs.
2. Call hook functions registered by vendor drivers inside the logic of different Verbs APIs.
3. Manage various RDMA resources and provide services to userspace.

* mlx5_ib.ko\
  The kernel-space driver module for Mellanox ConnectX-5 NICs, responsible for directly interacting with hardware.\

* Hardware\
  The Mellanox ConnectX-5 NIC.

Next, use the red dashed line as the boundary and read the diagram left and right. The left side is the hierarchy of userspace applications, while the right side is the hierarchy of kernel-space users. Userspace applications have two paths: the left path bypasses the kernel and represents the data path; the right path goes through the kernel and represents the control path.

**Strictly speaking, the term "kernel-space application" is not correct, because applications run in userspace. Here, "kernel-space application" refers to an application that indirectly uses kernel Verbs APIs through an intermediate module. This distinguishes it from most RDMA applications that use userspace Verbs APIs.**

### Why Enter Kernel Space? <a href="#h_346708569_4" id="h_346708569_4"></a>

As described in the first section, operations on the control path need to enter kernel space, while operations on the data path usually do not. Why must some Verbs enter kernel space? There are two main reasons:

**1. Userspace is not safe, and some resources must not be exposed for userspace modification**

Userspace `.so` libraries can be replaced by ordinary users. Components of rdma-core, including libibverbs, librdmacm, and userspace driver code, can all be modified. If RDMA resources were exposed to userspace, such as pointers to kernel QP structures or addresses of QPC configuration information, users who do not follow the IB specification, or even malicious users, could modify key information and create security problems. Therefore, userspace can only obtain RDMA resource handles. During communication with the kernel, the kernel uses mechanisms such as `idr` to recover the concrete QP pointer and continue the operation.

In addition, the kernel can track the usage of different RDMA resources and release those resources when a userspace program exits abnormally.

**2. Static virtual-to-physical address mappings must be established**

Userspace I/O posts virtual addresses directly to hardware inside WRs. Therefore, hardware must be able to find the physical memory addresses corresponding to those user virtual addresses. During MR registration, the kernel driver builds address-mapping tables for the hardware. To prevent paging from changing the virtual-to-physical mapping, the kernel driver triggers pinning, which fixes this mapping relationship.

Entering kernel space specifically means switching from user context to kernel context through a system call. This involves context switching, privilege checks, and other actions, so it has a certain time cost.

Some Verbs do not have the constraints above. For example, posting WQEs by directly reading and writing hardware registers mapped into userspace does not require modifying RDMA resources or managing memory mappings. Naturally, such operations do not need to enter the kernel. This is one of RDMA's advantages: bypassing the kernel on the data path. Compared with traditional socket data exchange, this saves the time spent switching back and forth between userspace and kernel space for every data send or receive operation.

Because of the overhead of entering the kernel, some people call the Verbs path that needs to enter kernel space the "slow path", and the path that does not need to enter kernel space the "fast path".

### How Userspace and Kernel Space Communicate <a href="#h_346708569_5" id="h_346708569_5"></a>

On the control path, userspace and kernel space mainly communicate by using the `write()` system call on the `/dev/infiniband/uverbsN` character device file. Recent protocol stacks also support the `ioctl()` system call, but I have not studied it in depth, so this note does not discuss it. To explain how userspace and kernel space communicate, it is necessary to distinguish two concepts: API and ABI.

#### API <a href="#h_346708569_6" id="h_346708569_6"></a>

An API (Application Programming Interface) is a programming interface between programs. The Verbs interface is a set of APIs. For example, an application may contain code like this:

```c
#include <verbs.h>

int send_message(struct msg *)
{
    ...
    struct ibv_qp *qp = ibv_create_qp(pd, init_attr);
    ...
}
```

Because the definition of `ibv_create_qp()` in the `verbs.h` header, including its name, parameters, and return value, has not changed across versions of the verbs library:

```c
/**
 * ibv_create_qp - Create a queue pair.
 */
struct ibv_qp *ibv_create_qp(struct ibv_pd *pd,
                 struct ibv_qp_init_attr *qp_init_attr);
```

the **source code of this program does not need any modification. It can be compiled against any version of the userspace library, rdma-core, to produce an executable application file**.

#### ABI <a href="#h_346708569_7" id="h_346708569_7"></a>

ABI (Application Binary Interface) is the binary interface between programs. In the RDMA software-stack diagram in this note, the `uverbs` interface between userspace and the kernel is an ABI. An ABI defines the runtime communication format between programs, such as how parameters are passed, whether they are placed in specific registers or on the stack, what format they use, and where return values are stored.

The uverbs ABI defines the format of command messages, `cmd`, and return messages, `resp`, between userspace and kernel space. Conceptually, it looks like this:

<figure><img src="https://pic3.zhimg.com/v2-707cb36bdde7f6f9233276a5fba71f86_b.jpg" alt=""><figcaption></figcaption></figure>

In the note "[RDMA Verbs](https://zhuanlan.zhihu.com/p/329198771)", we introduced userspace libraries and kernel drivers. They are released on their own schedules. Interaction between userspace and kernel space involves many command transfers, and the command formats differ across versions. The RDMA software stack uses the uverbs ABI to **guarantee compatibility between different userspace and kernel-space versions, so a userspace library of one version can run directly on many kernel versions**.

Still using Create QP as an example, the software stack defines the `cmd` and `resp` of `ibv_create_qp()` as follows:

<figure><img src="https://pic2.zhimg.com/v2-68e4a52e1f1f9fcb7eae66c3b56e04a9_b.png" alt=""><figcaption></figcaption></figure>

The `cmd` contains three parts:

* Command code: tells the kernel which operation is requested after entering kernel space.
* Common field section: parameters that every vendor's Create QP operation needs to pass from userspace to kernel space.
* Driver-defined field section: vendor-specific parameters that need to be passed into the kernel.

<figure><img src="https://pic1.zhimg.com/v2-ff733309bee722cddd263f58183cd6b0_b.png" alt=""><figcaption></figcaption></figure>

The `resp` contains two parts:

* Common field section: parameters that every vendor needs to return to userspace after creating the QP in the kernel.
* Driver-defined field section: vendor-specific return parameters.

The formats above are defined by the uverbs ABI. More specifically, the entire interaction mechanism between userspace and the kernel is implemented by cooperation between the kernel's `ib_uverbs.ko` and userspace's `libibverbs.so`.

In practice, except for vendor driver developers, RDMA application developers and ordinary users do not need to care about the ABI implementation. They only need to care about the API.

### An Example <a href="#h_346708569_8" id="h_346708569_8"></a>

Now connect the whole process and look at what happens after a Verbs API is called.

#### Verbs Interfaces That Need to Enter Kernel Space <a href="#h_346708569_9" id="h_346708569_9"></a>

Interfaces that need to enter kernel space take the "slow path" indicated by the red arrows:

<figure><img src="https://pic2.zhimg.com/v2-521f31811b625ba238f35a3d3cd4f105_b.jpg" alt=""><figcaption></figcaption></figure>

#### ibv_open_device() <a href="#h_346708569_10" id="h_346708569_10"></a>

As its name suggests, this interface opens a device. In plain terms, it mainly completes the following tasks:

* Create the `device_context` structure

This structure can be understood as the software representation of the device. Later code uses this structure to refer to the device we want to use.

* Map PCIe BAR space so the user can obtain the Doorbell address

The `mmap` mechanism here is relatively complex. The general idea is to allow userspace to directly read and write certain NIC registers to interact with hardware. Among these registers mapped into userspace, the most important one is Doorbell. As the name suggests, it is a notification mechanism. After the user prepares a WR, writing data to the Doorbell address is equivalent to ringing the doorbell, so hardware knows it can take the WQE from the QP and start working.

* Attach callback functions to the IB framework

Because each vendor's hardware and software implementation is different, the IB framework exposes many hook functions. Different hardware models will route execution into different vendor drivers.

* Query device capabilities

The user can learn the capabilities and specifications supported by the current hardware, such as how many QPs or CQs it supports.

The call stack for this process is shown below. Only key functions are listed, and the red dashed line indicates entering kernel space from userspace:

<figure><img src="https://pic2.zhimg.com/v2-db552eb9fc9dd2593409bbecde64cce1_b.png" alt=""><figcaption></figcaption></figure>

#### ibv_reg_mr() <a href="#h_346708569_11" id="h_346708569_11"></a>

This interface registers an MR. MRs have already been introduced in earlier notes, so this section does not repeat that explanation. This step mainly completes two tasks:

* Pin memory

This prevents memory containing critical data from being swapped out to disk by the system, which would cause hardware to read unexpected memory during data transmission.

* Establish a virtual-to-physical address mapping table

Only after this table is established can hardware find the actual physical address from the virtual address in the WQE.

The call stack for this process is shown below:

<figure><img src="https://pic1.zhimg.com/v2-ed3969e613fe87c08cdd36cfec38a6ec_b.png" alt=""><figcaption></figcaption></figure>

#### ibv_create_qp() <a href="#h_346708569_12" id="h_346708569_12"></a>

This interface appears many times in these notes. It creates a QP and mainly completes the following tasks:

* Validate and adjust QP parameters according to hardware capabilities

It checks whether the user-provided creation parameters are reasonable. Because of hardware limits, the actually allocated specification may be larger than the user's requested specification. If the parameters are valid, the actual parameters used are returned to the user.

* Create QP-related resources

As introduced in the QP note, this includes resources such as QPC and QPN.

* Allocate the QP buffer

This buffer mainly stores SQ WQEs, RQ WQEs, and related data, meaning the QP's own queue memory space. After creation, the driver tells hardware the QP base address, so hardware knows where to find the QP and its attributes.

The call stack for this process is shown below:

<figure><img src="https://pic2.zhimg.com/v2-ccb0c80420d9af6196c2bb609f48bf15_b.png" alt=""><figcaption></figcaption></figure>

#### Interfaces That Do Not Need to Enter Kernel Space <a href="#h_346708569_13" id="h_346708569_13"></a>

Verbs interfaces that do not need to enter the kernel take the "fast path" indicated by the red arrow on the left:

<figure><img src="https://pic4.zhimg.com/v2-971266d709efec393959cd620ba556cf_b.jpg" alt=""><figcaption></figcaption></figure>

#### ibv_post_send() <a href="#h_346708569_14" id="h_346708569_14"></a>

This interface has also been discussed several times. Its function is to let the user post a WR to hardware. Suppose the user posts a SEND WQE. What exactly happens during this process?

* Obtain the starting memory address of the next WQE from the QP buffer, which was allocated by `ibv_create_qp()`.
* Parse the content of the WR according to the structure agreed with hardware and fill it into the WQE.
* The data region is specified through an SGE. The memory pointed to by the SGE is inside the MR registered by `ibv_reg_mr()`.
* After filling the WQE, ring the Doorbell to notify hardware. The Doorbell address was mapped by `ibv_open_device()`.
* Hardware takes the WQE from the QP buffer and parses its content.
* Hardware uses the mapping table established by `ibv_reg_mr()` to translate the virtual address storing data into a physical address, and then fetches the data.
* Hardware assembles packets and sends the data.

This example shows that **RDMA kernel bypass does not mean the whole flow bypasses the kernel. Instead, the control path enters the kernel multiple times for preparation. Only after everything is ready can the data path avoid the cost of entering the kernel**.

\
