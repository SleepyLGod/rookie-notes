---
description: 'PROJECT #4 - CONCURRENCY CONTROL'
---

# Concurrency Control

> This project mainly implements the database lock manager and supports concurrent query execution. The lock manager manages tuple-level locks issued by transactions, and grants or releases shared and exclusive locks according to each transaction's isolation level.

* [**Task #1 - Lock Manager**](https://15445.courses.cs.cmu.edu/fall2020/project4/#lock\_manager)
* [**Task #2 - Deadlock Detection**](https://15445.courses.cs.cmu.edu/fall2020/project4/#deadlock\_detection)
* [**Task #3 - Concurrent Query Execution**](https://15445.courses.cs.cmu.edu/fall2020/project4/#execution\_engine)

### TASK #1 - LOCK MANAGER

The following describes the implementation ideas for the functions required by the lock manager. Note that each function below should first acquire the lock manager's `latch_`.

| <pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
</code></pre> | <pre class="language-cpp"><code class="lang-cpp">bool LockManager::LockShared(Transaction *txn, const RID &#x26;rid) {
    // Note that: READ UNCOMMITTED no S-Lock.
    /**
     * 1. Checking txn state. Txn should abort if its state isn't growing.
     * 2. Append lock request to this RID request queue. If the rid exist X-Lock previous, 
     *	  then txn blocked(lock_table_[rid].cv_.wait()). Otherwise S-Lock can granted.
     * 3. The S-Lock request is granted.
     */
}
bool LockManager::LockExclusive(Transaction *txn, const RID &#x26;rid) {
    // Note that: READ UNCOMMITTED no S-Lock.
    /**
     * 1. Checking txn state. Txn should abort if its state isn't growing.
     * 2. Append lock request to this RID request queue. If the rid exist S/X-Lock previous,
     *	  then txn blocked. Otherwise X-Lock can granted.
     * 3. X-Lock can be acquired only if txn in front of queue. Otherwise, it will be blocked.
     * 4. the X-Lock request is granted.
     */
}
bool LockManager::LockUpgrade(Transaction *txn, const RID &#x26;rid) {
    /**
     * 1.1 Checking txn state. Txn should abort if txn's state isn't growing.
     * 1.2 Whether another txn already ready upgrading.
     * 2. find correct postion to upgrade S-Lock to X-Lock
     * 3. Update txn request lock message.
     */
}
bool LockManager::Unlock(Transaction *txn, const RID &#x26;rid) {
    /**
     * 1. If txn doesn't have any this tuple lock, then return false.
     * 2. Delete this txn in request queue, and release tuple lock hold by txn.
     * 3. txn's state is growing when txn release lock, then set txn state is shrinking,
     *	  so txn can't acquire any lock. Note that: it is used in REPEATABLE READS isolation level
     */
}
</code></pre> |
| ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

***

### TASK #2 - DEADLOCK DETECTION

1. To detect deadlocked transactions, first build a wait-for graph.
2. Then run a DFS-based cycle detection algorithm to check whether the graph contains a cycle.

If a deadlocked transaction is found and aborted, how should other transactions be notified so they can continue acquiring tuple locks? Here I used an idea provided by classmates:

> 1. When entering condition-variable waiting, store a `txn_id_t -> RID` mapping in a hash table.
> 2. After finding a deadlock node, set it to `Aborted`, wake up the RID request queue corresponding to `txn_id_t`, and check the transaction state after wakeup. If the transaction state is `Aborted`, throw an exception.

***

### TASK #3 - CONCURRENT QUERY EXECUTION

* For `SeqScan`: if the isolation level is `READ_UNCOMMITTED`, no lock is needed. If the isolation level is `READ_COMMITTED` or `REPEATABLE_READ`, acquire an `S-Lock` when accessing a RID. The difference is the release timing: under `READ_COMMITTED`, the `S-Lock` can usually be released after the current tuple is read; under `REPEATABLE_READ`, read locks should be held until the transaction ends, so repeated reads within the same transaction do not change.
* For `Delete` and `Update`: if the current RID already has an `S-Lock`, upgrade it to an `X-Lock`; otherwise acquire an `X-Lock` on this RID.
* For `Unlock`: whether releasing a lock moves the transaction into the shrinking state must be understood together with the isolation level. Under `REPEATABLE_READ`, a transaction cannot acquire new locks after releasing a lock. Under `READ_COMMITTED`, read locks may be released earlier, but write locks should still be held until the transaction ends.

***

### Testing / Verification / Packaging

* Tests

| <pre><code>1
2
3
4
5
</code></pre> | <pre><code>cd build
make lock_manager_test
make transaction_test
./test/lock_manager_test
./test/transaction_test
</code></pre> |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |

* Format checks

| <pre><code>1
2
3
</code></pre> | <pre><code>make format
make check-lint
make check-clang-tidy
</code></pre> |
| ------------------------------ | -------------------------------------------------------------------------- |

* Packaging

| <pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
</code></pre> | <pre><code>zip project4-submission.zip src/include/buffer/lru_replacer.h src/buffer/lru_replacer.cpp \
	src/include/buffer/buffer_pool_manager.h src/buffer/buffer_pool_manager.cpp \
	src/include/storage/page/b_plus_tree_page.h src/storage/page/b_plus_tree_page.cpp \
	src/include/storage/page/b_plus_tree_internal_page.h src/storage/page/b_plus_tree_internal_page.cpp \
	src/include/storage/page/b_plus_tree_leaf_page.h src/storage/page/b_plus_tree_leaf_page.cpp \
	src/include/storage/index/b_plus_tree.h src/storage/index/b_plus_tree.cpp \
	src/include/storage/index/index_iterator.h src/storage/index/index_iterator.cpp \
	src/include/catalog/catalog.h src/include/execution/execution_engine.h src/include/storage/index/index.h \
	src/include/execution/executor_factory.h src/execution/executor_factory.cpp \
	src/include/execution/executors/seq_scan_executor.h src/execution/seq_scan_executor.cpp \
	src/include/execution/executors/index_scan_executor.h src/execution/index_scan_executor.cpp \
	src/include/execution/executors/insert_executor.h src/execution/insert_executor.cpp \
	src/include/execution/executors/update_executor.h src/execution/update_executor.cpp \
	src/include/execution/executors/delete_executor.h src/execution/delete_executor.cpp \
	src/include/execution/executors/nested_loop_join_executor.h src/execution/nested_loop_join_executor.cpp \
	src/include/execution/executors/nested_index_join_executor.h src/execution/nested_index_join_executor.cpp \
	src/include/execution/executors/limit_executor.h src/execution/limit_executor.cpp \
	src/include/execution/executors/aggregation_executor.h src/execution/aggregation_executor.cpp \
	src/include/storage/index/b_plus_tree_index.h src/storage/index/b_plus_tree_index.cpp \
	src/concurrency/lock_manager.cpp src/include/concurrency/lock_manager.h
</code></pre> |
| --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

Then go to [**Gradescope**](https://www.gradescope.com/) and submit the code.
