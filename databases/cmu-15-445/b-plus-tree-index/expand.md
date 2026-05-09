---
description: Project 2
---

# B+Tree Expansion

### 1. Lab Overview <a href="#1-lab-overview" id="1-lab-overview"></a>

> The first grading checkpoint implements the basic B+Tree structure, insertion, and search operations.
>
> This part does not consider the concurrency requirements of the second checkpoint, so locking, unlocking, and transactions are not covered here.

* [**Task #1 - B+Tree Pages**](https://15445.courses.cs.cmu.edu/fall2020/project2/#b+tree-pages)
* [**Task #2.a - B+Tree Data Structure (Insertion & Point Search)**](https://15445.courses.cs.cmu.edu/fall2020/project2/#b+tree-structure-1)

> The second grading checkpoint implements B+Tree deletion, the index iterator, and support for concurrent access.

* [**Task #2.b - B+Tree Data Structure (Deletion)**](https://15445.courses.cs.cmu.edu/fall2020/project2/#b+tree-structure-2)
* [**Task #3 - Index Iterator**](https://15445.courses.cs.cmu.edu/fall2020/project2/#index-iterator)
* [**Task #4 - Concurrent Index**](https://15445.courses.cs.cmu.edu/fall2020/project2/#concurrent\_index)

### Task 1 B+TREE PAGES[#](https://www.cnblogs.com/JayL-zxl/p/14324297.html#task-1-btree-pages) <a href="#task-1-btree-pages" id="task-1-btree-pages"></a>

You need to implement three page classes to store B+Tree data.

* B+ Tree Parent Page
* B+ Tree Internal Page
* B+ Tree Leaf Page

#### <mark style="color:purple;">1.</mark>  B+ Tree Parent Page <a href="#1-b-tree-parent-page" id="1-b-tree-parent-page"></a>

This is the parent class inherited by both internal pages and leaf pages. It contains only the information shared by those two subclasses. The parent page is divided into the fields shown below.\
`*`_`B+Tree Parent Page Content`_

| Variable Name      | Size | Description                             |
| ------------------ | ---- | --------------------------------------- |
| page\_type\_       | 4    | Page Type (internal or leaf)            |
| lsn\_              | 4    | Log sequence number (Used in Project 4) |
| size\_             | 4    | Number of Key & Value pairs in page     |
| max\_size\_        | 4    | Max number of Key & Value pairs in page |
| parent\_page\_id\_ | 4    | Parent Page Id                          |
| page\_id\_         | 4    | Self Page Id                            |

The parent page must be implemented in the specified files. Only the header file (`src/include/storage/page/b_plus_tree_page.h`) and its corresponding source file (`src/storage/page/b_plus_tree_page.cpp`) should be modified.

Most methods here are simple setters and getters, so they are not listed in detail.

* `IsRootPage` returns `true` or `false` depending on whether `parent_id_` is `INVALID_PAGE_ID`.
* `GetMinSize()` needs to distinguish three cases: **1**) the root is also a leaf, **2**) the page is the root, and **3**) all other pages.

#### <mark style="color:purple;">2.</mark> B+TREE INTERNAL PAGE <a href="#2-btree-internal-page" id="2-btree-internal-page"></a>

Internal pages do not store actual tuple data. Instead, they store `m` ordered key entries and `m + 1` pointers, also called `page_id`s.&#x20;

Because the number of pointers is not equal to the number of keys, the first key is set as invalid, and lookup should always start from the second key.&#x20;

At all times, each internal page should be at least half full, subject to the usual root exceptions. During deletion, two half-full pages may be merged into a valid page, or entries may be redistributed to avoid merging. During insertion, a full page may be split into two pages.

You may only modify the header file (`src/include/storage/page/b_plus_tree_internal_page.h`) and the corresponding source file (`src/page/b_plus_tree_internal_page.cpp`).

{% code overflow="wrap" lineNumbers="true" %}
```cpp
* Internal page format (keys are stored in increasing order):
*  --------------------------------------------------------------------------
* | HEADER | KEY(1)+PAGE_ID(1) | KEY(2)+PAGE_ID(2) | ... | KEY(n)+PAGE_ID(n) |
*  --------------------------------------------------------------------------
#define INDEX_TEMPLATE_ARGUMENTS template <typename KeyType, typename ValueType, typename KeyComparat>
```
{% endcode %}

#### <mark style="color:purple;">3.</mark>  B+TREE LEAF PAGE <a href="#3-btree-leaf-page" id="3-btree-leaf-page"></a>

Leaf pages store `m` ordered key entries and `m` value entries.&#x20;

In this implementation, the value is only a 64-bit `record_id` used to locate the actual tuple storage position. See the `RID` class defined in `src/include/common/rid.h`.&#x20;

Leaf pages follow similar occupancy constraints on key/value pairs and should follow the same merge, redistribution, and split logic.

The leaf page must be implemented in the specified files. Only the header file (`src/include/storage/page/b_plus_tree_leaf_page.h`) and its corresponding source file (`src/storage/page/b_plus_tree_leaf_page.cpp`) should be modified.

<mark style="color:red;">**Important**</mark>: **the `KeyIndex` function**

This function returns the index of the first key that is greater than or equal to the input key. It is frequently used during insertion and makes the code reusable. `LevelDB` has a similar operation.

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_LEAF_PAGE_TYPE::KeyIndex(const KeyType &key, const KeyComparator &comparator) const {
  // binary search
  // TODO: std::lower_bound() would be cleaner
  int l = 0;
  int r = GetSize();
  if (l >= r) {
    return GetSize();
  }
  while (l < r) {
    int mid = (l + r) / 2;
    if (comparator(array_[mid].first, key) < 0) {
      l = mid + 1;
    } else {
      r = mid;
    }
  }
  return l;
}
```
{% endcode %}

Important note: although leaf pages and internal pages contain the same key type, they may use different value types, so their maximum sizes may differ.

Each `B+Tree` leaf/internal page corresponds to the contents of a storage page fetched from the buffer pool, namely the `data_` region. Therefore, whenever reading or writing a leaf/internal page, first fetch the page from the buffer pool using its unique `page_id`, reinterpret it as a leaf or internal page, and then `unpin` it after writing or deleting.

### Task 2.A - B+TREE DATA STRUCTURE (INSERTION & POINT SEARCH) <a href="#task-2a---btree-data-structure-insertion--point-search" id="task-2a---btree-data-structure-insertion--point-search"></a>

> This is essentially implementing the functions related to `b_plus_tree.cpp/InsertIntoLeaf`.

The B+Tree index here supports only unique keys. In other words, if you try to insert a key/value pair with a duplicate key, the operation should return `false`.

For `checkpoint1`, the B+Tree index only needs to support insertion (`Insert`) and point search (`GetValue`). Deletion is not required.&#x20;

After insertion, if the number of key/value pairs reaches `max_size`, the page should be split correctly. Because any write operation may change the `root_page_id` of the B+Tree index, the `root_page_id` in `src/include/storage/page/header_page.h` must be updated to ensure that the index root is durable on disk. The `BPlusTree` class already provides a function named `UpdateRootPageId`; call it whenever the B+Tree index's `root_page_id` changes.

Your B+Tree implementation must hide details such as the concrete key and value types. The suggested structure is:

{% code overflow="wrap" lineNumbers="true" %}
```cpp
template <typename KeyType,
          typename ValueType,
          typename KeyComparator>
class BPlusTree{
   // ---
};
```
{% endcode %}

These types have already been implemented for you:

* `KeyType`: The type of each key in the index. This will only be `GenericKey`, the actual size of `GenericKey` is specified and instantiated with a template argument and depends on the data type of indexed attribute.
* `ValueType`: The type of each value in the index. This will only be 64-bit RID.
* `KeyComparator`: The class used to compare whether two `KeyType` instances are less/greater-than each other. These will be included in the `KeyType` implementation files.

1. You must use the incoming `transaction` to track pages that have already been latched.
2. The project provides a reader-writer latch implementation in `src/include/common/rwlatch.h`. Helper functions for acquiring and releasing latches have also been added to the page header in `src/include/storage/page/page.h`.

First, here is the B+Tree insertion algorithm from the textbook.

<figure><img src="../../../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

**Analysis of the cases above**

#### **1. If the tree is empty, create a leaf page that is also the root** <a href="#1-create-root-leaf-for-empty-tree" id="1-create-root-leaf-for-empty-tree"></a>

* This is a `leaf` node, so the functions inside the `leaf page` implementation are needed.
* Use the buffer pool manager implemented in Lab 1 to obtain the page. Remember to `unpin` the newly created page after creating the node.
* Use binary-search-based insertion to optimize insertion.

**1. Create a new node**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::StartNewTree(const KeyType &key, const ValueType &value) {
  Page *new_page = buffer_pool_manager_->NewPage(&root_page_id_);
  if (new_page == nullptr) {
    throw "out of memory";
  }

  LeafPage *leaf_page = reinterpret_cast<LeafPage *>(new_page);
  leaf_page->Init(root_page_id_,INVALID_PAGE_ID,leaf_max_size_);

  //update root page id
  UpdateRootPageId(1);
  // Insert entry directly into leaf page.
  // For a new B+ tree, the root page is the leaf page.
  leaf_page->Insert(key, value, comparator_);
  buffer_pool_manager_->UnpinPage(root_page_id_, true);  // unpin
}
```
{% endcode %}

**2. The `Insert` function** [**#**](https://www.cnblogs.com/JayL-zxl/p/14324297.html#2-insert%E5%87%BD%E6%95%B0)

The `Insert` function can directly reuse the `KeyIndex` function from above.

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_LEAF_PAGE_TYPE::Insert(const KeyType &key, const ValueType &value, const KeyComparator &comparator) {
  // 1. boundary check
  if (GetSize() == GetMaxSize()) {
    std::cout << "leaf_page Insert: size=" << GetSize() << ", max_size=" << GetMaxSize() << std::endl;
  }
  // 2. find position
  int pos = KeyIndex(key, comparator);
  // 3. insert
  // make room
  for (int i = GetSize() - 1; i >= pos; i--) {
    array_[i + 1] = array_[i];
  }

  array_[pos] = MappingType{key, value};
  // 4. update size
  IncreaseSize(1);
  return GetSize();
}
```
{% endcode %}

#### 2. **Otherwise, find the target leaf page and insert without splitting** <a href="#2-find-target-leaf-and-insert-without-split" id="2-find-target-leaf-and-insert-without-split"></a>

1. First find the leaf page.
2. If the number of elements in the leaf page is below the maximum, insert directly.
3. Otherwise, split the page, create a new node, and promote the separator key.
4. If inserting into the parent also requires splitting, recursively split the parent; otherwise stop.

If the number of keys in the leaf page is less than `m - 1`, insert directly into the leaf page.

**1. `Lookup` function implementation**

> The `Lookup` function finds the child pointer, which is essentially a `page_id`, that should contain the input `key`.

{% code overflow="wrap" lineNumbers="true" %}
```c
INDEX_TEMPLATE_ARGUMENTS
ValueType B_PLUS_TREE_INTERNAL_PAGE_TYPE::Lookup(const KeyType &key, const KeyComparator &comparator) const {
  // start from the second entry
  for (int i = 1; i < GetSize(); i++) {
    KeyType cur_key = array_[i].first;
    if (comparator(key, cur_key) < 0) {
      return array_[i - 1].second;
    }
  }
  return array_[GetSize() - 1].second;
}
```
{% endcode %}

**2. findLeafPage**[**#**](https://www.cnblogs.com/JayL-zxl/p/14324297.html#2-findleafpage)

This function is important because it finds the `LeafPage` where insertion should happen. This version is for non-concurrent insertion and is used to test the insertion algorithm. It will be modified later for the concurrent case.

1. Start from the root of the B+Tree and keep descending until a leaf page is reached.
2. Because a B+Tree is a multi-way search tree, the downward search is driven by key comparisons.
3. The search through internal pages uses the `Lookup` function mentioned above.

{% code overflow="wrap" lineNumbers="true" %}
```cpp
// only need for inserting test
INDEX_TEMPLATE_ARGUMENTS
Page *BPLUSTREE_TYPE::FindLeafPage(const KeyType &key, bool leftMost) {
  throw Exception(ExceptionType::NOT_IMPLEMENTED, "Implement this for test");
  if (root_page_id_ == INVALID_PAGE_ID) {
    throw std::runtime_error("Unexpected. root_page_id is INVALID_PAGE_ID");
  }
  Page *page = buffer_pool_manager_->FetchPage(root_page_id_);  // now root page is pin
  BPlusTreePage *node = reinterpret_cast<BPlusTreePage *>(page);
  while (!node->IsLeafPage()) {
    InternalPage *internal_node = reinterpret_cast<InternalPage *>(node);
    page_id_t next_page_id = leftMost ? internal_node->ValueAt(0) : internal_node->Lookup(key, comparator_);
    Page *next_page = buffer_pool_manager_->FetchPage(next_page_id);  // next_level_page pinned
    BPlusTreePage *next_node = reinterpret_cast<BPlusTreePage *>(next_page);
    buffer_pool_manager_->UnpinPage(node->GetPageId(), false);  // curr_node unpinned
    page = next_page;
    node = next_node;
  }
  return page;
}

```
{% endcode %}

**3. Direct insertion without splitting**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
bool BPLUSTREE_TYPE::InsertIntoLeaf(const KeyType &key, const ValueType &value, Transaction *transaction) {
  // locking is not considered in this version
  {
    if (IsEmpty()) {
      StartNewTree(key, value);
      return true;
    }
    // [Attention] the fetched page is pinned here
    Page *right_leaf = FindLeafPage(key, false);
    LeafPage *leaf_page = reinterpret_cast<LeafPage *>(right_leaf);

    // 1. if insert key is exist
    if (leaf_page->Lookup(key, nullptr, comparator_)) {
      buffer_pool_manager_->UnpinPage(leaf_page->GetPageId(), false); // unpin page
      return false;
    }
    // 2. insert entry
    leaf_page->Insert(key, value, comparator_);
    // the split case is analyzed below
```
{% endcode %}

#### **3. Split case** <a href="#3-split-case" id="3-split-case"></a>

The main `InsertIntoLeaf` flow continues from the previous code.

{% code overflow="wrap" lineNumbers="true" %}
```cpp
// 2.1 need to split
    // split need to do two things
    // 1. create new page copy [mid, r] to new page
    // 2. handle recursively if necessary
    if (leaf_page->GetSize() == maxSize(leaf_page) + 1) {
      LeafPage *new_leaf = Split(leaf_page);
      InsertIntoParent(leaf_page, new_leaf->KeyAt(0), new_leaf, transaction);
    }
    // 2.2 no need to split
    // must unpin right leaf page
    buffer_pool_manager_->UnpinPage(right_leaf->GetPageId(), true);
    return true;
```
{% endcode %}

**1. Call `Split` to split the leaf page**

1. Splitting creates a new node containing `m - m / 2` keys. Remember to link the two leaf pages.
2. The `Split` function must distinguish leaf pages from internal pages because leaf pages need to update the linked-list pointers.

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
N *BPLUSTREE_TYPE::Split(N *node) {
  // 1, ask a new page
  page_id_t new_page_id;
  Page *new_page = buffer_pool_manager_->NewPage(&new_page_id);  // pinned
  if (new_page == nullptr) {
    throw std::string("out of memory");
  }
  N *new_node;
  if (node->IsLeafPage()) {
    LeafPage *leaf_node = reinterpret_cast<LeafPage *>(node);
    LeafPage *new_leaf_node = reinterpret_cast<LeafPage *>(new_page);
    new_leaf_node->Init(new_page_id, leaf_node->GetParentPageId(),leaf_max_size_);
    leaf_node->MoveHalfTo(new_leaf_node);
    // update the leaf-level linked list
    new_leaf_node->SetNextPageId(leaf_node->GetNextPageId());
    leaf_node->SetNextPageId(new_leaf_node->GetPageId());
    new_node = reinterpret_cast<N *>(new_leaf_node);
  } else {
    // internal pages do not need linked-list pointers
    InternalPage *internal_node = reinterpret_cast<InternalPage *>(node);
    InternalPage *new_internal_node = reinterpret_cast<InternalPage *>(new_page);
    new_internal_node->Init(new_page_id, internal_node->GetParentPageId(),internal_max_size_);
    internal_node->MoveHalfTo(new_internal_node,buffer_pool_manager_);
    new_node = reinterpret_cast<N *>(new_internal_node);
  }
  return new_node;
}
```
{% endcode %}

This involves the `MoveHalfTo` function. A simple version is shown below.

{% code overflow="wrap" lineNumbers="true" %}
```
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::MoveHalfTo(BPlusTreeLeafPage *recipient) {
  // Either side can hold the extra entry; the textbook keeps more on the left, while this version keeps more on the right.
  int moved_num = GetSize() - GetSize() / 2;
  int start = GetSize() - moved_num;
  CopyNFrom(array_ + start, moved_num);
  IncreaseSize(-1 * moved_num);
  recipient->IncreaseSize(moved_num);
}
```
{% endcode %}

**2. `InsertIntoParent` function implementation**

Before implementing this function, first look at the algorithm from the textbook.

<figure><img src="../../../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

1. If `old_node` is the root, create a new node `R` as the new root. Use `key` as the separator key in the new root. Then update the parent pointers of `old_node` and `new_node`, as well as the root's child pointers.

{% code overflow="wrap" lineNumbers="true" %}
```cpp
 // 1. old_node is root
  if (old_node->IsRootPage()) {
    page_id_t new_root_pid;
    Page *page = buffer_pool_manager_->NewPage(&new_root_pid);
    InternalPage *new_root_page = reinterpret_cast<InternalPage *>(page->GetData());

    // new root page init
    new_root_page->Init(new_root_pid, INVALID_PAGE_ID, internal_max_size_);
    UpdateRootPageId(0);  // update not insert
    root_page_id_ = new_root_pid;
    // set parent page id
    new_root_page->PopulateNewRoot(old_node->GetPageId(), key, new_node->GetPageId());
    old_node->SetParentPageId(new_root_pid);
    new_node->SetParentPageId(new_root_pid);
    buffer_pool_manager_->UnpinPage(new_root_pid, true);
  }
```
{% endcode %}

1. Find the parent of the split leaf page and then check its state.

a. If the separator can be inserted directly, insert it directly.

b. Otherwise, split the parent as well, which means making a recursive call.

{% code overflow="wrap" lineNumbers="true" %}
```cpp
else {
    page_id_t parent_pid = old_node->GetParentPageId();
    Page *parent_page = buffer_pool_manager_->FetchPage(parent_pid);  // parent_page pined
    InternalPage *parent_node = reinterpret_cast<InternalPage *>(parent_page->GetData());

    parent_node->InsertNodeAfter(old_node->GetPageId(), key, new_node->GetPageId());

    if (parent_node->GetSize() == maxSize(parent_node) + 1) {
      // need to split
      InternalPage *new_parent_node = Split(parent_node);  // new_parent_node pined
      InsertIntoParent(parent_node, new_parent_node->KeyAt(0), new_parent_node);

      buffer_pool_manager_->UnpinPage(new_parent_node->GetPageId(), true);  // unpin new_parent_node
    }
    buffer_pool_manager_->UnpinPage(parent_node->GetPageId(), true);  // unpin parent_node
  }
```
{% endcode %}

At this point, the first part of the test passes.

<figure><img src="../../../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

The following screenshot shows the first part passing.\
If we insert `1`, `2`, `3`, `4`, and `5`, the program produces the result below:

<figure><img src="../../../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

The result is correct.

### 3. Some Details <a href="#3-some-details" id="3-some-details"></a>

#### 1. Difference Between Internal Pages and Leaf Pages <a href="#1-difference-between-internal-pages-and-leaf-pages" id="1-difference-between-internal-pages-and-leaf-pages"></a>

**1.1 Different sizes**

* The maximum number of entries in an internal page is one greater than that of a leaf page.
* For example, if `m = 3`, an internal page can contain three entries, while a leaf page can contain at most two. However, `array[0]` in the internal page is effectively used to store a child pointer; its key is not shown in the draw result.

**1.2 Different behavior during `Split`**

* When splitting a leaf page, the leaf-level linked list must be maintained.
* Internal pages do not need that linked-list maintenance.
* The shared steps are: obtain a new page -> reinterpret the page type -> call `MoveHalfTo`.

#### 2. Notes on Pin and Unpin <a href="#2-notes-on-pin-and-unpin" id="2-notes-on-pin-and-unpin"></a>

* When you obtain a page through `FetchPage`, the page is pinned.
* Remember to `unpin` the page after using it. This is important.

#### 3. Debugging Tips <a href="#3-debugging-tips" id="3-debugging-tips"></a>

* Use the [visualization website](https://www.cs.usfca.edu/\~galles/visualization/BPlusTree.html) together with the provided `b_plus_print_test`. Print the input graph as `xxx.dot`, copy its contents into [GraphvizOnline](http://dreampuf.github.io/GraphvizOnline/), and compare the result.
* On `Mac`, CLion can be used to debug the test files directly. `lldb` is especially useful here.

#### 4. Meaning of `maxSize` <a href="#4-meaning-of-maxsize" id="4-meaning-of-maxsize"></a>

Pay attention to the values supplied during B+Tree initialization.

`internal_max_size` can be understood as the **number of pointers**. Suppose we have a B+Tree with `m = 3`.

`internal_max_size = 3` allows three internal entries in this implementation context, while `leaf_max_size = 2` means the leaf can effectively contain fewer keys before splitting under the helper logic below. This was discovered from failing tests.

Therefore, `maxSize()` can be implemented like this:

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
int BPLUSTREE_TYPE::maxSize(N *node) {
  return node->IsLeafPage() ? leaf_max_size_ - 1 : internal_max_size_;
}
```
{% endcode %}
