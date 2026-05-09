# B/B+ Tree Background

> This page preserves and combines the two previous background notes. The first part is the general B/B+ tree comparison; the second part is the BusTub-oriented B+ tree implementation note.

# 😌 Pre: B & B+

#### 1 B-Tree <a href="#item-1" id="item-1"></a>

Before introducing B+ trees, first briefly introduce B-trees. These two data structures have similarities as well as differences. At the end, we also compare the differences between them.

**1.1 B-tree Concept**

A B-tree, also written as B-Tree, is a multi-way balanced search tree. Everyone is probably familiar with binary trees. In fact, B-trees and the B+ trees discussed later are also derived from the simplest binary-tree ideas, so there is nothing mysterious about them. First look at the definition of a B-tree.

* Each node has at most `m-1` **keys**, meaning key-value pairs that can be stored.
* The root node may have as few as 1 **key**.
* A non-root node has at least `m/2` **keys**.
* Keys inside each node are sorted in increasing order. For each key, all keys in its left subtree are smaller than it, while all keys in its right subtree are larger than it.
* All leaf nodes are at the same level; equivalently, the path length from the root to every leaf node is the same.
* Each node stores both index and data, meaning the corresponding key and value.

Therefore, the number of **keys** in the root node satisfies `1 <= k <= m-1`, while the number of **keys** in a non-root node satisfies `m/2 <= k <= m-1`.

Also note one concept: when describing a B-tree, we need to specify its order. The order indicates the maximum number of child nodes that a node can have, and it is usually represented by the letter `m`.

Use an example to illustrate the concepts above. Suppose this is a B-tree of order 5. The number of keys in the root node is in the range `1 <= k <= 4`, and the number of keys in a non-root node is in the range `2 <= k <= 4`.

Next, use an insertion example to explain the B-tree insertion process, and then explain key deletion.

**1.2 B-tree Insertion**

During insertion, remember one rule: **check whether the number of keys in the current node is less than or equal to `m-1`. If so, insert directly. If not, split the node into left and right parts using the middle key, and move the middle key into the parent node.**

Example: in a B-tree of order 5, each node can have at most 4 keys and at least 2 keys. Note that the diagrams below use one node to represent both key and value.

* Insert 18, 70, 50, 40

![](<../../../.gitbook/assets/image (19).png>)

* Insert 22

![](<../../../.gitbook/assets/image (8).png>)

When inserting 22, the number of keys in this node becomes greater than 4, so the node must split. The split rule was described above. After splitting, the tree becomes:

<figure><img src="../../../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

* Continue inserting 23, 25, 39

<figure><img src="../../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

After splitting, we get:

<figure><img src="../../../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

More insertion steps are not introduced here. With this example, the insertion process should already be clear.

**1.3 B-tree Deletion**

B-tree deletion is more complex than insertion, but if you remember a few cases, it is still easy to master.

* Start with a B-tree in the following initial state, then perform deletion.

<figure><img src="../../../.gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

* Delete 15. This case deletes an element from a leaf node. If the number of keys after deletion is still greater than `m/2`, we can delete it directly.

<figure><img src="../../../.gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>

* Next, delete 22. The rule for this case is: 22 is in a non-leaf node. **For deletion from a non-leaf node, use the successor key to overwrite the key being deleted, and then delete that successor key from the child branch where it is located**. To delete 22, move successor element 24 into the node where 22 was deleted.

<figure><img src="../../../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

At this point, the node containing 26 has only one element, fewer than 2 (`m/2`), so this node violates the requirement. The rule here is to borrow an element from a sibling node: **if deleting from a leaf node causes the number of elements to be less than `m/2`, and a sibling node has more than `m/2` elements, meaning the sibling has more than the minimum, first move an element from the parent node into this node, and then move an element from the sibling node into the parent node**. This restores the requirement.

The process is clearer from the following diagrams.

<figure><img src="../../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

* Next delete 28. This is **deletion from a leaf node**. After deletion, the requirement is not satisfied. We need to consider borrowing from a sibling node, but the sibling node also does not have extra keys; it has only 2. What should we do? In this case, **first move an element from the parent node into this node, then merge the keys in the current node and its sibling node into a new node**.

<figure><img src="../../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

After moving, merge with the sibling node.

<figure><img src="../../../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

Deletion has only the cases above. Apply the corresponding action for each case.

After the discussion above, you should have a basic understanding of B-trees. The next part introduces B+ trees. With the comparison against B+ trees, the picture should become clearer.

#### 2 B+ Tree <a href="#item-2" id="item-2"></a>

**2.1 B+ Tree Overview**

B+ trees are very similar to B-trees. First look at their **similarities**.

* The root node has at least one element.
* The element count range for a non-root node is `m/2 <= k <= m-1`.

**Differences**:

* A B+ tree has two types of nodes: internal nodes, also called index nodes, and leaf nodes. Internal nodes are non-leaf nodes. Internal nodes do not store data; they store only indexes. All data is stored in leaf nodes.
* Keys in internal nodes are sorted in increasing order. For a key in an internal node, all keys in the left subtree are smaller than it, and keys in the right subtree are greater than or equal to it. Records in leaf nodes are also sorted by key.
* Each leaf node stores a pointer to its adjacent leaf node, and leaf nodes are linked in increasing key order.
* The parent node stores the first element index of the right child.

Now look at a B+ tree example to get a feel for it.

<figure><img src="../../../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

**2.2 Insertion**

Insertion is simple. Remember one technique: **when the number of elements in a node exceeds `m-1`, split the node into left and right parts around the middle element. The middle element is copied into the parent as an index, but the middle element itself remains in the right split part**.

The following uses insertion into a B+ tree of order 5 as an example. A B+ tree of order 5 has at least 2 elements and at most 4 elements per node.

* Insert 5, 10, 15, 20

![](<../../../.gitbook/assets/image (30).png>)

* Insert 25. Now the number of elements exceeds 4, so split.

<figure><img src="../../../.gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>

Then insert 26 and 30, continuing to split.

<figure><img src="../../../.gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

With these examples, insertion should be clear. Next, look at deletion.

**2.3 Deletion**

Deletion is simpler than in a B-tree because **leaf nodes have sibling pointers. When borrowing an element from a sibling, the operation does not need to go through the parent node; it can move directly through the sibling, assuming the sibling has more than `m/2` elements, and then update the parent index. If the sibling does not have more than `m/2` elements, meaning the sibling also has no extra element, merge the current node and the sibling node, and delete the corresponding key from the parent node**. The following example illustrates the details.

* Initial state

<figure><img src="../../../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

* Delete 10. After deletion, the node violates the requirement. The left sibling has an extra element, so borrow one and finally update the parent index.

<figure><img src="../../../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

* Delete element 5. The node violates the requirement, and neither left nor right sibling has an extra element. Therefore, merge with a sibling and finally update the parent index.

<figure><img src="../../../.gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>

* The parent index also violates the requirement, so perform the same kind of operation as above.

<figure><img src="../../../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

At this point, B+ tree deletion is complete. After walking through the examples, it should feel straightforward.

#### 3 B-tree and B+ Tree Summary <a href="#item-3" id="item-3"></a>

Compared with B-trees, B+ trees have several advantages:

* A single node stores more elements, reducing the number of I/O operations during lookup. This makes B+ trees more suitable as the underlying data structure for database indexes such as MySQL indexes.
* Every query reaches a leaf node, so query performance is stable. In a B-tree, data may be found at any node, so lookup depth is less uniform.
* All leaf nodes form an ordered linked list, making range lookup more convenient.

---

---
description: Knowledge summary
---

# 🤣 Pre: B+Tree

## B+ Tree <a href="#b-shu" id="b-shu"></a>

### Why B+ Trees Are Needed <a href="#wei-shen-me-xu-yaobshu" id="wei-shen-me-xu-yaobshu"></a>

A B+ tree is essentially an index data structure. For example, suppose we want to retrieve a `student` record by a given ID. Without an index, we may need to scan every `student` record from the first record until we find one whose ID matches the given ID. This is clearly time-consuming.\
If we already maintain an index structure whose key is ID, we can query the index for the location of the record corresponding to that ID, and then directly read the record from that location. Querying the index for the location corresponding to an ID must be efficient, and a B+ tree can guarantee this in `O(log n)` time complexity.

### B+ Tree Properties <a href="#b-shu-de-xing-zhi" id="b-shu-de-xing-zhi"></a>

A B+ tree consists of leaf nodes and internal nodes. Like other tree structures, it has requirements for the number and ordering of `(KEY, VALUE)` pairs.

#### Leaf Nodes <a href="#ye-zi-jie-dian" id="ye-zi-jie-dian"></a>

The format is as follows:

```markdown
 *  ---------------------------------------------------------------------------
 * | HEADER | KEY(1) + RID(1) | KEY(2) + RID(2) | ... | KEY(n) + RID(n) 
 *  ---------------------------------------------------------------------------
```

Assume a leaf node can hold at most `n` `(KEY, RID)` pairs. Then this leaf node must never contain fewer than `ceil(n/2)` `(KEY, RID)` pairs. If the number of `(KEY, RID)` pairs is `x`, then `x` must satisfy:

```markdown
ceil(n/2) <= x <= n
```

`ceil` means rounding up.\
`KEY` is the search key, and `RID` is the location of the record corresponding to that key. `(KEY, RID)` pairs are sorted in increasing `KEY` order.\
The HEADER structure is:

```markdown
 * ----------------------------------------------------------------------------------------
 * | PageType (4) | LSN (4) | CurrentSize (4) | MaxSize (4) | ParentPageId (4) | PageId(4) |
 * ---------------------------------------------------------------------------------------
```

`ParentPageId` points to the parent node.

#### Internal Nodes <a href="#nei-bu-jie-dian" id="nei-bu-jie-dian"></a>

```markdown
 *  ----------------------------------------------------------------------------------------
 * | HEADER | INVALID_KEY+PAGE_ID(1) | KEY(2)+PAGE_ID(2) | ... | KEY(n)+PAGE_ID(n) |
 *  ----------------------------------------------------------------------------------------
```

Assume an internal node can hold at most `n` `(KEY, PAGE_ID)` pairs. As with leaf nodes, `x` must satisfy:

```
ceil(n/2) <= x <= n
```

`KEY` is the search key, and `PAGE_ID` is the ID of the child node.\
`(KEY, PAGE_ID)` pairs are sorted in increasing `KEY` order.\
The first `KEY` is invalid.\
Assume the keys in the subtree corresponding to `PAGE_ID(i)` are represented by `SUB_KEY`; then those keys satisfy `KEY(i) <= SUB_KEY < KEY(i+1)`.\


<figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_1_page_node.PNG" alt=""><figcaption></figcaption></figure>

### Lookup <a href="#cha-zhao-cao-zuo" id="cha-zhao-cao-zuo"></a>

The textbook gives pseudocode for `find` on p.489. In summary, first find the leaf node where the `KEY` should appear, and then search that leaf node for the `RID` corresponding to the `KEY`.\
As shown below:

<figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_2_find.PNG" alt=""><figcaption></figcaption></figure>

\
Suppose the key we want to find is 38. First, in root node A, determine which child node should contain 38. Based on the properties above, 38 should appear in the subtree rooted at B. Continue searching node B, and so on. Eventually, 38 should appear in leaf node H. Finally, search for 38 inside H.\
Therefore, for internal nodes, we need a `Lookup(const KeyType &key, const KeyComparator &comparator)` method to find which child node's subtree should contain `key`.

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
ValueType
B_PLUS_TREE_INTERNAL_PAGE_TYPE::Lookup(const KeyType &key,
                                       const KeyComparator &comparator) const {
    assert(GetSize() >= 2);
    // Find the first index whose array[index].first is greater than or equal to key.
    // The search starts from index 1.
    int left = 1;
    int right = GetSize() - 1;
    int mid;
    int compareResult;
    int targetIndex;
    while (left <= right) {
        mid = left + (right - left) / 2;
        compareResult = comparator(array[mid].first, key);
        if (compareResult == 0) {
            left = mid;
            break;
        } else if (compareResult < 0) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    targetIndex = left;

    // key is greater than every key in array.
    if (targetIndex >= GetSize()) {
        return array[GetSize() - 1].second;
    }

    if (comparator(array[targetIndex].first, key) == 0) {
        return array[targetIndex].second;
    } else {
        return array[targetIndex - 1].second;
    }
}
```
{% endcode %}

Because the keys are sorted, we can first use binary search to find the first index `targetIndex` whose key is greater than or equal to `KEY`. If the key at `targetIndex` is the key we are looking for, the value at `targetIndex` is the next node to search. Otherwise, the value at `targetIndex - 1` is the next node to search.

### Insertion <a href="#cha-ru-cao-zuo" id="cha-ru-cao-zuo"></a>

The textbook gives complete pseudocode for `insert(key, value)` on p.494.\
The idea is:

1. First find the leaf node where `key` should appear, then insert `(key, value)` into that leaf node.
2.  If the number of key-value pairs in the leaf node exceeds the maximum after insertion, split it. If it does not exceed the maximum, the operation is complete.\
    \
    In the figure above, suppose we want to insert `(7, 'g')`, but leaf node `p1` is already full before insertion. First insert the pair, then split the resulting node and create a new node `p3`. Move half of the original elements from `p1` to `p3`, then insert `(6, p3)` into the parent node `p2`, where 6 is the first key of the newly created node `p3`.\
    Similarly, if inserting `(6, p3)` into parent node `p2` causes `p2` to exceed the maximum, `p2` also needs to split, and this may continue recursively. The process may create a new root node.

    <figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_3_insert.png" alt=""><figcaption><p>img.png</p></figcaption></figure>

### Deletion <a href="#shan-chu-cao-zuo" id="shan-chu-cao-zuo"></a>

#### Idea <a href="#shan-chu-cao-zuo" id="shan-chu-cao-zuo"></a>

1. First find the leaf node where `key` should appear, then delete the key-value pair corresponding to `key` in that leaf node.
2.  If the number of pairs after deletion is less than the required minimum, there are two possible actions. If the total number of elements in the current node and its sibling does not exceed the allowed maximum, merge them. Otherwise, borrow one element from the sibling node.

    <figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_4_delete.png" alt=""><figcaption></figcaption></figure>

First case in the figure above:\
After deleting `(7, 'g')`, `p3` has only one element, fewer than the minimum allowed count of 2. Therefore, move `(6, 'f')` into sibling node `p1`, delete node `p3`, and delete `(6, p3)` from parent node `p2`. If `p2` also has fewer than the minimum allowed count, apply the process recursively.\
Second case:\
After deleting `(8, 'h')` from `p3`, `p3` has only one element. Borrow one element `(6, f)` from sibling node `p1`, then change the parent entry `(7, 'g')` to `(6, 'f')`. This case does not require recursion.

## Supporting Concurrent Operations <a href="#zhi-chi-bing-fa-cao-zuo" id="zhi-chi-bing-fa-cao-zuo"></a>

The most brute-force method is to acquire a lock at the beginning of `find`, `insert`, and `delete`, and release it after execution completes. This is logically correct, but concurrency is poor because operations are effectively serialized.

### Crabbing Protocol <a href="#crabbing-xie-yi" id="crabbing-xie-yi"></a>

This protocol allows multiple threads to access and modify a B+ tree concurrently.

### Basic Algorithm <a href="#ji-ben-suan-fa" id="ji-ben-suan-fa"></a>

1. For lookup operations, start from the root node. First acquire the **read lock** of the root node, then find the child node where the key should appear, acquire the **read lock** of the child node, and then release the **read lock** of the root node. Continue this way until the target leaf node is reached; at that point, the leaf node holds the read lock.
2. For delete and insert operations, also start from the root node. First acquire the **write lock** of the root node. Once the child node also acquires its **write lock**, check whether the root node is safe. If it is safe, release the **write locks** of all ancestors of the child node. Continue this way until the target leaf node is reached. A node is safe if, for insertion, inserting one more element will not cause a split, or for deletion, deleting one more element will not cause a merge.

Example lookup process for `key = 38`:\


<figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_5_crabbing_protol_find.png" alt=""><figcaption></figcaption></figure>

Example insertion process for inserting 25:\


<figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_6-crabbing_protol_insert.png" alt=""><figcaption></figcaption></figure>

The word "crab" explains the name of the crabbing protocol. After understanding the locking process, it should be easy to see why the protocol is called crabbing.

### Things to Watch Out For <a href="#xu-yao-zhu-yi-de-di-fang" id="xu-yao-zhu-yi-de-di-fang"></a>

We need to protect the root node ID.\
Consider the following situation:\
Two threads execute insertions at the same time. Before insertion, the B+ tree has only one node. After thread 1 inserts its key, the node will split and generate a new root node. Before thread 1 finishes the split, another thread reads the old root node and inserts its key into the wrong leaf node.\
Solution:\
Add a lock around every access or modification of `root_page_id_`, and release the lock after accessing or modifying `root_page_id_`. `root_page_id_` points to the root node of the B+ tree and is stored in memory for fast lookup.

## Pitfalls Encountered in the Lab and Solutions <a href="#shi-yan-yu-dao-de-keng-he-jie-jue-fang-an" id="shi-yan-yu-dao-de-keng-he-jie-jue-fang-an"></a>

1. As mentioned earlier, `root_page_id_` needs to be protected. A mutex can be used: lock before access or modification, and unlock afterward. One lock operation must correspond to one unlock operation. If `unlock()` is called one extra time, the protection is lost. Because `unlock()` calls are distributed across multiple functions, it is easy to accidentally call it one time too many, so be very careful.
2. A page lock must be released before the page is unpinned. Why? We know that after unpinning, if `pin_count` becomes 0, the page may be sent to the `LRUReplacer`. When there are not enough pages, the system takes a page from the `LRUReplacer`, writes its content back to disk, and uses it to store another page. Consider this scenario: during insertion of 25, the target leaf node has been found and must hold a write lock. If after insertion we first unpin this page and only then release its lock, the following situation may occur. After unpinning but before releasing the lock, this page is sent to the `LRUReplacer`. Another thread requests page 1, but all pages are occupied. The `LRUReplacer` chooses this still-locked page for eviction and uses it to store page 1. Because the lock on the old page has not yet been released, another thread may directly access or modify it. When the original thread returns and releases the lock, it is already too late.
3. The test cases provided by the lab itself are far from sufficient. Even if all of them pass, the code is not guaranteed to be correct. I added many tests myself, covering multi-threaded cases, root-node splits, and so on. The original code only tested `BPlusTree`, so I added separate tests for `BPlusTreeInternalPage` and `BPlusTreeLeafPage`. This ensures those two components are correct before using them to build `BPlusTree`.
4. After using a page, it should be unpinned immediately. Do not forget to unpin it. If a page is never unpinned, it can never be reused to store other pages. Once all pages are occupied, the system can no longer continue running. This problem bothered me for a long time, so be very careful.
5. One difficult part of this lab is debugging. Use `assert` and logs frequently.
