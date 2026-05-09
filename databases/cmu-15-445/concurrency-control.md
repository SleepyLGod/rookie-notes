---
description: 'PROJECT #4 - CONCURRENCY CONTROL'
---

# 😉 Concurrency Control

> 这个 project 主要是实现数据库的锁管理（Lock Manager），并且支持并发查询执行。LM 负责管理事务发出的 tuple-level 锁，并且 LM 还有基于事务的隔离级别实现 Shared & Exclusive 锁的授予和释放。

* [**Task #1 - Lock Manager**](https://15445.courses.cs.cmu.edu/fall2020/project4/#lock\_manager)
* [**Task #2 - Deadlock Detection**](https://15445.courses.cs.cmu.edu/fall2020/project4/#deadlock\_detection)
* [**Task #3 - Concurrent Query Execution**](https://15445.courses.cs.cmu.edu/fall2020/project4/#execution\_engine)

### TASK #1 - LOCK MANAGER

下面主要是 LM 需要实现的函数的思路，注意每个调用下面的每个函数应该先获得 LM 的 `latch_`。

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

1. 为了检测死锁的事务，首先应该构建一个 wait for graph。
2. 然后运行 DFS 判断环的算法检测图中是否存在环。

如果发现死锁的事务，将它 Aborted 后，如何通知其他事务继续获得 tuple-lock 呢？这里采用了群友提供的思路

> 1. 进入条件变量等待时，使用哈希表保存 `txn_id_t -> RID` 的映射。
> 2. 找到死锁节点后，设置为 Aborted，唤醒 `txn_id_t` 对应的 RID 请求队列，唤醒后进行状态检查。如果事务的状态的 Aborted，那么抛出异常。

***

### TASK #3 - CONCURRENT QUERY EXECUTION

* 对于 SeqScan：如果隔离级别是 `READ_UNCOMMITTED`，则不需要加锁；如果隔离级别是 `READ_COMMITTED` 和 `REPEATABLE_READ`，则访问某个 RID 时需要对它加 `S-Lock`。区别在释放时机：`READ_COMMITTED` 通常读完当前 tuple 后即可释放 `S-Lock`；`REPEATABLE_READ` 需要持有读锁直到事务结束，避免同一事务内重复读发生变化。
* 对于 Delete 和 Update：如果当前 RID 拥有 `S-Lock`，则需要将 `S-Lock` 升级为 `X-Lock`，否则对这个 RID 加 `X-Lock`
* 对于 Unlock：释放锁是否让事务进入 shrinking 状态，需要和隔离级别配套理解。`REPEATABLE_READ` 下释放锁后不能再获取新锁；`READ_COMMITTED` 下读锁可以较早释放，但写锁仍应持有到事务结束。

***

### 测试/验证/打包

* 测试

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

* 格式验证

| <pre><code>1
2
3
</code></pre> | <pre><code>make format
make check-lint
make check-clang-tidy
</code></pre> |
| ------------------------------ | -------------------------------------------------------------------------- |

* 打包

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

然后前往 [**Gradescope**](https://www.gradescope.com/) 提交代码
