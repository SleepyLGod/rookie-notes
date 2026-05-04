---
description: 接续上文sstable结构
---

# 🤓 SsTable in LevelDB

#### **Data Block**

Data Block 存放的就是一系列有序的key-value，为了节省存储空间，Data block做了一些改进。

首先要了解到，data block是变长的。

```tex
"block_size" is not a "size", it is a threshold.  
Data is never split across blocks.  
A single block contains one or more key/value pairs.
Leveldb starts a new block only when the total size of all key/values in the current block exceed the threshold.
```

只不过它总是在写满`options.block_size`的时候开始`Flush`，追加写入到`sstable file`，如**table/table\_build.cc**中的

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

而这个`options.block_size`默认为4KB。

注意，很多key可能有**重复的字节**，比如“hellokitty”和”helloworld“是两个相邻的key。

因此，如果**将公共的部分提取**，可以有效的节省存储空间。

处于这种考虑，LevelDB采用了前缀压缩(**prefix-compressed**)，由于LevelDb中key是按序排列的，这可以显著的减少空间占用。

另外，每间隔16个keys(**目前版本中`options_->block_restart_interval`默认为16**)，LevelDB就取消使用前缀压缩，而是**存储整个key**(我们把存储整个key的点叫做**重启点**)。

![img](https://s2.loli.net/2022/07/24/A6DlYdV5hHxoXMP.png)

向sstable添加一个key－value，函数的入口点是：

```cpp
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->num_entries > 0) {
    assert(r->options.comparator->Compare(key, Slice(r->last_key)) > 0);
  }

  /*此处我们先忽略index block的部分*/
  if (r->pending_index_entry) {
    assert(r->data_block.empty());
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;
    r->pending_handle.EncodeTo(&handle_encoding);
    r->index_block.Add(r->last_key, Slice(handle_encoding));
    r->pending_index_entry = false;
  }

  /*此处我们先忽略filter block的部分*/
  if (r->filter_block != NULL) {
    r->filter_block->AddKey(key);
  }

  r->last_key.assign(key.data(), key.size());
  r->num_entries++;
  /* 向data block中添加一组key-value pair */
  r->data_block.Add(key, value);

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```

我们先忽略index block和filter block的部分，集中精力查看**data block如何新增key－value对**：

```cpp
void BlockBuilder::Add(const Slice& key, const Slice& value) {
  Slice last_key_piece(last_key_);
  assert(!finished_);
  assert(counter_ <= options_->block_restart_interval);
  assert(buffer_.empty() // No values yet?
         || options_->comparator->Compare(key, last_key_piece) > 0);
  size_t shared = 0;
  if (counter_ < options_->block_restart_interval) {`
    // See how much sharing to do with previous string
    const size_t min_length = std::min(last_key_piece.size(), key.size());
    while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
      shared++;
    }
  } else {
    // Restart compression
    /*新的重启点，记录下位置*/
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

对于data block中的每一个记录，其格式如下：

![](http://bean-li.github.io/assets/LevelDB/data\_block\_record.png)

当前data\_block中的内容足够多时，预计大于预设的门限值的时候，就开始flush，所谓data block的Flush就是将所有的重启点指针记录下来，并且记录重启点的个数：

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

至此，介绍完了SSTable文件中的data block。

注意，一个SSTable中存在着多个data block，尽管他们之间是有序的，可是你查找的key到底位于哪个block上？典型的sstable文件大小为2M，可以设置的更大，每个sstable 文件中data block 的个数可能上百，如何在这上百个data block中寻找你要的key？

显然依次查找效率太低，这时候 index block就起到作用了。
