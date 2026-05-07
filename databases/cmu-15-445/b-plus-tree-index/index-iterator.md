# Index Iterator

# 😂 Expand3: Index\_Iterator

这里就是需要实现迭代器的一些操作,比如begin、end、isend等等

下面是对于`IndexIterator`的构造函数

其中idx表示当前page中的第几个tuple

{% code lineNumbers="true" %}
```cpp
INDEXITERATOR_TYPE::IndexIterator(LeafPage *leftmost_leaf, int idx, BufferPoolManager *buffer_pool_manager)
    : curr_page(leftmost_leaf), curr_index(idx), bpm(buffer_pool_manager) {}
```
{% endcode %}

### 1. 首先我们来看begin函数的实现[#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#1-%E9%A6%96%E5%85%88%E6%88%91%E4%BB%AC%E6%9D%A5%E7%9C%8Bbegin%E5%87%BD%E6%95%B0%E7%9A%84%E5%AE%9E%E7%8E%B0)[#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#1-%E9%A6%96%E5%85%88%E6%88%91%E4%BB%AC%E6%9D%A5%E7%9C%8Bbegin%E5%87%BD%E6%95%B0%E7%9A%84%E5%AE%9E%E7%8E%B0)

1. 利用key值找到叶子结点
2. 然后获取当前key值的index就是begin的位置

{% code lineNumbers="true" %}
```cpp
Page *page = FindLeafPage(KeyType{}, true);  // leftmost_leaf pinned
  LeafPage *leftmost_leaf = reinterpret_cast<LeafPage *>(page->GetData());
  buffer_pool_manager_->UnpinPage(leftmost_leaf->GetPageId(), false);
  return INDEXITERATOR_TYPE(leftmost_leaf, 0, buffer_pool_manager_);
```
{% endcode %}

### 2. end函数的实现[#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#2-end%E5%87%BD%E6%95%B0%E7%9A%84%E5%AE%9E%E7%8E%B0)[#](https://www.cnblogs.com/JayL-zxl/p/14333395.html#2-end%E5%87%BD%E6%95%B0%E7%9A%84%E5%AE%9E%E7%8E%B0)

1. 找到最开始的结点
2. 然后一直向后遍历直到`nextPageId=-1`结束
3. 这里注意需要重载`!=`和`==`

`end`函数

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

`==和 !=`函数

这里注意在！= 那里不能写成itr != \*this

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

### 3. 重载++和\*(解引用符号)

1. 重载++

> 简单的index++然后设置nextPageId即可

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

1. 重载\*

> return array\[index]即可

{% code overflow="wrap" %}
```cpp
const MappingType &INDEXITERATOR_TYPE::operator*() { 
    return curr_page->GetItem(curr_index); 
}
```
{% endcode %}
