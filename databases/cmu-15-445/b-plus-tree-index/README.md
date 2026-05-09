---
description: 'PROJECT #2 - B+TREE'
---

# B+ Tree Index

> An **index** is responsible for retrieving data quickly without scanning every row in a database table. It also provides fast random access and efficient access to ordered records. In this programming project, we need to implement a **dynamic index structure based on a B+ tree**.

### CHECKPOINT #1

#### TASK #1 - B+TREE PAGES

Implement three `Page` classes to store B+ tree data:

* B+Tree Parent Page
* B+Tree Internal Page
* B+Tree Leaf Page

**Implementation of b\_plus\_tree\_page.cpp**

> Directly set or return the corresponding fields according to each function's requirements.

* `IsRootPage` returns `true` or `false` according to whether `parent_id_` is `INVALID_PAGE_ID`.
* `GetMinSize()` needs to distinguish three cases: 1) root node and leaf node, 2) root node, and 3) all other nodes.

**Implementation of b\_plus\_tree\_internal\_page.cpp**

* `MappingType` is `std::pair<KeyType, ValueType>`. How do we get the `i`-th `MappingType`? In the header file, there is a `MappingType` array named `array`; `array[0]` indicates that this is a flexible array member.
* Some functions need to validate whether `index` is legal. Use `assert()`.
* Convert the page fetched from `bpm` into an `internal page` or `leaf page`. This is important.

{% code title="b_plus_tree_internal_page.cpp" overflow="wrap" lineNumbers="true" %}
```cpp
Page *page = buffer_pool_manager_->FetchPage(page_id);
assert(page != nullptr);
// Convert page into the BPlusTreeInternalPage class.
BPlusTreeInternalPage *bpt_node = reinterpret_cast<BPlusTreeInternalPage *>(page->GetData());
// do something
buffer_pool_manager_->UnpinPage(bpt_node->GetPageId(), flag); // Whether the page was modified; flag is true or false.
```
{% endcode %}

* **Helper method**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
// Return the key at index.
KeyType KeyAt(int index) const;
// Set the key at index to key.
void SetKeyAt(int index, const KeyType &key);
// Return the index in array where Value == value.
int ValueIndex(const ValueType &value) const;
// Return the value at index.
ValueType ValueAt(int index) const;
```
{% endcode %}

* **Lookup**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/**
 * Use binary search to find the first index whose key is greater than or equal to key.
 * Note that the left boundary starts from 1.
 */
ValueType Lookup(const KeyType &key, const KeyComparator &comparator) const;
```
{% endcode %}

* **Insertion**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/**
 * When insertion overflows from a leaf page up to the root page, create a new root page and initialize it.
 * array[0].first = old_value
 * Note: this method should only be called by InsertIntoParent() in b_plus_tree.cpp.
 */
void PopulateNewRoot(const ValueType &old_value, const KeyType &new_key, const ValueType &new_value);
/**
 * Insert a <Key, Value> after the position of old_value.
 * Note: ValueIndex can be used to find the index of old_value. Remember to update size_ after insertion.
 */
int InsertNodeAfter(const ValueType &old_value, const KeyType &new_key, const ValueType &new_value);
```
{% endcode %}

* **Remove**

```cpp
/**
 * Remove array[index]. Elements after it are shifted forward by one position.
 * Note: remember to update size_ after removing the element.
 */
void Remove(int index);
/**
 * Remove the only element in the internal page and return its value.
 */
ValueType RemoveAndReturnOnlyChild();
```

* **Split and Merge utility methods**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/**
 * Split: because inserting a <K, V> makes the page full, move half of this page's <K, V> entries
 * into the "recipient" page.
 */
void MoveHalfTo(BPlusTreeInternalPage *recipient, BufferPoolManager *buffer_pool_manager);

// Helper function: copy <K, V> entries and update the parent_page_id of the moved child nodes.
void CopyNFrom(MappingType *items, int size, BufferPoolManager *buffer_pool_manager);

/**
 * Merge: move all <K, V> entries from this page into the "recipient" page.
 */
void MoveAllTo(BPlusTreeInternalPage *recipient, const KeyType &middle_key, BufferPoolManager *buffer_pool_manager);

/****** Redistribute ******/
/**
 * Borrow the first valid <K, V> entry from the current page.
 * Move <middle_key, ValueAt(0)> to the last slot of the "recipient" page.
 * Then update the middle_key in the parent page.
 */
void MoveFirstToEndOf(BPlusTreeInternalPage *recipient, const KeyType &middle_key,
                    BufferPoolManager *buffer_pool_manager);
/**
 * Append pair to the end of array, then update this entry's parent page id.
 */
void CopyLastFrom(const MappingType &pair, BufferPoolManager *buffer_pool_manager);
/**
 * Borrow the last valid <K, V> entry from the current page.
 * Move <middle_key, ValueAt(size - 1)> to the first slot of the "recipient" page.
 * Then update the middle_key in the parent page.
 */
void MoveLastToFrontOf(BPlusTreeInternalPage *recipient, const KeyType &middle_key,
                     BufferPoolManager *buffer_pool_manager);
/**
 * Add pair to the first valid position of array, then update this entry's parent page id.
 */
void CopyFirstFrom(const MappingType &pair, BufferPoolManager *buffer_pool_manager);
```
{% endcode %}

Note: the `Merge` and `Redistribute` helper methods can be completed in CP2.

**Implementation of b\_plus\_tree\_Leaf\_page.cpp**

* **Helper method**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
// Return the page id of the next page.
page_id_t GetNextPageId() const;
// Set next_page_id.
void SetNextPageId(page_id_t next_page_id);
// Return the key at array[index].
KeyType KeyAt(int index) const;
// Return the first index in array whose key is greater than or equal to key.
int KeyIndex(const KeyType &key, const KeyComparator &comparator) const;
// Return array[index].
const MappingType &GetItem(int index);
```
{% endcode %}

* **Insertion, Search, Delete**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/**
 * Insert <key, value> into the leaf node and return the page size after insertion.
 * 1. Find the insertion position and check whether the key at that position equals "key".
 *    Because only unique keys are supported, return the page size directly if the key already exists.
 * 2. Perform the insertion.
 * 3. Update size_.
 */
int Insert(const KeyType &key, const ValueType &value, const KeyComparator &comparator);
/**
 * Check whether key exists in the leaf node.
 * If it exists, store the corresponding value in "value" and return true; otherwise return false.
 */
bool Lookup(const KeyType &key, ValueType *value, const KeyComparator &comparator) const;
/**
 * Delete key from the leaf node.
 * If key does not exist, return the page size directly.
 * Otherwise delete the key, update size_, and return the page size.
 */
int RemoveAndDeleteRecord(const KeyType &key, const KeyComparator &comparator);
```
{% endcode %}

* **Split and Merge utility methods**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
// Similar to the implementation in BPlusTreeInternalPage.
void MoveHalfTo(BPlusTreeLeafPage *recipient);
void CopyNFrom(MappingType *items, int size);
void MoveAllTo(BPlusTreeLeafPage *recipient);
void MoveFirstToEndOf(BPlusTreeLeafPage *recipient);
void CopyLastFrom(const MappingType &item);
void MoveLastToFrontOf(BPlusTreeLeafPage *recipient);
void CopyFirstFrom(const MappingType &item);
```
{% endcode %}

Comment: the four `Move` functions above have parameters that differ from the same-named functions in `BPlusTreeInternalPage`, which caused compile errors later when calling them from `Split`. I changed this group of function parameters to make them consistent before the project compiled successfully.

***

#### TASK #2.A - B+TREE DATA STRUCTURE (INSERTION & POINT SEARCH)

* `FindLeafPage` is the foundation of **lookup, insertion, and deletion**. It finds the leaf node where `key` should be located. When `leftMost = true`, it returns the leftmost leaf node in the B+ tree.
* In `InsertIntoLeaf`, if the number of `<K, V>` entries in the leaf node equals `GetMaxSize()` after insertion, the leaf needs to split.
* In `InsertIntoParent`, an internal node splits only when the number of `<K, V>` entries after insertion is greater than `GetMaxSize()`.

***

### CHECKPOINT #2

#### TASK #3 - INDEX ITERATOR

> The key point in implementing the iterator is deciding when iteration ends: it ends at the last slot of the rightmost leaf node. The `++` operation also needs to handle moving to the next leaf node.

***

#### TASK #4 - CONCURRENT INDEX

* During `coalesce` or `redistribute`, obtain the sibling's `W Latch`.
* To protect `root_page_id_`, use a virtual page `v_root_page` together with a `mutex_`.

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/** 
 * Define member variables Page v_root_page and std::mutex mutex_ in the b_plus_tree class.
 * Every time FindLeafPage runs, acquire mutex_ first, then add &v_root_page into transaction->page_set_.
 * Releasing mutex_ depends on whether the root page is safe.
 */
```
{% endcode %}

* Crabbing locking: the unlock order should be consistent with the lock acquisition order.

{% code overflow="wrap" lineNumbers="true" %}
```cpp
/**
 * 1. If op = search, after acquiring the child node's RLatch, release the pages previously added to page_set_.
 * 2. If op = insert, after acquiring the child node's WLatch, check whether the page is safe (size < max_size_).
 *    If it is safe, release all previous pages.
 * 3. If op = delete, after acquiring the child node's WLatch, check whether the page is safe (size > min_size_).
 *    If it is safe, release all previous pages.
 * Finally, add the current page to page_set_.
 */
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::LockThisPage(int op, Page *page, Transaction *transaction) {}

/**
 * 1. Release the R/W latches in page_set_.
 * 2. Delete the pages stored in delete_page_set_, then clear delete_page_set_.
 */
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::UnlockPrevPage(int op, Transaction *transaction, bool is_dirty) {}
```
{% endcode %}

***

### Testing / Verification / Packaging

* Tests

```bash
cd build
make b_plus_tree_insert_test
./test/b_plus_tree_insert_test
make b_plus_tree_delete_test
./test/b_plus_tree_delete_test
make b_plus_tree_concurrent_test
./test/b_plus_tree_concurrent_test
```

* Format checks

```bash
make format
make check-lint
make check-clang-tidy
```

* Packaging

```bash
zip project2-submission.zip src/include/buffer/lru_replacer.h src/buffer/lru_replacer.cpp \
	src/include/buffer/buffer_pool_manager.h src/buffer/buffer_pool_manager.cpp \
	src/include/storage/page/b_plus_tree_page.h src/storage/page/b_plus_tree_page.cpp \
	src/include/storage/page/b_plus_tree_internal_page.h src/storage/page/b_plus_tree_internal_page.cpp \
	src/include/storage/page/b_plus_tree_leaf_page.h src/storage/page/b_plus_tree_leaf_page.cpp \
	src/include/storage/index/b_plus_tree.h src/storage/index/b_plus_tree.cpp \
	src/include/storage/index/index_iterator.h src/storage/index/index_iterator.cpp
```

Then go to [**Gradescope**](https://www.gradescope.com/) and submit the code.
