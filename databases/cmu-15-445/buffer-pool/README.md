---
description: 'PROJECT #1 - BUFFER POOL'
---

# Buffer Pool

> Implement the `buffer pool` for storage management.

* Check the functions that need to be implemented in `lru_replace.h` and `buffer_pool_manager.h`.
* Check the member variables and functions of the `Page` and `DiskManager` classes.

### TASK #1 - LRU REPLACEMENT POLICY

> Implement the LRU policy for page replacement: evict the least recently used page first.

#### Idea

* Use a `std::list<frame_id_t> lru` doubly linked list to store pages that can be evicted.
* Use `std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> mp` to accelerate `frame` lookup and deletion.
* Use `std::mutex latch_;` to support multithreading.

When using `std::lock_guard<std::mutex> guard(latch_);`, the constructor of `guard` automatically calls `latch_.lock()`, and the destructor of `guard` automatically calls `latch_.unlock()` to release the lock.

#### Victim

* Return `false` when `Size()` is 0.
* Convention in this note: the head of the linked list represents the frame that most recently became replaceable, and the tail represents the least recently used frame.
* `Victim` takes the tail element of `lru`, stores it in `frame_id`, removes the tail element, and removes `frame_id` from `mp`.

#### Pin

* When `frame_id` exists in `mp`, remove that `frame_id` from both the `lru` linked list and the `mp` dictionary.
* Because `mp` already stores the iterator for `frame_id`, removing it from `lru` has time complexity `O(1)`.

#### Unpin

* If `frame_id` already exists in `mp`, return directly.
* Otherwise, add `frame_id` to the head of `lru`, then store this `frame_id` and its iterator, `lru.begin()`, in `mp`.

***

### TASK #2 - BUFFER POOL MANAGER

> When implementing `buffer_pool_manager`, first inspect the member variables of the `Page` object. The implementation logic of each function has already been given in detail.

#### Notes

* `page_id` refers to a page on disk.
* `page_table_` maps a `page_id` to the corresponding `frame_id`.
* `pages_` points to an array of `Page` objects, and `frame_id` is used to locate a specific `Page` object.
* In `UnpinPageImpl`, if `page_id` is not in `page_table_`, return `true` rather than `false`. Also, set the `Page`'s `is_dirty_` flag only when `is_dirty` is `true`.
* In `DeletePageImpl(page_id_t page_id)`, besides removing the page corresponding to `page_id` from `page_table_`, call `Pin` to remove that page from `replacer_`.

***

### Tests

```bash
make lru_replacer_test
./test/lru_replacer_test
```

```bash
make buffer_pool_manager_test
./test/buffer_pool_manager_test
```

### Code format checks

```bash
make format
make check-lint
make check-clang-tidy
```

### Packaging

* Package the submission:

```
zip project1-submission.zip src/include/buffer/lru_replacer.h src/buffer/lru_replacer.cpp src/include/buffer/buffer_pool_manager.h src/buffer/buffer_pool_manager.cpp
```

* Then go to [**Gradescope**](https://www.gradescope.com/) and submit the code.
