# LRU Replacer

# Expand

### 1. Task 1: LRU Replacement Policy [#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#1-task1-lru-replacement-policy) <a href="#1-task1-lru-replacement-policy" id="1-task1-lru-replacement-policy"></a>

#### 0. Task description [#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#0-%E4%BB%BB%E5%8A%A1%E6%8F%8F%E8%BF%B0) <a href="#0-ren-wu-miao-shu" id="0-ren-wu-miao-shu"></a>

This task requires us to implement the LRU algorithm, namely the least recently used replacement policy described in class.

You need to implement the following functions. Make sure they are all thread-safe.

* `Victim(T*)` : Remove the object that was accessed the least recently compared to all the elements being tracked by the `Replacer`, store its contents in the output parameter and return `True`. If the `Replacer` is empty return `False`.
* `Pin(T)` : This method should be called after a page is pinned to a frame in the `BufferPoolManager`. It should remove the frame containing the pinned page from the `LRUReplacer`.
* `Unpin(T)` : This method should be called when the `pin_count` of a page becomes 0. This method should add the frame containing the unpinned page to the `LRUReplacer`.
* `Size()` : This method returns the number of frames that are currently in the `LRUReplacer`.

For the difference between a `Lock` and a `Latch`, see the following discussion.

{% embed url="https://stackoverflow.com/questions/3111403/what-is-the-difference-between-a-lock-and-a-latch-in-the-context-of-concurrent-a/42464336#42464336" %}

#### 1. Implementation [#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#1-%E5%AE%9E%E7%8E%B0) <a href="#1-shi-xian" id="1-shi-xian"></a>

This task is fairly simple as long as the least recently used algorithm is clear.

> The design principle of LRU is: if a piece of data has not been accessed for a recent period of time, it is unlikely to be accessed in the near future. Therefore, when the bounded space is full, the data that has not been accessed for the longest time should be evicted.

This problem is familiar because LeetCode has a direct version of it, and it asks for an `O(1)` implementation.

{% embed url="https://leetcode-cn.com/problems/lru-cache/" %}

To perform lookup in `O(1)` time, we can use a hash table. We also need to record recency so that we know which block is least recently used. A `list` can be used for that.

If we access an element in the linked list, move that element to the head of the list. Then the element at the tail of the list is always the least recently used element.

To make both insertion and deletion `O(1)`, use a linked list.

The `pin` and `unpin` operations are actually related to Task 2. Why do we need `pin`? The textbook gives the answer, and the following analysis also explains it.

**1.1 Data structure design** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#11-%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%AE%BE%E8%AE%A1)

```cpp
  std::mutex latch;  // thread safety
  int capacity;      // max number of pages LRUReplacer can handle
  std::list<frame_id_t> lru_list;
	std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> lruMap;
```

Here we use a linked list plus a hash table, mainly to make deletion and insertion both `O(1)`. The hash table lets us quickly find the corresponding position in the `list` by `frame_id`. Otherwise, we would need to traverse the linked list, which would no longer be `O(1)`.

**1.2 Implementation of the Victim function** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#12-victim-%E5%87%BD%E6%95%B0%E5%AE%9E%E7%8E%B0)

> A latch must be acquired here to prevent concurrency bugs.

1. If there is no page that can be evicted, return `false` directly.
2. Otherwise, choose the page at the tail of the linked list and remove it. This deletion involves both the linked list and the hash table.

```cpp
bool LRUReplacer::Victim(frame_id_t *frame_id) {
  // Choose a victim frame.
  latch.lock();
  if (lruMap.empty()) {
    latch.unlock();
    return false;
  }

  // Choose the tail of the list, which is the least recently used frame.
  frame_id_t lru_frame = lru_list.back();
  lruMap.erase(lru_frame);
  // Delete it from the list.
  lru_list.pop_back();
  *frame_id = lru_frame;
  latch.unlock();
  return true;
}
```

**1.3 Implementation of the pin function** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#13-pin-%E5%87%BD%E6%95%B0%E5%AE%9E%E7%8E%B0)

> A latch must be acquired here to prevent concurrency bugs.

1. The `pin` function means this frame is being referenced by some process.
2. A referenced frame cannot become a victim of the LRU algorithm, so remove it from our data structures.

```cpp
void LRUReplacer::Pin(frame_id_t frame_id) {
  // A referenced frame must not appear in the LRU list.
  latch.lock();

  if (lruMap.count(frame_id) != 0) {
    lru_list.erase(lruMap[frame_id]);
    lruMap.erase(frame_id);
  }

  latch.unlock();
}
```

#### **1.4 Implementation of the unpin function** [#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#14-unpin-%E5%87%BD%E6%95%B0%E5%AE%9E%E7%8E%B0) <a href="#14unpin-han-shu-shi-xian" id="14unpin-han-shu-shi-xian"></a>

> A latch must be acquired here to prevent concurrency bugs.

1. First check whether this page is already in the replaceable list.
2. If it is not, check whether the current list still has free capacity. If it does, add the page directly.
3. If there is no free capacity, remove nodes from the tail until there is space.

```cpp
void LRUReplacer::Unpin(frame_id_t frame_id) {
  // Add it into the LRU list.
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

#### 2. Testing [#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#2-%E6%B5%8B%E8%AF%95) <a href="#2-ce-shi" id="2-ce-shi"></a>

Run the following commands:

```bash
 cd build
 make lru_replacer_test
 ./test/lru_replacer_test
# The test passes successfully.
```

<figure><img src="../../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>
