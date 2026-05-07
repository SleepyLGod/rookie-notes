---
description: "A Distributed Storage System for Structured Data, the first paper in Google's classic infrastructure trilogy."
---

# Bigtable

> This note is mainly my understanding of the paper.

> As one of Google's three classic big-data infrastructure systems, Bigtable was built on top of Google File System, Chubby, and SSTable.
>
> It was designed to address the fact that different Google products had different requirements for storage capacity and response latency.
>
> The goal was to **store very large amounts of data while keeping query latency low**.
>
> Apache HBase was heavily influenced by Bigtable's design.

## **Data Model**

Data is stored in multiple `Table`s. Each `Cell` in a `Table` is identified by the intersection of row, column, and timestamp:

`(row:string, column:string, time:int64) -> string`. The row, column, and timestamp form three dimensions. The values are byte strings, with a maximum size of 64 KB.

Suppose we want to keep a large copy of web pages and related information that can be used by many different projects. Let us call this table `Webtable`. In `Webtable`, we use URLs as row keys, different aspects of a web page as column names, and store the page content under the `contents:` column with the timestamp of the fetch.

![image-20220711123729151](https://s2.loli.net/2022/07/24/XIviLCBwFZnVbQ2.png)

The figure shows part of an example table for storing web pages. Row names are reversed URLs. The `contents` column family stores page content, and the `anchor` column family stores the text of anchors that reference the page. CNN's home page is referenced by `Sports Illustrated` and the `MY-look` home page, so the row contains columns named `anchor:cnnsi.com` and `anchor:my.look.ca`. Each anchor cell has one version. The contents column has three versions with timestamps `t3`, `t5`, and `t6`.

**ROW**

Every read or write under a single row key is atomic, regardless of how many different columns are read or written in that row.

**---->** This makes it easier for clients to reason about concurrent updates to the same row.

When `Bigtable` stores data, it sorts the `Table` lexicographically by each `Cell`'s `Row Key`.

Bigtable supports row-level transactions, but not cross-row transactions, similar to `MongoDB` in this respect.

Bigtable splits a `Table` by row into multiple adjacent Tablets, and assigns those Tablets to different Tablet Servers for storage.

**----->** When a client queries nearby row keys, the corresponding cells are more likely to be located in the same Tablet, and the query can be more efficient.

**COLUMN**

Bigtable controls access to a Table by Column Family, where each Column Family contains multiple Columns.

All data stored in the same column family usually belongs to the same type. Data in the same column family is also compressed together.

A column family must be created before data can be stored under any column key in that family. After the family is created, any column key inside that family can be used.

The number of different column families in a table is small, at most hundreds, and column families rarely change during operation. By contrast, a table may have an unbounded number of columns.

A Column Key has the form `family:qualifier`.

Before using a table, users must first declare which Column Families exist in that table. After declaration, arbitrary Columns can be created inside that Column Family.

Because data in the same Column Family usually belongs to the same type, Bigtable also combines and compresses data belonging to the same Column Family.

Because Bigtable lets users set access permissions by Column Family, data-analysis jobs sometimes read data from one Column Family and write the computed result into another Column Family.

**TIMESTAMPS**

Different cells in a Table can store multiple versions of the same data, distinguished by timestamp.

A timestamp is essentially a 64-bit integer. Bigtable can automatically set it to the current write time in microseconds, or the application can set it manually. If the application sets it manually, it must ensure that there are no conflicts between cells.

For cells with the same Row Key and Column Key, Bigtable sorts versions by timestamp in descending order, so the newest data is read first.

On top of this, users can configure Bigtable to keep only the most recent N versions, or only versions whose timestamps fall inside a specified time range.

## **Building Blocks**

* Bigtable uses the distributed Google File System (**GFS**) to store logs and data files.
  * It depends on a cluster-management system for scheduling jobs, managing resources on shared machines, handling machine failures, and monitoring machine state.
* The **Google SSTable** file format is used internally to store Bigtable data.
  * **SSTable provides a persistent, ordered, immutable** mapping from keys to values, where both keys and values are **arbitrary byte strings**.
  * It supports looking up the value associated with a specified key, and iterating over **all** key/value pairs in a specified key range.
  * Each **SSTable contains a sequence of blocks**, usually 64 KB each, although the block size is configurable.
  * A **block index**, stored at the end of the SSTable, is used to locate blocks. When an SSTable is opened, the index is loaded into **memory**. A lookup can be performed with **one disk seek**:

      First, Bigtable performs a binary search in the in-memory index to **find the appropriate block**, and then reads that block from disk.

      Optionally, an SSTable **can be mapped entirely into memory**, which allows lookups and scans without touching disk.
* Bigtable depends on a highly available and persistent distributed lock service called **Chubby**.
  * Chubby has **five** active replicas, one of which is elected as the **master** and actively serves requests.
  * The service is live when a majority of replicas are running and can communicate with each other.
  * Chubby keeps replicas consistent in the face of failures using the Paxos algorithm.
  * Its namespace consists of directories and small files. A directory or file can be used as a lock, and file reads and writes are atomic.
  * The client library provides a consistent cache of Chubby files. Each client has a session with the service:

      If the client cannot renew its lease before the lease expires, the client session expires and it loses all locks and open handles.
  * Clients can also **register callbacks** on files and directories to be notified about changes or session expiration.
  * Bigtable uses Chubby for several tasks:
    * Ensuring that at most one active Master exists at any time.
    * Storing the bootstrap location of Bigtable data, namely the Root tablet.
    * Discovering Tablet Servers and detecting their failure.
    * Storing schema information, such as the column families of each table.
    * Storing access control lists.
  * If Chubby is unavailable for an extended period, whether because of a Chubby outage or network problems, Bigtable becomes unavailable.

## **System Architecture**

There are three main implementation components: the client library linked into every client, the Master server, and Tablet Servers.

The client library stores tablet locations.

A complete Bigtable cluster consists of two kinds of nodes: the Master and Tablet Servers.

**Master**:

* Detects which Tablet Servers are in the cluster, and detects their join and leave events.
* Assigns Tablets to Tablet Servers.
* Balances storage load across Tablet Servers.
* Reclaims unused files from GFS.
* Manages schema changes, such as creating and deleting Tables and Column Families.

**Tablet Server** (can be added or removed):

* Manages a set of Tablets assigned by the Master, usually 10 to 1000 Tablets.
* Handles read and write requests for those Tablets.
* Splits a Tablet when it becomes too large.
* Acts like a single-node distributed-storage component. Client data does not move through the Master. Clients communicate directly with Tablet Servers for reads and writes. Since clients do not depend on the Master to locate tablets on every operation, the Master is not on the data path.

A cluster contains multiple Tables. Each Table consists of multiple Tablets. Each Tablet is associated with a specific **Row Key range**, and contains all data in that range for the Table.

Initially, a Table has only one Tablet. As the Tablet grows, Tablet Servers automatically split it, and the Table gradually contains more Tablets.

## **Tablet Location**

Bigtable organizes Tablets into a three-level structure similar to a B+ tree:

* A file in Chubby stores the location of the Root Tablet.
* The Root Tablet stores a special `METADATA` Table, which contains the locations of all Tablets.
* `METADATA` Tablets store the locations of the Tablets for all other Tables.

![](https://s2.loli.net/2022/07/24/2wi9V6HhdtnToa7.jpg)

It is worth noting that the Root Tablet is special: **no matter how large it grows, it is never split, so it remains unique**.

Each row in `METADATA` represents one Tablet of another Table in Bigtable. Its **Row Key is encoded from the Tablet's Table name identifier and its upper Row Key bound**. In addition to Tablet-location information, the `METADATA` table also stores other useful **metadata**, such as event logs for the Tablet, for example when a server started serving it.

When a client wants to locate a Tablet, it recursively follows this hierarchy **downward** to find the location, and caches the intermediate results **in its own memory**.

If at some point the client discovers that a cached address is no longer valid, it recursively walks back up through the hierarchy and then back down again to find the needed Tablet location.

If the client's cache is empty, the location algorithm needs three network round trips, including one read from Chubby. If the client's cache is stale, the algorithm may require up to six round trips, because stale cache entries are discovered only on misses, assuming `METADATA` Tablets do not move frequently.

## **Cluster Membership Changes and Tablet Assignment**

The `Master` uses Chubby to detect when Tablet Servers join or leave the cluster.

Each Tablet Server has a corresponding **unique** file in Chubby. When a Tablet Server starts, it obtains an exclusive lock on that file in Chubby. The Master watches the file's **parent** directory to detect Tablet Server joins. If a Tablet Server loses the exclusive lock, the Master considers it to have left the cluster. Even so, as long as the file still exists, the Tablet Server keeps trying to reacquire its lock. If the file has been deleted, as described below, the Tablet Server shuts itself down.

After the Master knows which Tablet Servers are in the cluster, it must assign Tablets to Tablet Servers. At any moment, a Tablet can be assigned to only one Tablet Server. The Master assigns a Tablet by sending a Tablet-load request to a Tablet Server. Unless the request was not received by the Tablet Server before the Master failed, the assignment can be considered successful. A Tablet Server accepts requests only from the current Master. When a Tablet Server decides it will no longer serve a Tablet, it also sends a request to **notify the Master**.

After the Master detects a Tablet Server failure, meaning the exclusive lock is lost, it reassigns the Tablets served by that Tablet Server. To do this, the Master tries to acquire the exclusive lock on the failed Tablet Server's file in Chubby, and **after successfully acquiring it, deletes** the file to ensure the Tablet Server can shut down correctly. After that, the Master can safely assign the Tablets to other Tablet Servers.

If the communication connection between the **Master and Chubby** is lost, the Master assumes it has failed and shuts itself down. After Master failure, a new Master recovers as follows:

* It acquires the Master's unique lock in Chubby, ensuring that no other Master starts at the same time.
* It uses **Chubby** to obtain the still-live Tablet Servers.
* It **gets** the list of Tablets served by each Tablet Server and announces itself as the new Master, ensuring that future communication from Tablet Servers goes to this new Master.
* The Master ensures that the Root Tablet and the Tablets of the `METADATA` table have been assigned.
* The Master scans the `METADATA` table to obtain all Tablets in the cluster, and reassigns any unassigned Tablets.

## **Tablet Reads, Writes, and Maintenance**

As described above, Tablet data is actually stored in GFS, which provides redundant storage. The following figure shows Tablet data reads and writes:

![](https://s2.loli.net/2022/07/24/hPolgFQUrGR5OxK.jpg)

A **Tablet consists of several SSTable files stored on GFS**, an in-memory MemTable, and a Commit Log.

**Write** operation:

* Bigtable first writes to the WAL, or Write-Ahead Log, by recording the change in the Commit Log.
* Then the inserted data enters a MemTable, and the MemTable keeps its **internal data ordered**.
* Persisted data is stored as SSTable files in GFS.

**Read** operation:

* The Tablet Server performs the corresponding permission checks.
* It first tries to obtain the required latest data from the MemTable.
* If it cannot find the data there, it searches the SSTables.

When a Tablet Server receives an operation request, it also checks whether the requesting user has sufficient permissions. The list of allowed users is stored in a Chubby file.

When a Tablet Server loads a Tablet, it first reads the Tablet's SSTable files and Commit Log information from the metadata table, and uses the entries in the Commit Log to recover the Tablet's MemTable.

Both MemTable and SSTable follow an **immutable-data** design:

* New entries produced by modifications are inserted into the MemTable in a copy-on-write style.
* Bigtable's **Minor Compaction** works as follows:

After the number of entries in the MemTable reaches a threshold, Bigtable writes new requests into another MemTable and starts writing the old MemTable to a new SSTable file. For old data that already exists in previous SSTable files, Bigtable does not remove it at this point.

Every Minor Compaction produces a new SSTable file. Too many SSTable files force later read operations to scan more SSTables in order to obtain the latest correct data. To limit the number of SSTable files, Bigtable periodically performs **Merging Compaction**, which merges data from several SSTables and a MemTable **as-is** into one SSTable.

Bigtable also periodically runs a special kind of Merging Compaction called **Major Compaction**. During this process, Bigtable not only merges several SSTables into one SSTable, but also **removes entries that were marked invalid by later updates or deletions**.

### **Additional Optimizations**

The main optimizations Google added to make Bigtable practically usable in performance and availability are as follows.

#### **Locality Group**

Bigtable allows clients to specify a Locality Group for a Column Family, and to configure the actual file storage format and compression method based on that Locality Group.

During compaction, Bigtable generates separate SSTable files for **each Locality Group in a Tablet**. This lets users place **Column Families that are rarely accessed together into different Locality Groups**, improving query efficiency. Bigtable also provides other tuning parameters based on Locality Groups, such as configuring a Locality Group to be in-memory.

For compression, Bigtable lets users specify **whether** data in a Locality Group should be compressed and **which compression format** should be used. It is worth noting that Bigtable compresses SSTables at the **Block** level rather than compressing the entire file directly. Although this reduces compression efficiency, it also means that when users read data, Bigtable **only needs to decompress selected SSTable Blocks**.

#### **Read Cache and Bloom Filter**

Bigtable uses an LSM-tree storage approach:

* It converts random disk writes into sequential writes, at the cost of lower read performance.
* The reason is that Bigtable files are actually stored in GFS, and GFS is mainly optimized for sequential writes while supporting random writes poorly.
* Therefore, after using an LSM-tree to preserve write performance, Bigtable must use other techniques to preserve read performance. The first technique is **read caching**.
* Overall, Bigtable's read cache consists of **two cache layers**: **Scan Cache and Block Cache**.
  * Block Cache caches **SSTable file Blocks read from GFS**, improving the efficiency of reading data near a previously read item.
  * Scan Cache sits above Block Cache and caches **key/value pairs returned from SSTables to Tablet Servers**, improving the efficiency of repeatedly reading the same data.

In addition, to improve lookup efficiency, Bigtable allows users to enable a **Bloom Filter** for a Locality Group. By spending some memory to store Bloom Filters built for SSTable files, Bigtable can use Bloom Filters during record lookup to quickly **exclude** SSTables that cannot contain the record, reducing the number of SSTable files that must be read.

#### **Commit Log**

Bigtable uses Write-Ahead Logging to keep data highly available, which means it performs many Commit Log writes.

First, if Bigtable used a separate Commit Log for each Tablet, the system would have many Commit Log files being **written at the same time**, increasing the seek overhead of the underlying disk. To avoid this, a Tablet Server writes all Tablet write operations it receives into **one shared Commit Log file**.

This design creates another problem. If the Tablet Server goes down, the Tablets it served may be reassigned to several other Tablet Servers. During MemTable recovery, those servers may repeatedly read the Commit Log produced by the old Tablet Server. To solve this, **before a Tablet Server reads the Commit Log, it sends a signal to the Master**, and the Master **initiates a sorting operation over the original Commit Log**:

The original Commit Log is split into multiple 64 MB pieces, and each piece is sorted concurrently by **`(table, row name, log sequence number)`**. After sorting finishes, a Tablet Server reading the Commit Log only needs to read the part it needs, reducing repeated reads.
