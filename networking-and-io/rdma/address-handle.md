# 😁 Address Handle

As introduced earlier, the basic unit of RDMA communication is the QP. Consider this question: if a QP on node A wants to exchange information with a QP on node B, besides knowing the QP number of node B, namely QPN, what other information is needed? QPN is maintained independently by each node; it is not unique across the whole network. For example, if QP 3 on A wants to communicate with QP 5 on B, there is not only one QP 5 in the network. Many nodes may each have their own QP 5. Therefore, naturally, each node also needs an independent identifier.

In the traditional TCP/IP protocol stack, the familiar IP address identifies each node at the network layer. In the IB protocol, this identifier is called **GID (Global Identifier)** and is a 128-bit sequence. This note does not expand on GID; it will be introduced later.

### What Is AH?

AH stands for Address Handle. Here, the address refers to a collection of information used to locate a remote node. In the IB protocol, address information includes GID, port number, and other fields. A handle can be understood as a pointer to an object.

Recall that the IB protocol has four basic service types: RC, UD, RD, and UC. The most commonly used ones are RC and UD. In RC, a reliable connection is established between two nodes' QPs. Once the connection relationship is established, it does not change easily, and peer information is stored in the QP Context when the QP is created.

For UD, there is no connection relationship between QPs. If the user wants to send to someone, the peer address information is filled into the WQE. **However, the user does not directly fill peer address information into the WQE. Instead, the user prepares an "address book" in advance and specifies the peer node address by index each time. That index is the AH.**

The AH concept can be roughly shown as follows:

<figure><img src="https://pic2.zhimg.com/v2-3779a6c1466cac62517263ed61a21d49_b.jpg" alt=""><figcaption></figcaption></figure>

For each destination node, the local side creates a corresponding AH, and the same AH can be shared by multiple QPs.

### What AH Does

Before each UD service-type communication, the user must use the interface provided by the IB framework to **create an AH for each possible peer node**. These AHs are then placed by the driver into a "safe" region, and an index, pointer, or handle is returned to the user. When the user actually posts a WR (Work Request), passing this index is enough.

The process above is shown below. Node A receives a task from the user: use local QP4 to exchange data with QP3 on node B, where B is specified through the AH.

<figure><img src="https://pic3.zhimg.com/v2-35ba165fc28068426bfe48d2ab8f510a_b.jpg" alt=""><figcaption></figcaption></figure>

The IB protocol does not explain why AH is used. I think the concept of AH exists for three reasons:

1. Ensure destination-address usability and improve efficiency

Because UD is connectionless, users can directly specify the destination through a WR in userspace. If users were allowed to freely fill address information and hardware assembled packets based on that information, problems would arise. For example, the user might tell hardware through a WR: send data to port Z of the node with GID X and MAC address Y. However, X, Y, and Z may not be a legal combination, or the node with GID X may not exist in the network at all. Hardware cannot validate all of this; it would only obediently assemble and send the packet, wasting bandwidth on an invalid destination.

Preparing address information in advance avoids this situation. When the user creates an AH, the call enters kernel space. If the parameters passed by the user are valid, the kernel stores the destination-node information and returns a pointer to the user. If the parameters are invalid, AH creation fails. This process guarantees that the address information is valid. The user can then quickly specify the destination node through the pointer, accelerating data exchange.

One might ask: since the kernel is trusted, why not enter kernel space during data transmission to validate the address information passed by the user? Do not forget one of RDMA's major advantages: the data path can go directly from userspace to hardware and completely bypass the kernel, avoiding system-call and copy overhead. If every send had to validate address legality, communication speed would inevitably decrease.

2. Hide low-level address details from users

When creating an AH, the user only needs to pass information such as GID, port number, and static rate. Other address information required for communication, mainly the MAC address, is resolved by the kernel driver through mechanisms such as querying the system neighbor table. There is no need to expose this extra low-level information to userspace.

3. Use PD to manage destination addresses

When introducing Protection Domain earlier, we mentioned that AH, like QP and MR, is also partitioned by PD. After defining AH as a software entity, we can isolate and manage all destinations reachable by QPs.

<figure><img src="https://pic2.zhimg.com/v2-95491ba30503d10d9406e3bea6845805_b.jpg" alt=""><figcaption></figcaption></figure>

For example, in the figure above, AH1 through AH3 can only be used by QP3 and QP9 under the same PD, while AH4 can only be used by QP5.

### Protocol Sections

The protocol does not spend much space on AH, and it does not even have a standalone section introducing the concept:

\[1] 9.8.3: what parts compose the destination address in the UD service type, including AH, QPN, and Q_Key.

\[2] 10.2.2.2: notes related to destination addresses.

\[3] 11.2.2.1: AH-related Verbs interfaces.

\


That is all for AH. Thanks for reading. The next note describes more details about QP.
