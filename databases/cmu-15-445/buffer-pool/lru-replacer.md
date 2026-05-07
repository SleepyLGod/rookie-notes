# LRU Replacer

# 😄 Expand

### 1. Task1 LRU REPLACEMENT POLICY[#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#1-task1-lru-replacement-policy) <a href="#1-task1-lru-replacement-policy" id="1-task1-lru-replacement-policy"></a>

#### 0. 任务描述[#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#0-%E4%BB%BB%E5%8A%A1%E6%8F%8F%E8%BF%B0) <a href="#0-ren-wu-miao-shu" id="0-ren-wu-miao-shu"></a>

这个任务要求我们实现在课堂上所描述的LRU算法最近最少使用算法。

你需要实现下面这些函数。请确保他们都是线程安全的。

* `Victim(T*)` : Remove the object that was accessed the least recently compared to all the elements being tracked by the `Replacer`, store its contents in the output parameter and return `True`. If the `Replacer` is empty return `False`.
* `Pin(T)` : This method should be called after a page is pinned to a frame in the `BufferPoolManager`. It should remove the frame containing the pinned page from the `LRUReplacer`.
* `Unpin(T)` : This method should be called when the `pin_count` of a page becomes 0. This method should add the frame containing the unpinned page to the `LRUReplacer`.
* `Size()` : This method returns the number of frames that are currently in the `LRUReplacer`.

关于`Lock`和`Lathes`的区别请看下文。

{% embed url="https://stackoverflow.com/questions/3111403/what-is-the-difference-between-a-lock-and-a-latch-in-the-context-of-concurrent-a/42464336#42464336" %}

#### 1. 实现[#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#1-%E5%AE%9E%E7%8E%B0) <a href="#1-shi-xian" id="1-shi-xian"></a>

其实这个任务还是蛮简单的。你只需要清楚什么是最近最少使用算法即可。

> LRU 算法的设计原则是：如果一个数据在最近一段时间没有被访问到，那么在将来它被访问的可能性也很小。也就是说，当限定的空间已存满数据时，应当把最久没有被访问到的数据淘汰。

这个题我熟啊。`leetcode`上有原题。而且要求在o(1)的时间复杂度实现这一任务。

{% embed url="https://leetcode-cn.com/problems/lru-cache/" %}

为了实现在O(1)时间内进行查找。因此我们可以用一个hash表。而且我们要记录一个时间戳来完成记录最近最少使用的块是谁。这里我们可以用`list`来实现。

如果我们访问了链表中的一个元素。就把这个元素放在链表头部。这样放在链表尾部的元素一定就是最近最少使用的元素。

为了让插入和删除均为O(1)我们可以用链表来实现。

这里对于`pin`和`unpin`操作实际上对于了`task2`。我们为什么需要`pin`。书上给了我们答案。下面我们也进行了分析

**1.1 数据结构设计**[**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#11-%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%AE%BE%E8%AE%A1)

```cpp
  std::mutex latch;  // thread safety
  int capacity;      // max number of pages LRUReplacer can handle
  std::list<frame_id_t> lru_list;
	std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> lruMap;
```

这里我们用了链表 + hash表。主要是为了删除和插入均为0(1)的时间复杂度。引入hash表就是可以根据`frame_id`快速找到其在`list`中对应的位置。否则的话你需要遍历链表这就不是o(1)了

**1.2 Victim 函数实现**[**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#12-victim-%E5%87%BD%E6%95%B0%E5%AE%9E%E7%8E%B0)

> 注意这里必须要加锁，以防止并发错误。

1. 如果没有可以牺牲的页直接返回false
2. 如果有的话选择在链表尾部的页。remove它即可。这里的删除涉及链表和hash表两个数据结构的删除

```cpp
bool LRUReplacer::Victim(frame_id_t *frame_id) {
  // 选择一个牺牲frame
  latch.lock();
  if (lruMap.empty()) {
    latch.unlock();
    return false;
  }

  // 选择列表尾部 也就是最少使用的frame
  frame_id_t lru_frame = lru_list.back();
  lruMap.erase(lru_frame);
  // 列表删除
  lru_list.pop_back();
  *frame_id = lru_frame;
  latch.unlock();
  return true;
}
```

**1.3 pin 函数实现**[**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#13-pin-%E5%87%BD%E6%95%B0%E5%AE%9E%E7%8E%B0)

> 注意这里必须要加锁，以防止并发错误。

1. pin函数表示这个frame被某个进程引用了
2. 被引用的frame不能成为LRU算法的牺牲目标，所以这里把它从我们的数据结构中删除

```cpp
void LRUReplacer::Pin(frame_id_t frame_id) {
  // 被引用的frame 不能出现在lru list中
  latch.lock();

  if (lruMap.count(frame_id) != 0) {
    lru_list.erase(lruMap[frame_id]);
    lruMap.erase(frame_id);
  }

  latch.unlock();
}
```

#### **1.4 unpin 函数实现**[#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#14-unpin-%E5%87%BD%E6%95%B0%E5%AE%9E%E7%8E%B0) <a href="#14unpin-han-shu-shi-xian" id="14unpin-han-shu-shi-xian"></a>

> 注意这里必须要加锁，以防止并发错误。

1. 先看一下这个页是否在可替换链表中
2. 如果它不存在的话。则需要看一下当前链表是否还有空闲位置。如果有的话则直接加入
3. 如果没有则需要移除链表尾部的节点知道有空余位置

```cpp
void LRUReplacer::Unpin(frame_id_t frame_id) {
  // 加入lru list中
  std::lock_guard<std::mutex> guard(latch);
  if (lruMap.count(frame_id) != 0) {
    return;
  }
  // if list size >= capacity
  // while {delete front}
  while (lru_list.size() >= capacity) {
     frame_id_t need_del = lru_list.front();
      lru_list.pop_front();
      lruMap.erase(need_del);
  }
  // insert
  lru_list.push_front(frame_id);
  lruMap[frame_id] = lru_list.begin();
}

```

#### 2. 测试[#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#2-%E6%B5%8B%E8%AF%95) <a href="#2-ce-shi" id="2-ce-shi"></a>

执行下面的语句即可

```bash
 cd build
 make lru_replacer_test
 ./test/lru_replacer_tes
可以发现成功通过
```

<figure><img src="../../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>
