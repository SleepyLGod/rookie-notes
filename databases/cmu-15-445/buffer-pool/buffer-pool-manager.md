# Buffer Pool Manager

### Task2 BUFFER POOL MANAGER[#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#task2-buffer-pool-manager) <a href="#task2-buffer-pool-manager" id="task2-buffer-pool-manager"></a>

#### 0. Task Description [#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#0-%E4%BB%BB%E5%8A%A1%E6%8F%8F%E8%BF%B0-1) <a href="#0-task-description" id="0-task-description"></a>

Next, you need to implement the buffer pool manager (`BufferPoolManager`) in the system. `BufferPoolManager` is responsible for fetching database pages from `DiskManager` and storing them in memory. `BufferPoolManager` can also write dirty pages back to disk when requested, or when it needs to evict a page to make room for a new page. To ensure that your implementation works correctly with the rest of the system, some functions are already provided. You also do not need to implement the actual disk read/write logic, which is handled by `DiskManager` in this project.

All in-memory pages in the system are represented by `Page` objects. `BufferPoolManager` does not need to understand the contents of those pages. However, as a system implementer, it is important to understand that a `Page` object is only a memory container inside the buffer pool; it is not permanently tied to one logical page. Each `Page` object contains a memory region where `DiskManager` can copy the contents of a physical page read from disk. `BufferPoolManager` reuses the same `Page` objects as pages move between memory and disk. Therefore, over the lifetime of the system, the same `Page` object may contain different physical pages. The `page_id` of a `Page` object tracks which physical page it currently contains. If a `Page` object does not contain a physical page, its `page_id` must be set to `INVALID_PAGE_ID`.

Each `Page` object also maintains a counter that records how many users have pinned the page. `BufferPoolManager` must not evict a pinned page. Each `Page` object also tracks a dirty flag. Your job is to determine whether a page has been modified before it is unpinned; if so, mark the page dirty. Before reusing a dirty `Page` object, `BufferPoolManager` must write its contents back to disk.

The `BufferPoolManager` implementation uses the `LRUReplacer` class implemented in the previous step of this assignment. It uses `LRUReplacer` to track page-object access so that, when a frame must be freed to make room for a physical page copied from disk, the system can decide which page object to evict.

You need to implement the following functions in `src/buffer/buffer_pool_manager.cpp`:

* `FetchPageImpl(page_id)`
* `NewPageImpl(page_id)`
* `UnpinPageImpl(page_id, is_dirty)`
* `FlushPageImpl(page_id)`
* `DeletePageImpl(page_id)`
* `FlushAllPagesImpl()`

#### 1. Analysis [#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#1-%E5%88%86%E6%9E%90) <a href="#1-analysis" id="1-analysis"></a>

**1.1 Why pinning is needed** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#11-%E4%B8%BA%E4%BB%80%E4%B9%88%E9%9C%80%E8%A6%81pin)

The basic idea is shown in the figure below.

<figure><img src="../../../.gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>

Consider this case. A block is placed into the buffer, and a process reads the block contents from buffer memory. While that block is being read, if another concurrent process evicts the block and replaces it with a different block, the reader of the old block will see incorrect data. If the block is being written while it is evicted, the writer may corrupt the contents of the replacement block.

Therefore, before a process reads data from a buffered block, it must ensure that the block cannot be evicted. To do that, the process performs a `pin` operation on the block. The buffer manager never evicts pinned blocks, meaning blocks whose pin count is not zero. When the process finishes reading or writing the data, it should perform an `unpin` operation so the block may be evicted later if needed.

Therefore, we need a `pin_counter` to record the number of pins. Conceptually, this is the same idea as reference counting.

**1.2 How to manage and access pages** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#12-%E5%A6%82%E4%BD%95%E7%AE%A1%E7%90%86%E9%A1%B5%E5%92%8C%E8%AE%BF%E9%97%AE%E9%A1%B5)

In one sentence: base address plus offset.

> `page` as the base address plus `frame_id` as the offset is essentially array addressing.
>
> At the same time, the DBMS maintains a page table that records each page's in-memory location and metadata such as whether it has been written (`Dirty Flag`) and whether it is referenced or pinned (`Pin/Reference Counter`), as shown below:

Here, a hash table implements `page_table`, mapping `page_id` to `frame_id`.

<figure><img src="../../../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

#### 2. Implementation [#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#2-%E5%AE%9E%E7%8E%B0) <a href="#2-implementation" id="2-implementation"></a>

**2.1 The `find_replace()` function** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#21-find\_replace%E5%87%BD%E6%95%B0)

1. If the free list is not empty, no replacement algorithm is needed. Return a free frame directly. This is the case where the buffer pool is not full.
2. If the free list is empty, the buffer pool is full, so the LRU replacement algorithm must be used.

**Process for finding a replacement frame**

1. Call the previously implemented `Victim` function to obtain the victim frame's `frame_id`.
2. Find the corresponding victim page in `pages_`. If the page is dirty, write it back to disk and reset the pin count.
3. Then remove the corresponding mapping from `page_table`: `[page_id -> frame_id]`.

> Be careful not to reverse steps 2 and 3; otherwise you cannot find the corresponding victim page.

```
bool BufferPoolManager::find_replace(frame_id_t *frame_id) {
  // if free_list not empty then we don't need replace page
  // return directly
  if (!free_list_.empty()) {
    *frame_id = free_list_.front();
    free_list_.pop_front();
    return true;
  }
  // else we need to find a replace page
  if (replacer_->Victim(frame_id)) {
    // Remove entry from page_table
    int replace_frame_id = -1;
    for (const auto &p : page_table_) {
      page_id_t pid = p.first;
      frame_id_t fid = p.second;
      if (fid == *frame_id) {
        replace_frame_id = pid;
        break;
      }
    }
    if (replace_frame_id != -1) {
      Page *replace_page = &pages_[*frame_id];

      // If dirty, flush to disk
      if (replace_page->is_dirty_) {
        char *data = pages_[page_table_[replace_page->page_id_]].data_;
        disk_manager_->WritePage(replace_page->page_id_, data);
        replace_page->pin_count_ = 0;  // Reset pin_count
      }
      page_table_.erase(replace_page->page_id_);
    }

    return true;
  }

  return false;
}
```

**2.2 Implementing `FetchPageImpl`** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#22-fetchpageimpl-%E5%AE%9E%E7%8E%B0)

```
Page *BufferPoolManager::FetchPageImpl(page_id_t page_id)
```

This function fetches a `page`. The logic can be analyzed in several cases:

1. If the page is already in the buffer pool, access it directly, increment its `pin_count`, and call `Pin` to notify the replacer.
2. Otherwise, call `find_replace` to obtain an available `frame_id`, regardless of whether it comes from the free list or from replacement.
3. If no replacement frame is available, return `nullptr`.
4. Then create the new `page_table` mapping.

```
 latch_.lock();
  std::unordered_map<page_id_t, frame_id_t>::iterator it = page_table_.find(page_id);
  // 1.1 P exists
  if (it != page_table_.end()) {
    frame_id_t frame_id = it->second;
    Page *page = &pages_[frame_id];

    //
    page->pin_count_++;        // pin the page
    replacer_->Pin(frame_id);  // notify replacer

    latch_.unlock();
    return page;
  }
  // 1.2 P not exist
  frame_id_t replace_fid;
  if (!find_replace(&replace_fid)) {
    latch_.unlock();
    return nullptr;
  }
  Page *replacePage = &pages_[replace_fid];
  // 2. write it back to the disk
  if (replacePage->IsDirty()) {
    disk_manager_->WritePage(replacePage->page_id_, replacePage->data_);
  }
  // 3
  page_table_.erase(replacePage->page_id_);
  // create new map
  // page_id <----> replaceFrameID;
  page_table_[page_id] = replace_fid;
  // 4. update replacePage info
  Page *newPage = replacePage;
  disk_manager_->ReadPage(page_id, newPage->data_);
  newPage->page_id_ = page_id;
  newPage->pin_count_++;
  newPage->is_dirty_ = false;
  replacer_->Pin(replace_fid);
  latch_.unlock();

  return newPage;
```

**2.3 Implementing `UnpinPageImpl`** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#23-unpinpageimpl-%E5%AE%9E%E7%8E%B0)

```
bool BufferPoolManager::UnpinPageImpl(page_id_t page_id, bool is_dirty) 
```

The function signature is shown above.

This function is used when a process has finished operating on a page and needs to `unpin` it.

1. If the page's `pin_counter > 0`, decrement it directly.
2. If the page's `pin_counter == 0`, add it to the `LRUReplacer`. Since nobody references it, it can become a replacement candidate.

```
bool BufferPoolManager::UnpinPageImpl(page_id_t page_id, bool is_dirty) {
  latch_.lock();
  // 1. page_id is not in page_table
  auto iter = page_table_.find(page_id);
  if (iter == page_table_.end()) {
    latch_.unlock();
    return false;
  }
  // 2. find the page to unpin
  frame_id_t unpinned_Fid = iter->second;
  Page *unpinned_page = &pages_[unpinned_Fid];
  if (is_dirty) {
    unpinned_page->is_dirty_ = true;
  }
  // if the page's pin_count is already 0, return directly
  if (unpinned_page->pin_count_ == 0) {
    latch_.unlock();
    return false;
  }
  unpinned_page->pin_count_--;
  if (unpinned_page->GetPinCount() == 0) {
    replacer_->Unpin(unpinned_Fid);
  }
  latch_.unlock();
  return true;
}
```

**2.4 Implementing `FlushPageImpl`** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#24-flushpageimpl-%E5%AE%9E%E7%8E%B0)

```
bool BufferPoolManager::FlushPageImpl(page_id_t page_id)
```

This function writes a `page` to disk.

1. First find the page's location in the buffer pool.
2. Write it to disk.

```
  // Make sure you call DiskManager::WritePage!
  auto iter = page_table_.find(page_id);
  if (iter == page_table_.end() || page_id == INVALID_PAGE_ID) {
    latch_.unlock();
    return false;
  }

  frame_id_t flush_fid = iter->second;
  disk_manager_->WritePage(page_id, pages_[flush_fid].data_);
  
  return false;
```

**2.5 Implementing `NewPageImpl`** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#25-newpageimpl-%E5%AE%9E%E7%8E%B0)

```
Page *BufferPoolManager::NewPageImpl(page_id_t *page_id) 
```

Allocate a new page.

1. Use `find_replace` to find a suitable frame in the buffer pool and create the `page_id -> frame_id` mapping.
2. Update the new page's metadata.\
   Note that the newly created page should be written back to disk.

```
Page *BufferPoolManager::NewPageImpl(page_id_t *page_id) {
    latch_.lock();
  // 0.
  page_id_t new_page_id = disk_manager_->AllocatePage();
  // 1.
  bool is_all = true;
  for (int i = 0; i < static_cast<int>(pool_size_); i++) {
    if (pages_[i].pin_count_ == 0) {
      is_all = false;
      break;
    }
  }
  if (is_all) {
    latch_.unlock();
    return nullptr;
  }
  // 2.
  frame_id_t victim_fid;
  if (!find_replace(&victim_fid)) {
    latch_.unlock();
    return nullptr;
  }
  // 3.
  Page *victim_page = &pages_[victim_fid];
  victim_page->page_id_ = new_page_id;
  victim_page->pin_count_++;
  replacer_->Pin(victim_fid);
  page_table_[new_page_id] = victim_fid;
  victim_page->is_dirty_ = false;
  *page_id = new_page_id;
  // [attention]
  // if this not write to disk directly
  // maybe meet below case:
  // 1. NewPage
  // 2. unpin(false)
  // 3. operations on other pages cause this page to be removed from the buffer pool
  // 4. at that point, FetchPage cannot retrieve this page
  // Therefore, write it back to disk first
  disk_manager_->WritePage(victim_page->GetPageId(), victim_page->GetData());
  latch_.unlock();
  return victim_page;
}
```

**2.6 Implementing `DeletePageImpl`** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#26-deletepageimpl-%E5%AE%9E%E7%8E%B0)

```
bool BufferPoolManager::DeletePageImpl(page_id_t page_id)
```

This function removes a page from the buffer pool.

1. If the page is not in the buffer pool at all, return directly.
2. If the page's reference count is greater than 0 (`pin_counter > 0`), it cannot be deleted.
3. If the page has been modified, write it back to disk.
4. Otherwise, remove it normally by erasing it from the hash table.

```
bool BufferPoolManager::DeletePageImpl(page_id_t page_id) {
  // 0.   Make sure you call DiskManager::DeallocatePage!
  // 1.   Search the page table for the requested page (P).
  // 1.   If P does not exist, return true.
  // 2.   If P exists, but has a non-zero pin-count, return false. Someone is using the page.
  // 3.   Otherwise, P can be deleted. Remove P from the page table, reset its metadata and return it to the free list.
  latch_.lock();

  // 1.
  if (page_table_.find(page_id) == page_table_.end()) {
    latch_.unlock();
    return true;
  }
  // 2.
  frame_id_t frame_id = page_table_[page_id];
  Page *page = &pages_[frame_id];
  if (page->pin_count_ > 0) {
    latch_.unlock();
    return false;
  }
  if (page->is_dirty_) {
    FlushPageImpl(page_id);
  }
  // delete in disk in here
  disk_manager_->DeallocatePage(page_id);
  
  page_table_.erase(page_id);
  // reset metadata
  page->is_dirty_ = false;
  page->pin_count_ = 0;
  page->page_id_ = INVALID_PAGE_ID;
  // return it to the free list
  
  free_list_.push_back(frame_id);
  latch_.unlock();
  return true;
}
```

#### 3. Source Code Analysis [#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#3-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90) <a href="#3-source-code-analysis" id="3-source-code-analysis"></a>

**3.1 ResetMemory()**[**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#31-resetmemory)

This is straightforward memory allocation. It allocates the memory region for each frame.

**3.2 ReadPage**[**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#32-readpage)

```
void DiskManager::ReadPage(page_id_t page_id, char *page_data)
```

```
void DiskManager::ReadPage(page_id_t page_id, char *page_data) {
  int offset = page_id * PAGE_SIZE; // PAGE_SIZE = 4 KB; first compute the offset and check bounds
  // check if read beyond file length
  if (offset > GetFileSize(file_name_)) {
    LOG_DEBUG("I/O error reading past end of file");
    // std::cerr << "I/O error while reading" << std::endl;
  } else {
    // set read cursor to offset
    db_io_.seekp(offset); // move the file cursor to the offset
    db_io_.read(page_data, PAGE_SIZE); // read data into page_data
    if (db_io_.bad()) {
      LOG_DEBUG("I/O error while reading");
      return;
    }
    // if file ends before reading PAGE_SIZE
    int read_count = db_io_.gcount();
    if (read_count < PAGE_SIZE) {
      LOG_DEBUG("Read less than a page");
      db_io_.clear();
      // std::cerr << "Read less than a page" << std::endl;
      memset(page_data + read_count, 0, PAGE_SIZE - read_count); // if less than 4 KB is read, pad the rest with zeros
    }
  }
}
```

**3.3 WritePage**[**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#33-writepage)

```
void DiskManager::WritePage(page_id_t page_id, const char *page_data) {
  size_t offset = static_cast<size_t>(page_id) * PAGE_SIZE; // compute the offset first
  // set write cursor to offset
  num_writes_ += 1; // count writes
  db_io_.seekp(offset);
  db_io_.write(page_data, PAGE_SIZE); // write data at the offset
  // check for I/O error
  if (db_io_.bad()) {
    LOG_DEBUG("I/O error while writing");
    return;
  }
  // needs to flush to keep disk file in sync
  db_io_.flush(); // flush the stream buffer
}
```

**3.4 `DiskManager` constructor**

This constructor obtains and initializes the file streams.

```
DiskManager::DiskManager(const std::string &db_file)
    : file_name_(db_file), next_page_id_(0), num_flushes_(0), num_writes_(0), flush_log_(false), flush_log_f_(nullptr) {
  std::string::size_type n = file_name_.rfind('.');
  if (n == std::string::npos) {
    LOG_DEBUG("wrong file format");
    return;
  }
  log_name_ = file_name_.substr(0, n) + ".log";

  log_io_.open(log_name_, std::ios::binary | std::ios::in | std::ios::app | std::ios::out);
  // directory or file does not exist
  if (!log_io_.is_open()) {
    log_io_.clear();
    // create a new file
    log_io_.open(log_name_, std::ios::binary | std::ios::trunc | std::ios::app | std::ios::out);
    log_io_.close();
    // reopen with original mode
    log_io_.open(log_name_, std::ios::binary | std::ios::in | std::ios::app | std::ios::out);
    if (!log_io_.is_open()) {
      throw Exception("can't open dblog file");
    }
  }

  db_io_.open(db_file, std::ios::binary | std::ios::in | std::ios::out); // open the input/output stream for the database file
  // directory or file does not exist
  if (!db_io_.is_open()) {
    db_io_.clear();
    // create a new file
    db_io_.open(db_file, std::ios::binary | std::ios::trunc | std::ios::out);
    db_io_.close();
    // reopen with original mode
    db_io_.open(db_file, std::ios::binary | std::ios::in | std::ios::out);
    if (!db_io_.is_open()) {
      throw Exception("can't open db file");
    }
  }
  buffer_used = nullptr;
}
```

#### 4. Testing [#](https://www.cnblogs.com/JayL-zxl/p/14311883.html#4-%E6%B5%8B%E8%AF%95) <a href="#4-testing" id="4-testing"></a>

**4.1 Local test** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#41-%E6%9C%AC%E5%9C%B0%E6%B5%8B%E8%AF%95)

```bash
 cd build
 make buffer_pool_manager_test
 ./test/buffer_pool_manager_tes
```

<figure><img src="../../../.gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>

**4.2 CMU official test** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#42-cmu%E5%AE%98%E7%BD%91%E6%B5%8B%E8%AF%95)

**I later found that even non-CMU students could use their grading software for testing. After many revisions and help from more experienced people, I finally got full marks.** [**#**](https://www.cnblogs.com/JayL-zxl/p/14311883.html#%E5%90%8E%E9%9D%A2%E5%8F%91%E7%8E%B0%E5%8E%9F%E6%9D%A5%E4%B8%8D%E6%98%AFcmu%E8%87%AA%E5%B7%B1%E7%9A%84%E5%AD%A6%E7%94%9F%E4%B9%9F%E5%8F%AF%E4%BB%A5%E7%94%A8%E5%AE%83%E4%BB%AC%E7%9A%84%E8%BD%AF%E4%BB%B6%E8%BF%9B%E8%A1%8C%E6%B5%8B%E8%AF%95%E4%BF%AE%E6%94%B9%E4%BA%86%E5%A5%BD%E4%B9%85%E5%90%8C%E6%97%B6%E5%BE%97%E5%88%B0%E4%BA%86%E5%A4%A7%E4%BD%AC%E7%9A%84%E5%B8%AE%E5%8A%A9%E6%89%8D%E6%88%90%E5%8A%9F%E5%AE%9E%E7%8E%B0%E6%BB%A1%E5%88%86)

The CMU testing site is:

{% embed url="https://www.gradescope.com/courses/195440" %}
