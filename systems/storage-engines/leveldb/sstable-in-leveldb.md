---
description: Continuation of the previous SSTable structure notes.
---

# 🤓 SSTable in LevelDB

#### **Data Block**

A Data Block stores a sequence of ordered key-value entries. To reduce storage overhead, LevelDB applies several optimizations inside the data block format.

The first thing to note is that a data block is variable-sized.

```tex
"block_size" is not a "size", it is a threshold.
Data is never split across blocks.
A single block contains one or more key/value pairs.
Leveldb starts a new block only when the total size of all key/values in the current block exceed the threshold.
```

In practice, LevelDB starts flushing the current data block and appends it to the `sstable file` once it reaches `options.block_size`. This logic appears in **table/table\_build.cc**:

```cpp
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  ...
  r->data_block.Add(key, value);

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```

The default value of `options.block_size` is 4 KB.

Notice that many adjacent keys may share repeated bytes. For example, `"hellokitty"` and `"helloworld"` can be two neighboring keys.

Therefore, extracting and storing the shared prefix can save storage space effectively.

Based on this observation, LevelDB uses prefix compression. Since keys in LevelDB are stored in sorted order, prefix compression can significantly reduce the space occupied by repeated key prefixes.

In addition, after every 16 keys, LevelDB disables prefix compression and stores the complete key instead. In the current implementation, this interval is controlled by `options_->block_restart_interval`, whose default value is 16. A position where the complete key is stored is called a **restart point**.

![img](https://s2.loli.net/2022/07/24/A6DlYdV5hHxoXMP.png)

The entry point for adding a key-value pair to an SSTable is:

```cpp
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->num_entries > 0) {
    assert(r->options.comparator->Compare(key, Slice(r->last_key)) > 0);
  }

  /* Ignore the index block part for now. */
  if (r->pending_index_entry) {
    assert(r->data_block.empty());
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;
    r->pending_handle.EncodeTo(&handle_encoding);
    r->index_block.Add(r->last_key, Slice(handle_encoding));
    r->pending_index_entry = false;
  }

  /* Ignore the filter block part for now. */
  if (r->filter_block != NULL) {
    r->filter_block->AddKey(key);
  }

  r->last_key.assign(key.data(), key.size());
  r->num_entries++;
  /* Add a key-value pair to the data block. */
  r->data_block.Add(key, value);

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```

For now, ignore the index block and filter block logic and focus on how a data block adds a key-value pair:

```cpp
void BlockBuilder::Add(const Slice& key, const Slice& value) {
  Slice last_key_piece(last_key_);
  assert(!finished_);
  assert(counter_ <= options_->block_restart_interval);
  assert(buffer_.empty() // No values yet?
         || options_->comparator->Compare(key, last_key_piece) > 0);
  size_t shared = 0;
  if (counter_ < options_->block_restart_interval) {
    // See how much sharing to do with previous string
    const size_t min_length = std::min(last_key_piece.size(), key.size());
    while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
      shared++;
    }
  } else {
    // Restart compression
    /* This is a new restart point, so record the current position. */
    restarts_.push_back(buffer_.size());
    counter_ = 0;
  }
  const size_t non_shared = key.size() - shared;

  // Add "<shared><non_shared><value_size>" to buffer_
  PutVarint32(&buffer_, shared);
  PutVarint32(&buffer_, non_shared);
  PutVarint32(&buffer_, value.size());

  // Add string delta to buffer_ followed by value
  buffer_.append(key.data() + shared, non_shared);
  buffer_.append(value.data(), value.size());

  // Update state
  last_key_.resize(shared);
  last_key_.append(key.data() + shared, non_shared);
  assert(Slice(last_key_) == key);
  counter_++;
}
```

Each record in a data block has the following layout:

![](http://bean-li.github.io/assets/LevelDB/data\_block\_record.png)

When the current `data_block` contains enough content and its estimated size exceeds the configured threshold, LevelDB starts flushing the block. Flushing a data block means appending all restart-point offsets and then writing the number of restart points:

```cpp
Slice BlockBuilder::Finish() {
  // Append restart array
  for (size_t i = 0; i < restarts_.size(); i++) {
    PutFixed32(&buffer_, restarts_[i]);
  }
  PutFixed32(&buffer_, restarts_.size());
  finished_ = true;
  return Slice(buffer_);
}
```

This completes the data-block part of an SSTable file.

However, one SSTable contains multiple data blocks. Although these data blocks are ordered, a lookup still needs to answer a practical question: which block contains the target key? A typical SSTable file is about 2 MB, and it can be configured to be larger. Each SSTable may contain hundreds of data blocks. How should LevelDB locate the key among all those blocks?

Sequentially scanning the data blocks is clearly inefficient. This is where the index block becomes necessary.
