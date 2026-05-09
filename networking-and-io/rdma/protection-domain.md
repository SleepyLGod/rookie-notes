# ☺ Protection Domain

Previous notes briefly introduced several common RDMA resources, including different queues and the concept of MR. MR controls and manages HCA access permissions to local and remote memory, ensuring that the HCA can read or write user-registered memory regions only after obtaining the correct key. To provide better safety, the IB protocol introduces the concept of Protection Domain (PD), which ensures isolation between RDMA resources. This note introduces PD.

### What Is PD?

PD stands for Protection Domain. The concept of domain appears often: in mathematics, there are real-number domains and complex-number domains; in geography, there are airspace and sea areas. A domain represents a space or range. In RDMA, a PD is like a "container" that holds various resources, such as QPs and MRs, and brings them within its protection scope to prevent unauthorized access. A node can define multiple protection domains. Resources contained in different PDs are isolated from each other and cannot be used together.

This concept may still feel abstract, so look at what PD does and what problem it solves.

### What PD Does

One user may create multiple QPs and multiple MRs. Each QP may establish connections with different remote QPs, as shown below. The gray arrows represent connection relationships between QPs:

<figure><img src="https://pic4.zhimg.com/v2-2743ce566dd7634b25c7bbcab579fa5f_b.jpg" alt=""><figcaption></figcaption></figure>

Because there is no binding relationship between MR and QP, once a remote QP establishes a connection with a local QP and can communicate, then in theory the remote node can access the content of some MR on the local node as long as it knows the VA and R_Key, or even keeps guessing until it obtains a valid pair.

In ordinary cases, the virtual address VA and R_Key of an MR are difficult to guess and already provide a certain level of safety. However, to better protect data in memory and further isolate and divide permissions between different resources, each node can define PDs, as shown below:

<figure><img src="https://pic3.zhimg.com/v2-ad65c4e7f0a2b1b7b4deb4ad2426f29e_b.jpg" alt=""><figcaption></figcaption></figure>

In the figure, Node 0 has two PDs that divide three QPs and two MRs into two groups. Node 1 and Node 2 each have one PD containing all QPs and MRs. Resources in the two PDs on Node 0 cannot be used together. That is, QP3 and QP9 cannot access data in MR1, and QP6 cannot access data in MR0. If hardware is asked to use QP3 and MR1 during data send/receive, it checks that they do not belong to the same PD and returns an error.

For a remote node, Node 1 can access Node 0's memory only through QP3 connected to QP8. But because QP3 on Node 0 is "enclosed" in protection domain PD0, QP8 on Node 1 can only access the memory corresponding to MR0 and **cannot access data in MR1 under any circumstances**. This is restricted in two ways:

1. QP8 on Node 1 only has a connection relationship with QP3 on Node 0, so it cannot access memory through QP6 on Node 0.
2. MR1 and QP3 on Node 0 belong to different PDs. Even if QP8 on Node 1 obtains the VA and R_Key of MR1, hardware rejects service because the PDs differ.

Therefore, as stated at the beginning of this note, PD is like a container that protects some RDMA resources and isolates them from each other to improve safety. RDMA contains not only QP and MR resources. Address Handle, Memory Window, and other resources introduced later are also isolated and protected by PD.

### How to Use PD

Look again at the figure above. Node 0 has two PDs to isolate resources, while Node 1 and Node 2 each have only one PD containing all resources.

The reason for drawing it this way is to show that how many PDs are defined on a node is entirely decided by the user. **If stronger safety is desired, each QP connected to a remote node and each MR exposed for remote access should be isolated through PD partitioning as much as possible. If higher safety is not required, it is also valid to create one PD containing all resources.**

The IB protocol specifies that **each node must have at least one PD, each QP must belong to one PD, and each MR must belong to one PD.**

How is the PD containment relationship represented in software? PD itself has a software entity, a structure, that records information about this protection domain. Before creating resources such as QPs and MRs, the user must first create a PD through the IB framework interface and obtain its pointer or handle. Later, when creating QPs and MRs, this PD pointer or handle is passed in, so PD information is included in those QPs and MRs. During packet send/receive, hardware verifies the PDs of the QP and MR. More details about the software protocol stack will be introduced later.

Another important point: **PD is a local concept and exists only inside a node**. It is invisible to other nodes. MR, by contrast, is visible to both local and remote sides.

For easier lookup and study, future notes will list the protocol sections involved. Earlier notes may also be supplemented later when there is time.

### PD-Related Protocol Sections

* 3.5.5: basic concept and role of PD.\

* 10.2.3: relationship between PD and other RDMA resources, and PD-related software interfaces.\

* 10.6.3.5: again emphasizes the relationship between PD, MR, and QP.
* 11.2.1.5: detailed introduction to PD Verbs interfaces, including function, input parameters, output parameters, and return value.

That is all for PD. The next note introduces Address Handle, which is used by the UD service type.

### References

[infinibandta.org/ibta-s](http://link.zhihu.com/?target=https%3A//www.infinibandta.org/ibta-specifications-download/)\
This is the IB specification. The PDF can be downloaded from the IBTA official website, and most content is in volume 1.

Besides the IB protocol, the OFA training slides are also recommended: [openfabrics.org/trainin](http://link.zhihu.com/?target=https%3A//www.openfabrics.org/training/). In addition, see Mellanox's programming manual _RDMA Aware Networks Programming User Manual_ and the RDMA blog [rdmamojo.com/](http://link.zhihu.com/?target=https%3A//www.rdmamojo.com/).
