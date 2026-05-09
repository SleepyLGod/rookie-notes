# Index Iterator

# Expand3: Index Iterator

This part implements iterator operations such as `begin`, `end`, and `isEnd`.

The following is the constructor of `IndexIterator`.

Here, `idx` indicates which tuple in the current page the iterator points to.

{% code lineNumbers="true" %}
```cpp
INDEXITERATOR_TYPE::IndexIterator(LeafPage *leftmost_leaf, int idx, BufferPoolManager *buffer_pool_manager)
    : curr_page(leftmost_leaf), curr_index(idx), bpm(buffer_pool_manager) {}
```
{% endcode %}

### 1. First, look at the implementation of begin [#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#1-%E9%A6%96%E5%85%88%E6%88%91%E4%BB%AC%E6%9D%A5%E7%9C%8Bbegin%E5%87%BD%E6%95%B0%E7%9A%84%E5%AE%9E%E7%8E%B0)[#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#1-%E9%A6%96%E5%85%88%E6%88%91%E4%BB%AC%E6%9D%A5%E7%9C%8Bbegin%E5%87%BD%E6%95%B0%E7%9A%84%E5%AE%9E%E7%8E%B0)

1. Use the key value to find the leaf node.
2. Then get the index of the current key value; this index is the `begin` position.

{% code lineNumbers="true" %}
```cpp
Page *page = FindLeafPage(KeyType{}, true);  // leftmost_leaf pinned
  LeafPage *leftmost_leaf = reinterpret_cast<LeafPage *>(page->GetData());
  buffer_pool_manager_->UnpinPage(leftmost_leaf->GetPageId(), false);
  return INDEXITERATOR_TYPE(leftmost_leaf, 0, buffer_pool_manager_);
```
{% endcode %}

### 2. Implementation of end [#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#2-end%E5%87%BD%E6%95%B0%E7%9A%84%E5%AE%9E%E7%8E%B0)[#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#2-end%E5%87%BD%E6%95%B0%E7%9A%84%E5%AE%9E%E7%8E%B0)

1. Find the starting node.
2. Then keep traversing forward until `nextPageId = -1`.
3. Note that `!=` and `==` need to be overloaded.

The `end` function:

{% code lineNumbers="true" %}
```cpp
// find the right most page
  Page *page = FindLeafPage(KeyType{}, true);  // page pinned
  LeafPage *leaf = reinterpret_cast<LeafPage *>(page->GetData());

  while (leaf->GetNextPageId() != INVALID_PAGE_ID) {
    page_id_t next_page_id = leaf->GetNextPageId();
    buffer_pool_manager_->UnpinPage(leaf->GetPageId(), false);  // page unpinned

    Page *next_page = buffer_pool_manager_->FetchPage(next_page_id);  // next_page pinned
    leaf = reinterpret_cast<LeafPage *>(next_page->GetData());
  }

  return INDEXITERATOR_TYPE(leaf, leaf->GetSize(), buffer_pool_manager_);
```
{% endcode %}

The `==` and `!=` functions:

For `!=`, be careful not to write `itr != *this`.

{% code lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
bool INDEXITERATOR_TYPE::operator==(const IndexIterator &itr) const {
  return itr.curr_page == curr_page && itr.curr_index == curr_index;
}

INDEX_TEMPLATE_ARGUMENTS
bool INDEXITERATOR_TYPE::operator!=(const IndexIterator &itr) const { return !(itr == *this); }
```
{% endcode %}

### 3. Overload `++` and `*` (dereference)

1. Overload `++`.

> Simply increment `index`, then set `nextPageId` as needed.

{% code lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
INDEXITERATOR_TYPE &INDEXITERATOR_TYPE::operator++() {
  curr_index++;
  if (curr_index == curr_page->GetSize() && curr_page->GetNextPageId() != INVALID_PAGE_ID) {
    page_id_t next_pid = curr_page->GetNextPageId();
    Page *next_page = bpm->FetchPage(next_pid);  // pined page

    LeafPage *next_node = reinterpret_cast<LeafPage *>(next_page->GetData());
    curr_page = next_node;
    bpm->UnpinPage(next_page->GetPageId(), false);
    curr_index = 0;
  }
  return *this;
}
```
{% endcode %}

1. Overload `*`.

> Just return `array[index]`.

{% code overflow="wrap" %}
```cpp
const MappingType &INDEXITERATOR_TYPE::operator*() { 
    return curr_page->GetItem(curr_index); 
}
```
{% endcode %}
