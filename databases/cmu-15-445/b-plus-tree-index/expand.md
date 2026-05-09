---
description: Project 2
---

# 😂 Expand

### 1. 实验介绍 <a href="#1-shi-yan-jie-shao" id="1-shi-yan-jie-shao"></a>

> 第一个打分点---实现b+树的基本结构、插入、搜索操作
>
> 注意这里没有考虑打分点2的并发问题，所以对于加锁、解锁和事物都没有考虑。

* [**Task #1 - B+Tree Pages**](https://15445.courses.cs.cmu.edu/fall2020/project2/#b+tree-pages)
* [**Task #2.a - B+Tree Data Structure (Insertion & Point Search)**](https://15445.courses.cs.cmu.edu/fall2020/project2/#b+tree-structure-1)

> 第二个打分点--实现b+树的删除操作、索引迭代器和对并发访问的支持

* [**Task #2.b - B+Tree Data Structure (Deletion)**](https://15445.courses.cs.cmu.edu/fall2020/project2/#b+tree-structure-2)
* [**Task #3 - Index Iterator**](https://15445.courses.cs.cmu.edu/fall2020/project2/#index-iterator)
* [**Task #4 - Concurrent Index**](https://15445.courses.cs.cmu.edu/fall2020/project2/#concurrent\_index)

### Task 1 B+TREE PAGES[#](https://www.cnblogs.com/JayL-zxl/p/14324297.html#task-1-btree-pages) <a href="#task-1-btree-pages" id="task-1-btree-pages"></a>

您需要实现三个页面类来存储B+树的数据。

* B+ Tree Parent Page
* B+ Tree Internal Page
* B+ Tree Leaf Page

#### <mark style="color:purple;">1.</mark>  B+ Tree Parent Page <a href="#1-b-tree-parent-page" id="1-b-tree-parent-page"></a>

这是内部页和叶页都继承的父类，它只包含两个子类共享的信息。父页面被划分为如下表所示的几个字段。\
`*`_`B+Tree Parent Page Content`_

| Variable Name      | Size | Description                             |
| ------------------ | ---- | --------------------------------------- |
| page\_type\_       | 4    | Page Type (internal or leaf)            |
| lsn\_              | 4    | Log sequence number (Used in Project 4) |
| size\_             | 4    | Number of Key & Value pairs in page     |
| max\_size\_        | 4    | Max number of Key & Value pairs in page |
| parent\_page\_id\_ | 4    | Parent Page Id                          |
| page\_id\_         | 4    | Self Page Id                            |

必须在指定的文件中实现父页。只能修改头文件(`src/include/storage/page/b_plus_tree_page.h`) 和其对应的源文件 (`src/storage/page/b_plus_tree_page.cpp`).

这里都是一些简单的set、get就不写出来了

* `IsRootPage` 函数根据 `parent_id_` 是否是 `INVALID_PAGE_ID` 返回 `true` 或者 `false`
* `GetMinSize()`：需要区分 **1**）根结点且为叶子结点  **2**）根结点  **3**）其他

#### <mark style="color:purple;">2.</mark> B+TREE INTERNAL PAGE <a href="#2-btree-internal-page" id="2-btree-internal-page"></a>

内部页不存储任何实际数据，而是存储有序的m个键条目和m + 1个指针（也称为page\_id）。&#x20;

由于指针的数量不等于键的数量，因此将第一个键设置为无效，并且查找方法应始终从第二个键开始。&#x20;

任何时候，每个内部页面至少有一半已满。 在删除期间，可以将两个半满页面合并为合法页面，或者可以将其重新分配以避免合并，而在插入期间，可以将一个完整页面分为两部分。

你只能修改头文件(`src/include/storage/page/b_plus_tree_internal_page.h`) 和对应的源文件(`src/page/b_plus_tree_internal_page.cpp`).

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

叶子页存储有序的m个键条目(key)和m个值条目(value)。&#x20;

实现中，值只能是用于定位实际元组存储位置的64位`record_id`，请参阅`src / include / common / rid.h`中定义的`RID`类。&#x20;

叶子页与内部页在键/值对的数量上具有相同的限制，并且应该遵循相同的合并，重新分配和拆分操作。

必须在指定的文件中实现内部页。 仅允许修改头文件`（src / include / storage / page / b_plus_tree_leaf_page.h`）及其相应的源文件`（src / storage / page / b_plus_tree_leaf_page.cpp`）。

<mark style="color:red;">**‼️**</mark>** 重要的KeyIndex函数**

这个函数可以返回第一个>=当前key值的编号。这个在插入的时候经常会用到，这样就可以让代码重复利用，在`LevelDB`中也有类似的操作。

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_LEAF_PAGE_TYPE::KeyIndex(const KeyType &key, const KeyComparator &comparator) const {
  // 二分查找
  // TODO : 换成std::lower_bound()好看一点
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

重要提示：尽管叶子页和内部页包含相同类型的键，但它们可能具有不同类型的值，因此叶子页和内部页的最大大小可能不同。

每个`B + Tree`叶子/内部页面对应从缓冲池获取的存储页面的内容（即data\_部分）。 因此，每次尝试读取或写入叶子/内部页面时，都需要首先使用其唯一的`page_id`从缓冲池中提取页面，然后将其重新解释为叶子或内部页面，并在写入或删除后执行`unpin`操作。

### Task 2.A - B+TREE DATA STRUCTURE (INSERTION & POINT SEARCH) <a href="#task-2a---btree-data-structure-insertion--point-search" id="task-2a---btree-data-structure-insertion--point-search"></a>

> 其实就是实现`b_plus_tree.cpp/InsertIntoLeaf`函数所涉及到的相关函数。

这里 B +树索引只能支持唯一键。 也就是说，当您尝试将具有重复键的键值对插入索引时，它应该返回`false`。

对于`checkpoint1`，仅需要B + Tree索引支持插入（`Insert`）和点搜索（`GetValue`）。 我们不需要实现删除操作。&#x20;

插入后如果当前键/值对的数量等于`max_size`，则应该正确执行分割。 由于任何写操作都可能导致B + Tree索引中的`root_page_id`发生更改，因此要更新（`src / include / storage / page / header_page.h`）中的`root_page_id`，以确保索引在磁盘上具有持久性 。 在`BPlusTree`类中，我们已经为您实现了一个名为`UpdateRootPageId`的函数。 您需要做的就是在B + Tree索引的`root_page_id`更改时调用此函数。

您的B + Tree实现必须隐藏key/value等的详细信息，建议使用如下结构：

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

这些类别已经为你实现了

* `KeyType`: The type of each key in the index. This will only be `GenericKey`, the actual size of `GenericKey` is specified and instantiated with a template argument and depends on the data type of indexed attribute.
* `ValueType`: The type of each value in the index. This will only be 64-bit RID.
* `KeyComparator`: The class used to compare whether two `KeyType` instances are less/greater-than each other. These will be included in the `KeyType` implementation files.

1. 你必须使用传入的transaction，把已经加锁的页面保存起来。
2. 我们提供了读写锁存器的实现（`src / include / common / rwlatch.h`）。 并且已经在页面头文件下添加了辅助函数来获取和释放Latch锁（`src / include / storage / page / page.h`）。

首先附上书上的b+树插入算法

<figure><img src="../../../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

**对上面几种情况的分析**

#### **1. 如果当前为空树则创建一个叶子结点并且也是根节点** <a href="#1-ru-guo-dang-qian-wei-kong-shu-ze-chuang-jian-yi-ge-ye-zi-jie-dian-bing-qie-ye-shi-gen-jie-dian" id="1-ru-guo-dang-qian-wei-kong-shu-ze-chuang-jian-yi-ge-ye-zi-jie-dian-bing-qie-ye-shi-gen-jie-dian"></a>

* 这里是`leaf`结点所以这里需要用到`leaf page`内的函数
* 注意这里需要用lab1实现的buffer池管理器来获得page。 这里记得创建完新的结点之后要unpin
* 进行插入的时候用二分插入来进行优化

**1. 创建新结点**

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

**2. `insert`函数**[**#**](https://www.cnblogs.com/JayL-zxl/p/14324297.html#2-insert%E5%87%BD%E6%95%B0)

这里的`insert`函数可以直接用之前的`KeyIndex`函数

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_LEAF_PAGE_TYPE::Insert(const KeyType &key, const ValueType &value, const KeyComparator &comparator) {
  // 1. 边界判断
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

#### 2. **否则寻找到插入元素应该在的叶子结点**，并插入(不分裂) <a href="#2-fou-ze-xun-zhao-dao-cha-ru-yuan-su-ying-gai-zai-de-ye-zi-jie-dian-bing-cha-ru-bu-fen-lie" id="2-fou-ze-xun-zhao-dao-cha-ru-yuan-su-ying-gai-zai-de-ye-zi-jie-dian-bing-cha-ru-bu-fen-lie"></a>

1. 首先找到叶子结点
2. 如果叶子结点内的元素个数小于最大值则直接插入
3. 否则需要进行分裂。产生两个新的结点。把元素上提
4. 如果提到父亲结点，父结点仍需要分裂。则递归进行分裂否则结束

如果叶子结点内的关键字小于m-1,则直接插入到叶子结点

**1. LookUp函数实现**

> Lookup函数用来寻找包含输入"key"的children pointer(其实就是page\_id)

{% code overflow="wrap" lineNumbers="true" %}
```c
INDEX_TEMPLATE_ARGUMENTS
ValueType B_PLUS_TREE_INTERNAL_PAGE_TYPE::Lookup(const KeyType &key, const KeyComparator &comparator) const {
  // 从第二个节点开始
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

由于要找到应该插入的`LeafPage`所以这个函数狠狠重要。但是这里是非并发下插入，在这里用`findLeafPage`进行对插入算法的测试。后面对于并发情况会有所修改。

1. 从整个b+树的根节点开始。一直向下找到叶子结点
2. 因为b+树是多路搜索树，所以整个向下搜索就是通过key值进行比较
3. 其中内部结点向下搜索的过程利用了上面提到的`lookup`函数

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

**3. 无分裂直接插入**

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
bool BPLUSTREE_TYPE::InsertIntoLeaf(const KeyType &key, const ValueType &value, Transaction *transaction) {
  // 不考虑锁的实现
  {
    if (IsEmpty()) {
      StartNewTree(key, value);
      return true;
    }
    //[Attention] 这里获取到page是pined
    Page *right_leaf = FindLeafPage(key, false);
    LeafPage *leaf_page = reinterpret_cast<LeafPage *>(right_leaf);

    // 1. if insert key is exist
    if (leaf_page->Lookup(key, nullptr, comparator_)) {
      buffer_pool_manager_->UnpinPage(leaf_page->GetPageId(), false); // unpined page
      return false;
    }
    // 2. insert entry
    leaf_page->Insert(key, value, comparator_);
    // 下面分析需要分裂的情况
```
{% endcode %}

#### **3. 分裂的情况** <a href="#3-fen-lie-de-qing-kuang" id="3-fen-lie-de-qing-kuang"></a>

`InsertLeaf主函数`接上文。

{% code overflow="wrap" lineNumbers="true" %}
```cpp
// 2.1 need to split
    // split need to do two things
    // 1. create new page copy [mid, r] to new page
    // 2. if necessary 递归处理
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

**1. 调用`split`函数对叶子结点进行分割**

1. split的时候会产生一个含有m-m/2个关键字的新结点。注意把两个叶子结点连接起来。
2. 这里注意split函数要区分叶子结点和内部结点。因为叶子结点需要更新双向链表

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
    // 叶子节点更新双向链表
    new_leaf_node->SetNextPageId(leaf_node->GetNextPageId());
    leaf_node->SetNextPageId(new_leaf_node->GetPageId());
    new_node = reinterpret_cast<N *>(new_leaf_node);
  } else {
    // 内部节点不需要设置双向链表
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

这里涉及到了`MoveHalfTo`函数简单的附上一下，这个非常简单

{% code overflow="wrap" lineNumbers="true" %}
```
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::MoveHalfTo(BPlusTreeLeafPage *recipient) {
  // 这里好像是说 你左边多还是右边多都行，书上是左边多，我个人习惯右边多
  int moved_num = GetSize() - GetSize() / 2;
  int start = GetSize() - moved_num;
  CopyNFrom(array_ + start, moved_num);
  IncreaseSize(-1 * moved_num);
  recipient->IncreaseSize(moved_num);
}
```
{% endcode %}

**2. `InsertIntoParent`函数实现**

这个函数的实现先看一下书上给出的算法

<figure><img src="../../../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

1. 如果`old_node`就是根节点，那么就要创建一个新的节点R当作根节点。然后取`key`的值当作根节点的值。修改`old_node`和`new_node`的父指针。以及根节点的孩子指针

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

1. 找到分裂的叶子结点的父亲节点随后进行判断

a. 如果可以直接插入则直接插入

b. 否则需要对父结点在进行分裂，即递归调用。

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

好了第一部分的测试就通过了

<figure><img src="../../../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

附上一个pass的截图完成第一部分✅\
如果我们插入1、2、3、4、5那么我们用程序得到的结果如下：

<figure><img src="../../../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

可以发现是完全正确的 🌟

### 3. ⚠️一些细节 <a href="#3-yi-xie-xi-jie" id="3-yi-xie-xi-jie"></a>

#### 1. 关于内部结点和叶子结点的区别 <a href="#1-guan-yu-nei-bu-jie-dian-he-ye-zi-jie-dian-de-qu-bie" id="1-guan-yu-nei-bu-jie-dian-he-ye-zi-jie-dian-de-qu-bie"></a>

**1.1 大小不一样**

* 内部结点的最大结点个数是比叶子结点多一
* 例如m = 3, 那么内部结点的个数就可以是3。而叶子结点则最多是2，但是内部结点的array\[0]实际上就是个存地址的。它的key在我们的Draw结果图中都不显示。

**1.2 在`Split`的时候有区别**

* 在叶子结点split的时候需要进行双向链表的维护
* 而在内部结点则不需要
* 共有操作都是获得一个新页--> 类型转换 ---> MoveHalfTo

#### 2. upin的pin的注意事项 <a href="#2upin-de-pin-de-zhu-yi-shi-xiang" id="2upin-de-pin-de-zhu-yi-shi-xiang"></a>

* 当你利用`FetchPage`拿到一个page的时候他就是pined
* 当你使用完之后记得要`unpin`这很重要

#### 3. debug的一些小技巧 <a href="#3debug-de-yi-xie-xiao-ji-qiao" id="3debug-de-yi-xie-xiao-ji-qiao"></a>

* 利用[可视化网站](https://www.cs.usfca.edu/\~galles/visualization/BPlusTree.html)和代码中给的`b_plus_print_test`这个测试，把输入图打印成`xxx.dot`然后复制里面的内容在 [GraphvizOnline](http://dreampuf.github.io/GraphvizOnline/) 显示进行对比。
* 对于`Mac`系统利用Clion可以直接对测试文件debug。还是非常爽的。其中`lldb`的利用非常重要。

#### 4. maxSize的含义 <a href="#4maxsize-de-han-yi" id="4maxsize-de-han-yi"></a>

这里要注意在进行B+树初始化时候给的

`internal_max_size`可以认为指的是**指针数**。也就是说假设我们有m = 3的b+树

inter\_max\_size = 3 是可以有三个key。但是 leaf\_max\_size = 2 就只能包含一个key。这个在测试用例被卡才发现的。

所以`maxSize()`函数可以这样实现

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
int BPLUSTREE_TYPE::maxSize(N *node) {
  return node->IsLeafPage() ? leaf_max_size_ - 1 : internal_max_size_;
}
```
{% endcode %}
