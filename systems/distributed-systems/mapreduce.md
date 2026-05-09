---
description: The third note in Google's classic systems trilogy
---

# Google MapReduce

## **MapReduce Model**

At a high level, a distributed computation executed by Google MapReduce takes **a set of key/value pairs** as **input** and **produces another set of key/value pairs** as output.

Users specify the computation by writing a `Map` function and a `Reduce` function.

The user-defined `Map` function is applied to every **input** key/value pair and emits zero or more key/value pairs as **intermediate results**.

Then the MapReduce framework **passes all values associated with the same intermediate key `k2` to the same `Reduce` invocation**.

The user-defined `Reduce` function takes **an intermediate key `k2` and the collection of values associated with that key** as arguments, **merges the input values, and emits the merged output values**.

Formally, the user-provided `Map` and `Reduce` functions have the following types:

**`[Math Processing Error]`**

**`map(k1, v1) → list(k2, v2)`**

**`reduce(k2, list(v2)) → list(v2)`**

In the actual implementation, the `MapReduce` framework uses an `Iterator` to represent the input collection. This avoids requiring a potentially large collection to fit entirely in memory.

As an example, consider this problem: given a large collection of documents, compute how many times each word appears. This is the classic Word Count example. The user usually provides pseudocode like this:

```cpp
map(String key, String value):
  // key: document name
  // value: document contents
  for each word w in value:
    EmitIntermediate(w, “1”);

reduce(String key, Iterator values):
  // key: a word
  // values: a list of counts
  int result = 0;
  for each v in values:
    result += ParseInt(v);
  Emit(AsString(result));
```

### **Functional Programming Model**

Readers familiar with functional programming will notice that the MapReduce programming model comes from the `map` and `reduce` functions in functional programming. Later systems such as Spark adopted a similar programming model.

The advantage of this model is that it naturally supports parallel execution. The underlying system can parallelize large-scale computation relatively easily. At the same time, the determinism of user-provided functions allows the system to use re-execution as a primary fault-tolerance mechanism.

## MapReduce Execution Flow

The rough execution flow of a MapReduce job is shown below:

![](https://s2.loli.net/2022/07/24/D8dExufbkRa52eT.png)

First, the user uses the MapReduce **client** to specify the `Map` function, the `Reduce` function, and the configuration for this MapReduce job. The configuration includes the number of intermediate-result partitions `R` and the hash function `hash` used to **partition intermediate results**. After the user starts the MapReduce job, the execution flow can be summarized as follows:

1. The input files are divided into `M` splits. Each split is usually between 16 and 64 MB.
2. The entire MapReduce job therefore contains `M` Map tasks and `R` Reduce tasks. The Master node selects from **idle Worker nodes** and assigns them Map and Reduce tasks.
3. Workers that receive Map tasks, also called **Mappers**, read their corresponding splits, **parse the contents into input key/value pairs**, and call the user-defined **Map function**. The intermediate key/value pairs produced by the Map function are temporarily stored in an **in-memory buffer**.
4.  While the Map phase is running, Mappers **periodically spill the intermediate results from the buffer to their local disks**. They also divide the intermediate results into `R` partitions according to the user-specified `Partition` function, whose default is `hash(key) mod R`.

    When the task completes, the Mapper reports the **locations** of its local intermediate-result files to the Master.
5. The Master forwards the Mapper-reported intermediate-result **locations** to the Reducers. After a Reducer receives this information, it uses **RPC** to read the intermediate results for its partition from the Mapper's local disk. After reading the data, the Reducer **sorts it so key/value pairs with the same key are contiguous**.
6. The Reducer then gathers the values associated with each key and calls the user-defined `Reduce` function. The output of the `Reduce` function is written into the corresponding **Reduce partition output file**.

In a MapReduce cluster:

* The Master records the current completion status of every Map and Reduce task, as well as the Worker assigned to each task.
* The Master is also responsible for forwarding the locations and sizes of intermediate-result files produced by Mappers to Reducers.

It is worth noting that, for each MapReduce job, the values of `M` and `R` should usually be much larger than the number of Workers in the cluster. This helps achieve load balancing across the cluster.

## **MapReduce Fault Tolerance**

Because Google MapReduce relies heavily on distributed atomic file read/write operations provided by the Google File System, the fault-tolerance mechanism of a MapReduce cluster is comparatively simple. It mainly focuses on recovering from unexpected task interruption.

### **Worker Failure**

In a MapReduce cluster, the Master periodically sends **Ping** messages to every Worker. If a Worker does not respond for a period of time, the Master treats it as unavailable.

**Every** Map task assigned to that Worker, whether currently running or already completed, must be reassigned by the Master to another Worker. The reason is that if the Worker is unavailable, then **the intermediate results stored on that Worker's local disk are also unavailable**.

The Master also notifies all Reducers about the retry. Reducers that failed to fully fetch intermediate results from the original Mapper start fetching data from the new Mapper.

If Reduce tasks were assigned to the failed Worker, the Master reassigns only the unfinished Reduce tasks to other Workers. Since Google MapReduce stores final results in the Google File System, completed Reduce-task outputs remain available through GFS. Therefore, the MapReduce Master only needs to handle unfinished Reduce tasks.

### **Master Failure**

There is only one Master node in a MapReduce cluster, so Master failure is uncommon but important.

While running, the Master **periodically writes the current cluster state to disk as a checkpoint**. If the Master process terminates, a restarted Master process can use the data stored on disk to recover to the most recent checkpoint.

### **Straggler Worker**

If a Worker in the cluster **takes an unusually long time to finish the last few Map or Reduce tasks**, the entire MapReduce job is delayed. Such a Worker is called a **Straggler**.

When the overall computation is close to completion, MapReduce creates backup executions for the remaining tasks by assigning them to other idle Workers as well. Once one execution of a task finishes, the task is considered complete.

## **Other Optimizations**

In addition to availability, the Google MapReduce implementation also applies several optimizations to improve overall system efficiency.

### **Data Locality**

In Google's internal computing environment, cross-machine network bandwidth is a relatively scarce resource, so unnecessary data transfer between machines should be minimized.

Google MapReduce uses the Google File System to store input and output data. Therefore, when assigning Map tasks, the Master reads block-location information from GFS and tries to assign each Map task to a machine that holds a replica of the corresponding block. If that is not possible, the Master uses rack-topology information provided by GFS to assign the task to a nearby machine.

### **Combiner**

In some cases, the user-defined Map function may produce many **repeated intermediate keys**, and the user-defined Reduce function may be commutative and associative.

In this case, Google MapReduce allows the user to declare a **Combiner function executed on the Mapper**. The Mapper calls the Combiner over its own `R` intermediate-result partitions to locally merge intermediate results, reducing the amount of data transferred between Mappers and Reducers.
