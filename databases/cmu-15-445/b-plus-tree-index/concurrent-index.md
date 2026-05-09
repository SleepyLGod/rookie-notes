# Concurrent Index

## 5. Implementing the Concurrency Mechanism

### 0. First Review the Reader-Writer Latch Mechanism&#x20;

1. Read operations can share a latch among multiple threads, while write operations must be mutually exclusive.
2. The `MaxReader` limit is added to prevent waiting writer threads from starving.

### **What Can Go Wrong Without a Latch Mechanism**

1. Thread `T1` wants to delete `44`.
2. Thread `T2` wants to search for `41`.

![image-20210126184533688](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201246511-892266003.png)

1. Suppose `T2` reaches node `D`, and the scheduler switches to thread `T1`.
2. `T1` performs redistribution and moves `41` into node `I`.
3. After `T1` finishes, execution switches back to `T2`. If `T2` continues searching along the original path, it will not find `41`.

![image-20210126184727498](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201510577-283160563.png)

The following incorrect state can appear.

![image-20210126184901306](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201708545-1250779663.png)

### **Why Reader-Writer Latches Are Needed**

1. #### For `find`

> Since this is a read-only operation, once we move to the next node, we can release the latch on the previous node.

![image-20210126185917549](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201736266-1524422160.png)

The remaining operations are the same.

### `delete` is different

> Because deletion is a write operation.

Here we cannot release the latch on node `A`, because deletion may merge the root node.

![image-20210126190112632](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201804944-2073658211.png)

When we reach node `D`, we find that deleting `38` from `D` does not require a merge. Therefore, the write latches on `A` and `B` can be safely released.

![image-20210126190229333](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201823967-455255873.png)

### For `Insert`

Here we can safely release the latch on `A`. Since `B` still has free space, insertion will not affect `A`.

![image-20210126190452937](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201845743-1649741202.png)

When execution reaches `D`, we find that `D` is already full. At this point we do not release the latch on `B`, because we may need to write to `B`.

![image-20210126190613339](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201922260-1593188438.png)

The algorithm above is correct, but it has a bottleneck. Only one thread can hold a write latch, and insertion and deletion both need to acquire a write latch on the root. Therefore, when many concurrent insertions or deletions occur, performance drops significantly.

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202003618-880373652.png)

### **Introducing Optimistic Latching**

> The optimistic assumption is that most operations do not need merges or splits. Therefore, while descending the tree, we acquire read latches rather than write latches. Only the leaf page is latched in write mode.

1. Acquire read latches from top to bottom and release them gradually.
2. Acquire a write latch only when the leaf page needs modification. If the deletion is safe, the operation can finish directly.

![image-20210126192408392](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202039219-1666824693.png)

If the final step turns out to be unsafe, rerun the operation using the conservative protocol described above, as if optimistic latching had not been introduced.

![image-20210126192548748](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202108649-568164667.png)

### **Brief Introduction to B-Link Trees**

> Delay updates to parent nodes.

Here a star marks a parent update that is needed but has not yet been performed.

![image-20210126195848104](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202130834-1687270714.png)

At this point, other operations can still execute correctly, such as searching for `31`.

![image-20210126200003320](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202156088-1710986034.png)

Now execute `insert 33`.

When execution reaches node `C`, another thread holds the write latch. At this moment, the starred deferred update needs to be performed, and then `33` can be inserted.

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210127092842951-664251463.png)

**One final note about scan operations**

1. Thread 1 holds a write latch on node `C`.
2. Thread 2 has already scanned node `B` and wants to acquire a read latch on node `C`.

This creates a problem because thread 2 cannot acquire the read latch.

**There are several possible solutions**

1. Wait until `T1` completes its write operation.
2. Restart `T2`.
3. Let `T2` stop trying to acquire this latch.

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202324217-992759171.png)

Note that `Latch` and `Lock` are not the same concept here.

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/754297-20160131225332443-857830570.jpg)

## 6. Helper Function Analysis

### 1. Implementing the helper function `UnlockUnpinPages`

1. If the operation is a read, release the read latch.
2. Otherwise, release the write latch.

{% code lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::
UnlockUnpinPages(Operation op, Transaction *transaction) {
  if (transaction == nullptr) {
    return;
  }

  for (auto page:*transaction->GetPageSet()) {
    if (op == Operation::READ) {
      page->RUnlatch();
      buffer_pool_manager_->UnpinPage(page->GetPageId(), false);
    } else {
      page->WUnlatch();
      buffer_pool_manager_->UnpinPage(page->GetPageId(), true);
    }
  }
  transaction->GetPageSet()->clear();

  for (const auto &page_id: *transaction->GetDeletedPageSet()) {
    buffer_pool_manager_->DeletePage(page_id);
  }
  transaction->GetDeletedPageSet()->clear();

  // if root is locked, unlock it

  node_mutex_.unlock();
  }
```
{% endcode %}

The four built-in latch and unlatch operations are:

{% code lineNumbers="true" %}
```cpp
/** Acquire the page write latch. */
inline void WLatch() { rwlatch_.WLock(); }

/** Release the page write latch. */
inline void WUnlatch() { rwlatch_.WUnlock(); }

/** Acquire the page read latch. */
inline void RLatch() { rwlatch_.RLock(); }

/** Release the page read latch. */
inline void RUnlatch() { rwlatch_.RUnlock(); }
```
{% endcode %}

Here `rwlatch_` is a custom reader-writer latch class. The next section inspects this class.

> I was not fully comfortable with C++ concurrency when writing this note, so this section only gives a lightweight reading. It can be expanded after studying concurrency more deeply.

1.  The `WLock` function

    1. First acquire the mutex.
    2. Use the `writer_entered` flag to indicate whether a writer has entered.
    3. If another writer has already entered, the current thread must wait and remains blocked.
    4. If other threads are currently reading, the writer must still block, because writing is not allowed while others are reading.

    {% code lineNumbers="true" %}
    ```cpp
    void WLock() {
      std::unique_lock<mutex_t> latch(mutex_);
      while (writer_entered_) {
        reader_.wait(latch);
      }
      writer_entered_ = true;
      while (reader_count_ > 0) {
        writer_.wait(latch);
      }
    }
    ```
    {% endcode %}
2.  The `WUnlock` function

    1. Set the writer flag to `false`.
    2. Then notify all waiting reader threads.

    {% code lineNumbers="true" %}
    ```cpp
    void WUnlock() {
      std::lock_guard<mutex_t> guard(mutex_);
      writer_entered_ = false;
      reader_.notify_all();
    }
    ```
    {% endcode %}
3.  The `RLock` function

    1. If someone is currently writing, or if the maximum reader count has already been reached, block.
    2. Otherwise, just increment the reader count.

    This is safe because multiple threads are allowed to read together.

    {% code lineNumbers="true" %}
    ```cpp
    void RLock() {
      std::unique_lock<mutex_t> latch(mutex_);
      while (writer_entered_ || reader_count_ == MAX_READERS) {
        reader_.wait(latch);
      }
      reader_count_++;
    }
    ```
    {% endcode %}
4.  The `RUnlock` function

    1. Decrement the reader count.
    2. If a writer is waiting and no readers remain, notify one waiting writer.
    3. If the reader count was at the maximum before decrementing, notify another reader because one reader slot is now available.

    {% code lineNumbers="true" %}
    ```cpp
    void RUnlock() {
      std::lock_guard<mutex_t> guard(mutex_);
      reader_count_--;
      if (writer_entered_) {
        if (reader_count_ == 0) {
          writer_.notify_one();
        }
      } else {
        if (reader_count_ == MAX_READERS - 1) {
          reader_.notify_one();
        }
      }
    }
    ```
    {% endcode %}

## 7. Concurrent Index Implementation

### 1. Implementing `FindLeafPageRW`

#### 1.1 Overall idea

For concurrency control, this implementation uses the simplest latch-crabbing method described above. While finding the leaf page, it acquires latches step by step from the root to the leaf and checks whether earlier latches can be released. Because insertion and deletion both need to find a leaf page first, the previous lock-free version of `FindLeafPage` is not suitable under concurrency. Therefore, a new function with **incremental latching and incremental release** is needed.

1. The overall idea is almost the same as the earlier `FindLeafPage`, with several extra checks.
2. For read operations, acquire the latch directly and release the latch on the previous level.
3. For write operations, check whether the page is safe before releasing earlier latches.

{% code lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
Page *BPLUSTREE_TYPE::FindLeafPageRW(const KeyType &key, bool left_most, enum OpType op, Transaction *transaction) {
  Page *page = buffer_pool_manager_->FetchPage(root_page_id_);  // now root page is pin
  BPlusTreePage *node = reinterpret_cast<BPlusTreePage *>(page->GetData());
  while (!node->IsLeafPage()) {
    if (op == OpType::Read) {
      page->RLatch();
      UnlatchAndUnpin(op,transaction);
    } else {
      // else is write op
      page->WLatch();
      if (IsSafe(node, op)) {
        UnlatchAndUnpin(op, transaction);
      }
    }
    transaction->AddIntoPageSet(page);
    InternalPage *internal_node = reinterpret_cast<InternalPage *>(node);
    page_id_t next_page_id = left_most ? internal_node->ValueAt(0) : internal_node->Lookup(key, comparator_);
    Page *next_page = buffer_pool_manager_->FetchPage(next_page_id);  // next_level_page pinned
    BPlusTreePage *next_node = reinterpret_cast<BPlusTreePage *>(next_page);
    page = next_page;
    node = next_node;
  }
  return page;
}
```
{% endcode %}

#### 1.2 Function for checking whether a page is safe [#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#12-%E5%88%A4%E6%96%AD%E6%98%AF%E5%90%A6%E5%AE%89%E5%85%A8%E7%9A%84%E5%87%BD%E6%95%B0)

1. For insertion, the current node is safe if adding one entry will not cause a split.
2. For deletion, the current node is safe if removing one entry will not cause redistribution or merge.
3. The root needs special handling. If the root is a leaf page, it is safe. Otherwise, the root's size must be greater than `2`, because if it is exactly `2`, removing one entry leaves only one pointer and no valid separator key, which is unsafe.

{% code lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
bool BPLUSTREE_TYPE::IsSafe(N *node, enum OpType op) {
  // insert
  if (op == OpType::Insert) {
    return node->GetSize() < maxSize(node);
  }

  // remove
  if (node->IsRootPage()) {
    // If root is a leaf node, no constraint
    // If root is an internal node, it must have at least two pointers;
    if (node->IsLeafPage()) {
      return true;
    }

    return node->GetSize() > 2;
  }

  return node->GetSize() > minSize(node);
}
```
{% endcode %}

#### 1.3 Combine unlatch and unpin operations [#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#13-%E6%8A%8A%E9%87%8A%E6%94%BE%E9%94%81%E5%92%8Cunpin%E6%93%8D%E4%BD%9C%E5%90%88%E5%B9%B6)

Both operations apply to pages. Instead of repeating several lines, combine them in one helper function.

> `transaction->GetPageSet()` is the set of pages that were previously visited.

{% code lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::UnlatchAndUnpin(enum OpType op,Transaction *transaction) const {
  if (transaction == nullptr) {
    return;
  }

  auto pages = transaction->GetPageSet();
  for (auto page : *pages) {
    page_id_t page_id = page->GetPageId();

    if (op == OpType::Read) {
      page->RUnlatch();
      buffer_pool_manager_->UnpinPage(page_id, false);
    } else {
      page->WUnlatch();
      buffer_pool_manager_->UnpinPage(page_id, true);
    }
  }

  pages->clear();
}
```
{% endcode %}

### 2. Supporting Concurrent Reads and Writes

> This only requires small changes on top of the non-concurrent versions from the earlier notes.

#### 2.1 Support concurrent reads

#### 2.2 Support concurrent writes

> Both insertion and deletion need to be supported as write operations.

#### 1. Insertion&#x20;

1.  According to the project hint, first acquire the lock for the root node. My understanding is that this prevents the following case:

    > If this interpretation is wrong, corrections and discussion are welcome.
    >
    > consider the following case
    >
    > txn A read "A" from tree but not hold mutex (if "A" not in the tree)
    >
    > before A crab page0 latch , txnB crab page0 and insert "A" into the tree then unlatch page0
    >
    > txnA crab page0 get false result
2. Replace the previous `FindLeaf` function with `FindLeafRW`.
3. Replace the earlier simple `unpin` operation with `UnLatchAndPin`.

The full code is not repeated here; it only needs small changes based on the previous insertion implementation.

#### 2. Deletion

1. For deletion, first apply the same treatment to `remove` as for insertion.
2. In the core function `CoalesceOrRedistribute`, acquire a write latch before modifying a sibling page and release the latch afterward.
