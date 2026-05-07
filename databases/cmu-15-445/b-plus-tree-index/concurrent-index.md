# Concurrent Index

## 5. 并发机制的实现

### 0. 首先复习一下读写🔒机制&#x20;

1. 读操作是可以多个进程之间共享latch的而写操作则必须互斥
2. 加入`MaxReader`数就是为了防止等待的⌛️写进程饥饿

### **首先来看如果没有🔒机制多线程会发生什么问题**

1. 线程T1想要删除44。
2. 线程T2 想要查找41

![image-20210126184533688](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201246511-892266003.png)

1. 假设T2在执行到D位置的时候又切换到线程T1
2. 这个时候T1进行重新分配，会把41借到I结点上
3. T1执行完成切换回T2这时候T2再去原来的执行寻找41就会找不到

![image-20210126184727498](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201510577-283160563.png)

就会出现下面的情况。❓

![image-20210126184901306](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201708545-1250779663.png)

### **由此我们需要读写🔒的存在**

1. #### 对于find操作

> 由于我们是只读操作，所以我们到下一个结点的时候就可以释放上一个结点的Latch

![image-20210126185917549](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201736266-1524422160.png)

剩下的操作都是一样的

### 对于`delete`则不一样

> 因为我们需要写操作

这里我们不能释放结点A的Latch。因为我们的删除操作可能会合并根节点。

![image-20210126190112632](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201804944-2073658211.png)

到D的时候。我们会发现D中的38删除之后不需要进行合并，所以对于A和B的写Write是可以安全释放了

![image-20210126190229333](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201823967-455255873.png)

### 对于`Insert`操作

这里我们就可以安全的释放掉A的锁。因为B中还有空位，我们插入是不会对A造成影响的

![image-20210126190452937](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201845743-1649741202.png)

当我们执行到D这里发现D中已经满了。所以此时我们不会释放B的锁，因为我们会对B进行写操作

![image-20210126190613339](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126201922260-1593188438.png)

上面的算法虽然是正确的但是有瓶颈问题。由于只有一个线程可以获得写Latch。而插入和删除的时候都需要对头结点加写Latch。所以多线程在有许多个插入或者删除操作的时候，性能就会大打折扣

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202003618-880373652.png)

### **这里要引入乐观🔒**

> 乐观的假设大部分操作是不需要进行合并和分裂的。因此在我们向下的时候都是读Latch而不是写Latch。只有在叶子结点才是write Latch

1. 从上到下都是读Latch。而且逐步释放
2. 到叶子结点需要修改的时候才为写Latch。这个删除是安全的所以直接结束

![image-20210126192408392](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202039219-1666824693.png)

当我们到最后一步发现不安全的时候。则需要像上面我们没有引入乐观🔒的时候一样。重新执行一遍

![image-20210126192548748](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202108649-568164667.png)

### **B-Link Tree简介**

> 延迟更新父结点

这里用一个🌟来标记这里需要被更新但是还没有执行

![image-20210126195848104](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202130834-1687270714.png)

这个时候我们执行其他操作也是正确的比如查找31

![image-20210126200003320](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202156088-1710986034.png)

这里我们执行`insert 33`

当执行到结点C的时候。因为这个时候有另一个线程持有了write Latch。所以这个时候🌟操作要执行。随后在插入33

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210127092842951-664251463.png)

**最后一点补充关于扫描操作的**

1. 线程1在C结点上持有write Latch
2. 线程2已经扫描完了结点B想要获得结点C的read Latch

这时候会发生问题，因为线程2无法拿到read Latch

**这里有几种解决方法**

1. 可以等到T1的写操作完成
2. 可以重新执行T2
3. 可以直接让线程T2停止抢得这个Latch。

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126202324217-992759171.png)

注意这里的`Latch`和`Lock`并不一样

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/754297-20160131225332443-857830570.jpg)

## 6. 辅助函数分析

### 1. 辅助函数`UnlockUnpinPages`的实现

1. 如果是读操作则释放read锁
2. 否则释放write锁

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

四个自带的解锁和上锁操作

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

这里的rwlatch是自己实现的读写锁类下面来探究一下这个类

> 由于c++ 并发编程我现在还不太会。。。所以就简单看一下啦后面学完并发编程再补充

1.  `WLock`函数

    1. 首先获取一个锁
    2. 用一个记号`writer_entered`表示是否有写操作
    3. 如果之前已经有了现在的操作就需要等(这个线程处于阻塞状态)
    4. 当前如果有其他线程执行读操作。则仍需要阻塞(别人读的时候你不能写)

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
2.  `WunLock`函数

    1. 写标记置为false
    2. 然后通知所有的线程

    {% code lineNumbers="true" %}
    ```cpp
    void WUnlock() {
      std::lock_guard<mutex_t> guard(mutex_);
      writer_entered_ = false;
      reader_.notify_all();
    }
    ```
    {% endcode %}
3.  `RLock`函数

    1. 如果当前有人在写或者已经有最多的人读了则阻塞
    2. 否则只需要让读的计数++

    因为是允许多个线程一起读这样并不会出错

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
4.  `RUnLatch`函数

    1. 计数--
    2. 如果当前有人在写并且无人读的话需要通知所有其他线程
    3. 如果在计数--之前达到了最大读数，释放这个锁之后需要通知其他线程，现在又可以读了。

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

## 7. 并发索引实现

### 1. FindLeafPageRW的实现

#### 1. 1 整体思路

对于并发控制的实现，采用最简单的latch crabing方法实现，也就是上面讲的那种方法， 这种方法需要在找叶子结点的时候，从根节点到叶子结点的过程需要逐步加锁，然后检测是否能够释放。由于我们的插入和删除操作都需要先找到叶子结点，所以之前使用的无锁版本的`FindLeafPage`函数在并发条件下就并不适用了。因此这里需要实现一个**逐步加锁 + 逐步释放**的新函数

1. 整体思路和之前的findLeafPage几乎一样，只是多了几次判断
2. 如果是读操作，那则直接加锁，然后对上一层释放锁
3. 如果是写操作，释放锁之前则要判断一下是否安全。

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

#### 1.2 判断是否安全的函数[#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#12-%E5%88%A4%E6%96%AD%E6%98%AF%E5%90%A6%E5%AE%89%E5%85%A8%E7%9A%84%E5%87%BD%E6%95%B0)

1. 如果是插入操作，则只要当前node的size处于安全状态即 + 1 之后不会产生分裂，则为安全
2. 如果是删除状态。则只要当前node的size - 1 之后不会重分配或者合并，则为安全
3. 对于根节点需要进行特殊判断，如果这个根节点是叶子结点则为安全（这种情况随便删）。否则根节点的大小必须大于2(因为如果等于2 ，减去1之后还是1。则是一个没有有效key值的结点，不安全)

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

#### 1.3 把释放锁和unpin操作合并[#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#13-%E6%8A%8A%E9%87%8A%E6%94%BE%E9%94%81%E5%92%8Cunpin%E6%93%8D%E4%BD%9C%E5%90%88%E5%B9%B6)

这两个操都要对page做，与其多写几行不如写个函数给他合并在一起做。

> transaction->GetPageSet(); 就是之前访问过的page集合

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

### 2. 支持并发的读写操作

> 其实只需要之前博客1、2的非并发版本上做一些小小的改动

#### 2.1 支持并发读

#### 2.2 支持并发写

> 这里要支持插入和删除两种写操作

#### 1. 插入&#x20;

1.  根据实验提示，首先需要获取对于根节点的锁。我个人认为是为了防止下面这种情况发生

    > 如果理解的有问题，欢迎大家指出，互相讨论
    >
    > consider the following case
    >
    > txn A read "A" from tree but not hold mutex (if "A" not in the tree)
    >
    > before A crab page0 latch , txnB crab page0 and insert "A" into the tree then unlatch page0
    >
    > txnA crab page0 get false result
2. 用`FindLeafRW`替换之前的`FindLeaf`函数即可
3. 用`UnLatchAndPin`替换之前简单的unpin操作。

完整代码就不贴了，在之前的insert上改一下就行了

#### 2. 删除

1. 对于删除首先要在remove上做和insert一样的处理
2. 在核心函数`CoalesceOrRedistribute`中对兄弟结点做修改之前，先加写锁结束之后释放写锁就ok
