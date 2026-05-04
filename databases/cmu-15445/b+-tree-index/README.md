---
description: 'PROJECT #2 - B+TREE'
---

# 😉 B+ Tree Index

> **索引**负责快速地检索数据，而不需要搜索数据库表中的每一行；索引还提供了快速的随机访问以及高效地访问有序记录。因此，在这个编程作业中，我们需要实现**基于 B+ 树的动态索引结构**。

### CHECKPOINT #1

#### TASK #1 - B+TREE PAGES

实现三个 `Page` 类来存储 B+ 树的数据

* B+Tree Parent Page
* B+Tree Internal Page
* B+Tree Leaf Page

**b\_plus\_tree\_page.cpp 的实现**

> 直接根据函数要求设置或返回对应的字段即可。

* `IsRootPage` 函数根据 `parent_id_` 是否是 `INVALID_PAGE_ID` 返回 `true` 或者 `false`
* `GetMinSize()`：需要区分 1）根结点且为叶子结点。2）根结点。3）其他

**b\_plus\_tree\_internal\_page.cpp 的实现**

* `MappingType` 的类型为 `std::pair<KeyType, ValueType>`，如何获取第 `i` 个 `MappingType`？=> 在其头文件有一个 `array` 的 `MappingType` 数组（`array[0]` 表示这是一个弹性数组）
* 注意有些函数需要判断 `index` 是否合法，使用 `assert()`
* 将 `bpm` 取出的页面转换为 `internal page/leaf page`（重要）

{% code title="b_plus_tree_internal_page.cpp" overflow="wrap" lineNumbers="true" %}
```cpp
Page *page = buffer_pool_manager_->FetchPage(page_id);
assert(page != nullptr);
// 将 page 转成 BPlusTreeInternalPage 类
BPlusTreeInternalPage *bpt_node = reinterpret_cast<BPlusTreeInternalPage *>(page->GetData());
// do something
buffer_pool_manager_->UnpinPage(bpt_node->GetPageId(), flag); // 是否修改了页面，flag 为 true 或者 false
```
{% endcode %}

* **Helper method**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
// 返回 index 这个位置的 Key
KeyType KeyAt(int index) const;
// 设置 index 这个位置的 Key = key
void SetKeyAt(int index, const KeyType &key);
// 返回 array 中 Value == value 的下标
int ValueIndex(const ValueType &value) const;
// 返回 index 这个位置的 Value
ValueType ValueAt(int index) const;
```
{% endcode %}

* **Lookup**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/**
 * 使用二分搜索找到第一个大于等于 key 的下标，注意左区间从 1 开始
 */
ValueType Lookup(const KeyType &key, const KeyComparator &comparator) const;
```
{% endcode %}

* **Insertion**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/**
 * 当插入导致从 leaf page 到 root page 溢出时，我们应该创建一个新的 root page，然后初始化它
 * array[0].first = old_value
 * 注：这个方法只能由 InsertIntoParent() 调用(b_plus_tree.cpp)
 */
void PopulateNewRoot(const ValueType &old_value, const KeyType &new_key, const ValueType &new_value);
/**
 * 在 old_value 位置之后插入一个 <Key, Value>
 * 注：可以使用 ValueIndex 来找到 old_value 的下标；插入后别忘了更新 size_
 */
int InsertNodeAfter(const ValueType &old_value, const KeyType &new_key, const ValueType &new_value);
```
{% endcode %}

* **Remove**

```cpp
/**
 * 删除 array[index] 这个元素（后面的元素向前移一位）
 * 注：删除元素后别忘了更新 size_
 */
void Remove(int index);
/**
 * 删除 internal page 中的唯一一个元素，并返回它的 value
 */
ValueType RemoveAndReturnOnlyChild();
```

* **Split and Merge utility methods**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/**
 * Split: 由于插入一个 <K, V> 导致 page full，因此将这一页一半的 <K, V> 移到 "recipient" 页
 */
void MoveHalfTo(BPlusTreeInternalPage *recipient, BufferPoolManager *buffer_pool_manager);

// 辅助函数，执行 <K, V> 拷贝操作，同时更新这些被移动节点的 parent_page_id
void CopyNFrom(MappingType *items, int size, BufferPoolManager *buffer_pool_manager);

/**
 * Merge: 将该页的所有 <K, V> 移到 "recipient" 页
 */
void MoveAllTo(BPlusTreeInternalPage *recipient, const KeyType &middle_key, BufferPoolManager *buffer_pool_manager);

/****** Redistribute ******/
/**
 * 借当前页的第一个有效的 <K, V>
 * 将 <middle_key, ValueAt(0)> 移到 "recipient" page 的最后一个 slot
 * 然后更新 parent page 的 middl_key
 */
void MoveFirstToEndOf(BPlusTreeInternalPage *recipient, const KeyType &middle_key,
                    BufferPoolManager *buffer_pool_manager);
/**
 * 将 pair 添加到 array 末尾，然后更新这个 entry 的 parent page id
 */
void CopyLastFrom(const MappingType &pair, BufferPoolManager *buffer_pool_manager);
/**
 * 借当前页的最后一个有效的 <K, V>
 * 将 <middle_key, ValueAt(size - 1)> 移到 "recipient" page 的第一个 slot
 * 然后更新 parent page 的 middl_key
 */
void MoveLastToFrontOf(BPlusTreeInternalPage *recipient, const KeyType &middle_key,
                     BufferPoolManager *buffer_pool_manager);
/**
 * 将 pair 添加到 array 的第一个有效位置，然后更新这个 entry 的 parent page id
 */
void CopyFirstFrom(const MappingType &pair, BufferPoolManager *buffer_pool_manager);
```
{% endcode %}

注：`Merge` 和 `Redistribute` 的几个函数可以放在 CP2 中完成

**b\_plus\_tree\_Leaf\_page.cpp 的实现**

* **Helper method**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
// 返回当前 page 的下一个 page 的 page id
page_id_t GetNextPageId() const;
// 设置 next_page_id
void SetNextPageId(page_id_t next_page_id);
// 返回 array[index] 位置的 Key
KeyType KeyAt(int index) const;
// 返回 array 中第一个大于等于 key 的 index
int KeyIndex(const KeyType &key, const KeyComparator &comparator) const;
// 返回 array[index]
const MappingType &GetItem(int index);
```
{% endcode %}

* **Insertion、Search、Delete**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/**
 * 将 <key, value> 插入到叶子节，返回插入后的 page size
 * 1. 找到需要插入到位置，判断该位置的 key 是否等于 "key"，由于只支持 unique key，所以相等时直接返回 page size
 * 2. 执行插入操作
 * 3. 更新 size_
 */
int Insert(const KeyType &key, const ValueType &value, const KeyComparator &comparator);
/**
 * 查找 key 是否在叶子节点中。如果在，将 key 对应的 Value 存在 "value" 中，返回 true；否则返回 false
 */
bool Lookup(const KeyType &key, ValueType *value, const KeyComparator &comparator) const;
/**
 * 在叶子节点中删除 key
 * 如果 key 不存在，直接返回 page size
 * 否则删除这个 key，更新 size_，返回 page size
 */
int RemoveAndDeleteRecord(const KeyType &key, const KeyComparator &comparator);
```
{% endcode %}

* **Split and Merge utility methods**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
// 类似于 BPlusTreeInternalPage 的实现
void MoveHalfTo(BPlusTreeLeafPage *recipient);
void CopyNFrom(MappingType *items, int size);
void MoveAllTo(BPlusTreeLeafPage *recipient);
void MoveFirstToEndOf(BPlusTreeLeafPage *recipient);
void CopyLastFrom(const MappingType &item);
void MoveLastToFrontOf(BPlusTreeLeafPage *recipient);
void CopyFirstFrom(const MappingType &item);
```
{% endcode %}

吐槽：上面的四个 `Move` 函数和 `BPlusTreeInternalPage` 类中的同名函数参数不一致，导致后面 `Split` 中调用报错。之后我将这一组函数的参数改为一致才编译通过。

***

#### TASK #2.A - B+TREE DATA STRUCTURE (INSERTION & POINT SEARCH)

* `FindLeafPage` 这个函数是**查找、插入、删除**操作的基础，它是找到 `key` 所在的叶子节点，`leftMost = true` 时表示返回 B+ 树中最左边的叶子节点
* 在 `InsertIntoLeaf` 函数中，如果插入后，叶子结点包含的 `<K, V>` 个数等于 `GetMaxSize()`，此时就要进行分裂
* 在 `InsertIntoParent` 函数中，插入后的中间结点包含的 `<K, V>` 个数大于 `GetMaxSize()` 才进行分裂

***

### CHECKPOINT #2

#### TASK #3 - INDEX ITERATOR

> 迭代器的实现主要是注意什么时候结束？（到达最右边叶子节点的最后一个 slot）以及实现 `++` 操作时，处理下一个叶子节点的情况。

***

#### TASK #4 - CONCURRENT INDEX

* `coalesce` 或者 `redistribute` 操作时需要获得 sibling 的 `W Latch`
* 为了保护 `root_page_id_`，采用了虚拟页 `v_root_page`，和一个 `mutex_`

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/** 
 * 在 b_plus_tree class 里定义一个成员变量 Page v_root_page 和 std::mutex mutex_
 * 每次 FindLeafPage 的时候先使用 mutex_，再把 &v_root_page 加进 transaction->page_set_
 * mutex_ 的释放取决于 root page 是否安全
 */
```
{% endcode %}

* Crabbing Locking（解锁顺序和加锁顺序要一致）

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/**
 * 1. 如果 op = search，那么获取到子节点的 RLatch 后，应该释放之前添加到 page_set_ 的页面
 * 2. 如果 op = insert，那么获取到子节点的 WLatch 后，检查该 page 是否安全(size < max_size_)；是的话释放之前所有页面
 * 3. 如果 op = delete，那么获取到子节点的 WLatch 后，检查该 page 是否安全(size > min_size_)；是的话释放之前所有页面
 * 最后添加该页到 page_set_
 */
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::LockThisPage(int op, Page *page, Transaction *transaction) {}

/**
 * 1. 释放 page_set_ 中的 R/W Latch
 * 2. 删除保存着 delete_page_set_ 中的页面，然后清空 delete_page_set_
 */
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::UnlockPrevPage(int op, Transaction *transaction, bool is_dirty) {}
```
{% endcode %}

***

### 测试/验证/打包

* 测试

```bash
cd build
make b_plus_tree_insert_test
./test/b_plus_tree_insert_test
make b_plus_tree_delete_test
./test/b_plus_tree_delete_test
make b_plus_tree_concurrent_test
./test/b_plus_tree_concurrent_test
```

* 格式验证

```bash
make format
make check-lint
make check-clang-tidy
```

* 打包

```bash
zip project2-submission.zip src/include/buffer/lru_replacer.h src/buffer/lru_replacer.cpp \
	src/include/buffer/buffer_pool_manager.h src/buffer/buffer_pool_manager.cpp \
	src/include/storage/page/b_plus_tree_page.h src/storage/page/b_plus_tree_page.cpp \
	src/include/storage/page/b_plus_tree_internal_page.h src/storage/page/b_plus_tree_internal_page.cpp \
	src/include/storage/page/b_plus_tree_leaf_page.h src/storage/page/b_plus_tree_leaf_page.cpp \
	src/include/storage/index/b_plus_tree.h src/storage/index/b_plus_tree.cpp \
	src/include/storage/index/index_iterator.h src/storage/index/index_iterator.cpp
```

然后前往 [**https://www.gradescope.com**](https://www.gradescope.com/) 提交代码
