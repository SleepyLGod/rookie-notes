# Expand2: Delete

## 1. Deletion algorithm

CMU provides this [visualization site](https://www.cs.usfca.edu/\~galles/visualization/BPlusTree.html).

For a clearer explanation of the overall deletion algorithm, this [**slide deck**](http://courses.cms.caltech.edu/cs122/lectures-wi2018/CS122Lec11.pdf) is useful.

The algorithm description is shown in the table below:

![image-20210121112759338](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126135520429-1776802491.png)

And in the following figure:

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210928222239089-724076668.png)

## 2. Deletion implementation

### 2.1 Step 1: find the leaf node containing the target key and delete it

1. Find the leaf node containing the target key.
2. If the current tree is empty, return immediately.
3. Otherwise, first find the page where the key to be deleted is located.
4. Then call `RemoveAndDeleteRecord` on the leaf page to delete the key directly.

{% code lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::Remove(const KeyType &key, Transaction *transaction) {
  {
    // This is specifically for non-concurrent delete.
    Page *page = FindLeafPage(key, false);
    LeafPage *leafPage = reinterpret_cast<LeafPage*>(page->GetData()); // pinned leafPage

    leafPage->RemoveAndDeleteRecord(key,comparator_);
```
{% endcode %}

1. `RemoveAndDeleteRecord` function.
2. At this point the implementation increasingly feels similar to how LevelDB works.
3. Using `KeyIndex` makes this implementation straightforward.

{% code lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_LEAF_PAGE_TYPE::RemoveAndDeleteRecord(const KeyType &key, const KeyComparator &comparator) {
  int pos = KeyIndex(key, comparator);
  if (pos < GetSize() && comparator(array_[pos].first, key) == 0) {
    for (int j = pos + 1; j < GetSize(); j++) {
      array_[j - 1] = array_[j];
    }
    IncreaseSize(-1);
  }
  return GetSize();
}
```
{% endcode %}

After deletion, the main follow-up handling has two cases: merge and redistribute. The detailed flow is shown below.

### **2.2 Coalesce implementation flow**

If the number of keys in the leaf node is below the minimum, continue downward and call the core deletion helper `CoalesceAndRedistribute`.

**The following is the logic of CoalesceAndRedistribute.**

#### 1. If the current node is the root, call `AdjustRoot(node)`

> The project hint effectively says that this function handles only two cases:
>
> * Case 1: `old_root_node` is an internal node and its size is 1, meaning the internal node no longer has any real key. Its child should be promoted to become the new root.
> * Case 2: `old_root_node` is a leaf node and its size is 0. In this case, delete it directly.
>
> Otherwise, no page needs to be deleted, so return `false` directly.

{% code lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
bool BPLUSTREE_TYPE::AdjustRoot(BPlusTreePage *old_root_node) {
  // case 1 old_root_node (internal node) has only one size
  if (!old_root_node->IsLeafPage() && old_root_node->GetSize() == 1) {
    InternalPage *old_root_page = reinterpret_cast<InternalPage *>(old_root_node);
    page_id_t new_root_page_id = old_root_page->adjustRootForInternal();
    root_page_id_ = new_root_page_id;
    UpdateRootPageId(0);

    Page *new_root_page = buffer_pool_manager_->FetchPage(new_root_page_id);
    BPlusTreePage *new_root = reinterpret_cast<BPlusTreePage *>(new_root_page->GetData());
    new_root->SetParentPageId(INVALID_PAGE_ID);
    buffer_pool_manager_->UnpinPage(new_root_page_id, true);

    return true;
  }
  // case 2 : all elements deleted from the B+ tree
  if (old_root_node->IsLeafPage() && old_root_node->GetSize() == 0) {
    root_page_id_ = INVALID_PAGE_ID;
    UpdateRootPageId(0);
    return true;
  }
  return false;
}
```
{% endcode %}

#### 2. Otherwise, first decide whether coalesce is needed

> Here we need to find a sibling node and merge if the merge condition is satisfied.

**1. Decide whether the merge condition is satisfied**

Use a helper function for this check. If the combined size does not exceed the maximum `size`, the nodes can be merged.

{% code lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
bool BPLUSTREE_TYPE::IsCoalesce(N *nodeL, N *nodeR) {
  // maxSize consider InternalPage and LeafPage
  return nodeL->GetSize() + nodeR->GetSize() <= maxSize(nodeL);
}
```
{% endcode %}

The merge function merges with the immediate predecessor, meaning the node on the left.

**2. Check whether the left page can be merged**

{% code lineNumbers="true" %}
```cpp
  // firstly find from left
    if (left_sib_index >= 0) {
      page_id_t sibling_pid = parent->ValueAt(left_sib_index);
      Page *sibling_page = buffer_pool_manager_->FetchPage(sibling_pid);  // pinned sibling_page
      N *sib_node = reinterpret_cast<N *>(sibling_page->GetData());
      if (IsCoalesce(node, sib_node)) {
        // coalesce
        // unpin node and deleted node
        bool del_parent = Coalesce(&sib_node, &node, &parent, cur_index, transaction);
         // unpin sibling page
        buffer_pool_manager_->UnpinPage(sibling_pid, true);     
        // unpin parent page;
        buffer_pool_manager_->UnpinPage(parent_pid, true);                             
        if (del_parent) {
          buffer_pool_manager_->DeletePage(parent_pid);
        }
        return false;  // node is merged into sib, and already deleted
      }
      buffer_pool_manager_->UnpinPage(sibling_pid, false);  // unpin sibling page
    }
```
{% endcode %}

**3. Otherwise, check whether the right page can be merged**

{% code lineNumbers="true" %}
```cpp
// secondly find from right

    if (right_sib_index < parent->GetSize()) {
      page_id_t sibling_pid = parent->ValueAt(right_sib_index);
      Page *sibling_page = buffer_pool_manager_->FetchPage(sibling_pid);  // pinned sibling_page
      N *sib_node = reinterpret_cast<N *>(sibling_page->GetData());
      if (IsCoalesce(node, sib_node)) {
        // coalesce
        // unpin right sib and deleted right sib
        bool del_parent =
            Coalesce(&node, &sib_node, &parent, cur_index, transaction);  
        buffer_pool_manager_->UnpinPage(node->GetPageId(), true);         // unpin sibling page
        buffer_pool_manager_->UnpinPage(parent_pid, true);                // unpin parent page;
        if (del_parent) {
          buffer_pool_manager_->DeletePage(parent_pid);
        }
        return false;  // node is merged into sib, and already deleted
      }
      buffer_pool_manager_->UnpinPage(sibling_pid, false);  // unpin sibling page
    }
```
{% endcode %}

**4. Implementation of the Coalesce function**

Before implementing it, look at two figures.

<figure><img src="https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210928222258918-1565242168.png" alt=""><figcaption></figcaption></figure>

After a merge, the parent node must be updated. The move operation invalidates the previous parent pointers, so the parent has to be adjusted. This also involves the case where the parent itself may need to be deleted.

The specific case is shown below:

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210928222312846-145728912.png)

The merge of leaf nodes may cause the parent node to become a `null` or empty node, or make it fail the minimum-size requirement. In that case, the parent node also needs to be processed. The parent may itself be merged, and the original parent then disappears. The result after merging is shown below:

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210928222319763-730392232.png)

The key is to handle this recursively.

{% code lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
bool BPLUSTREE_TYPE::Coalesce(N **neighbor_node, N **node, BPlusTreeInternalPage<KeyType, 
                              page_id_t, KeyComparator> **parent, int index, Transaction *transaction) {
  // Assume that *neighbor_node is the left sibling of *node

  // Move entries from node to neighbor_node
  if ((*node)->IsLeafPage()) {
    LeafPage *leaf_node = reinterpret_cast<LeafPage *>(*node);
    LeafPage *leaf_neighbor = reinterpret_cast<LeafPage *>(*neighbor_node);

    leaf_node->MoveAllTo(leaf_neighbor);
    leaf_neighbor->SetNextPageId(leaf_node->GetNextPageId());
  } else {
    InternalPage *internal_node = reinterpret_cast<InternalPage *>(*node);
    InternalPage *internal_neighbor = reinterpret_cast<InternalPage *>(*neighbor_node);

    internal_node->MoveAllTo(internal_neighbor, (*parent)->KeyAt(index), buffer_pool_manager_);
  }

  buffer_pool_manager_->UnpinPage((*node)->GetPageId(), true);
  buffer_pool_manager_->DeletePage((*node)->GetPageId());

  (*parent)->Remove(index);

  // If parent is underfull, recursive operation
  if ((*parent)->GetSize() < minSize(*parent)) {
    return CoalesceOrRedistribute(*parent, transaction);
  }
  return false;
}
```
{% endcode %}

**5. The two MoveAllTo functions for leaf nodes and internal nodes**

I do not include the code for this function here. Its implementation is relatively simple. The only thing to pay attention to is the internal-node merge: the child pointer associated with the internal node being deleted must also be transferred. That means the implementation needs a line like this:

```cpp
recipient->array_[recipient->GetSize()].first = middle_key;
```

### 3. Redistribute flow

It is also useful to first look at the algorithm diagram.

The following figure shows the `Redistribute` function for leaf nodes.

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210928222404582-653166542.png)

When moving entries, remember to update the corresponding `index` key in the parent.

For internal nodes, however, the situation is not that simple. Can an internal node directly copy from its sibling and then modify its root key? Obviously not.

The handling for this case is shown in the following figures:

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210928222415485-1180061675.png)

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210928222424947-555321315.png)

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210928222432879-1318082133.png)

![img](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210928222439373-98329057.png)

Therefore, the full redistribution logic involves the following four cases:

> 1. Borrow from the left sibling for a leaf node.
> 2. Borrow from the right sibling for a leaf node.
> 3. Borrow from the left sibling for an internal node.
> 4. Borrow from the right sibling for an internal node.

#### 1. Borrowing from the left sibling for a leaf node

At this point, the deletion algorithm has been implemented. First, we can pass the test function:

```bash
cd build
make b_plus_tree_delete_test
./test/b_plus_tree_delete_test --gtest_also_run_disabled_tests
```

![image-20210126134731557](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126135615915-1581322684.png)

Then we can write a few tests ourselves. I use one example here.

Inserting `10`, `5`, `7`, `4`, and `9` produces the following correct result:

![image-20210126134908455](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126135636915-188462836.png)

Then delete element `7`.

![image-20210126134952965](https://raw.githubusercontent.com/SleepyLGod/images/dev/markdown/2282357-20210126135746654-1968820776.png)

The result is completely correct. This completes the second part. The final part is the implementation of latches and the iterator.
