---
description: 'PROJECT #1 - BUFFER POOL'
---

# 😉 Buffer Pool

> 实现存储管理的 `buffer pool`

* 查看 `lru_replace.h` 和 `buffer_pool_manager.h` 中需要实现的函数
* 查看 `Page`，`DiskManager` 类的一些成员变量和函数

### TASK #1 - LRU REPLACEMENT POLICY

> 实现页面替换的 LRU 策略，即优先替换最近最少使用的页面

#### 思路

* 使用 `std::list<frame_id_t> lru` 双链表的数据结构来添加可以被淘汰的页面
* 使用 `std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> mp` 来加速 `frame` 的查找和删除
* 使用 `std::mutex latch_;` 来支持多线程

使用 `std::lock_guard<std::mutex> guard(latch_);` 语句，在构造 `guard` 时会自动调用 `latch_.lock()` 加锁，并且在 `guard` 析构的时候自动调用 `latch_.unlock()` 释放锁。

#### Victim

* 当 `Size()` 为 0 时返回 `false`
* 本笔记约定：链表头部表示最近刚变为可替换的 frame，链表尾部表示最久未使用的 frame。
* `Victim` 取 `lru` 的尾部元素给 `frame_id`，删除尾部元素，并且在 `mp` 中删除 `frame_id`。

#### Pin

* 当 `frame_id` 在 `mp` 中时，我们需要在 `lru` 链表和 `mp` 字典中删除 `frame_id` 这个元素
* 因为 `mp` 中已经保存了 `frame_id` 的迭代器，因此从 `lru` 中删除它的时间复杂度为 O(1)O(1)

#### Unpin

* 如果 `frame_id` 已经在 `mp` 中了，那么直接返回
* 否则的话将 `frame_id` 添加到 `lru` 的头部，然后在 `mp` 中保存这个 `frame_id` 和它的迭代器（`lru.begin()`）

***

### TASK #2 - BUFFER POOL MANAGER

> 在实现 `buffer_pool_manager` 时，应该看一下 `Page` 对象的一些成员变量；各个函数如何实现已经详细给出了。

#### 注意

* `page_id` 表示的是磁盘上的页
* `page_table_` 可以根据 `page_id` 找到对应的 `frame_id`
* `pages_` 指向了一个 `Page` 数组，我们会用 `frame_id` 来找到某一个 `Page` 对象
* 在 `UnpinPageImpl` 函数中，如果 `page_id` 不在 `page_table_` 中，我们应该返回 `true`，而不是 `false`；还有就是只有当 `is_dirty` 为 `true` 时，我们才设置 `Page` 的 `is_dirty_`
* 在 `DeletePageImpl(page_id_t page_id)` 中，除了把 `page_id` 对应的页从 `page_table_` 移除，还需要调用 `Pin` 将该页从 `replacer_` 中移除

***

### 测试

```bash
make lru_replacer_test
./test/lru_replacer_test
```

```bash
make buffer_pool_manager_test
./test/buffer_pool_manager_test
```

### 代码格式验证

```bash
make format
make check-lint
make check-clang-tidy
```

### 打包

* 打包

```
zip project1-submission.zip src/include/buffer/lru_replacer.h src/buffer/lru_replacer.cpp src/include/buffer/buffer_pool_manager.h src/buffer/buffer_pool_manager.cpp
```

* 然后前往 [**https://www.gradescope.com**](https://www.gradescope.com/) 提交代码
