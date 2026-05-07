# B/B+ Tree Background

> This page preserves and combines the two previous background notes. The first part is the general B/B+ tree comparison; the second part is the BusTub-oriented B+ tree implementation note.

# 😌 Pre: B & B+

#### 1 B树 <a href="#item-1" id="item-1"></a>

在介绍B+树之前， 先简单的介绍一下B树，这两种数据结构既有相似之处，也有他们的区别，最后，我们也会对比一下这两种数据结构的区别。

**1.1 B树概念**

B树也称B-树,它是一颗多路平衡查找树。二叉树我想大家都不陌生，其实，B树和后面讲到的B+树也是从最简单的二叉树变换而来的，并没有什么神秘的地方，下面我们来看看B树的定义。

* 每个节点最多有m-1个**关键字**（可以存有的键值对）。
* 根节点最少可以只有1个**关键字**。
* 非根节点至少有m/2个**关键字**。
* 每个节点中的关键字都按照从小到大的顺序排列，每个关键字的左子树中的所有关键字都小于它，而右子树中的所有关键字都大于它。
* 所有叶子节点都位于同一层，或者说根节点到每个叶子节点的长度都相同。
* 每个节点都存有索引和数据，也就是对应的key和value。

所以，根节点的**关键字**数量范围：`1 <= k <= m-1`，非根节点的**关键字**数量范围：`m/2 <= k <= m-1`。

另外，我们需要注意一个概念，描述一颗B树时需要指定它的阶数，阶数表示了一个节点最多有多少个孩子节点，一般用字母m表示阶数。

我们再举个例子来说明一下上面的概念，比如这里有一个5阶的B树，根节点数量范围：1 <= k <= 4，非根节点数量范围：2 <= k <= 4。

下面，我们通过一个插入的例子，讲解一下B树的插入过程，接着，再讲解一下删除关键字的过程。

**1.2 B树插入**

插入的时候，我们需要记住一个规则：**判断当前结点key的个数是否小于等于m-1，如果满足，直接插入即可，如果不满足，将节点的中间的key将这个节点分为左右两部分，中间的节点放到父节点中即可。**

例子：在5阶B树中，结点最多有4个key,最少有2个key（注意：下面的节点统一用一个节点表示key和value）。

* 插入18，70，50,40

![](<../../../.gitbook/assets/image (19).png>)

* 插入22

![](<../../../.gitbook/assets/image (8).png>)

插入22时，发现这个节点的关键字已经大于4了，所以需要进行分裂，分裂的规则在上面已经讲了，分裂之后，如下。

<figure><img src="../../../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

* 接着插入23，25，39

<figure><img src="../../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

分裂，得到下面的。

<figure><img src="../../../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

更过的插入的过程就不多介绍了，相信有这个例子你已经知道怎么进行插入操作了。

**1.3 B树的删除操作**

B树的删除操作相对于插入操作是相对复杂一些的，但是，你知道记住几种情况，一样可以很轻松的掌握的。

* 现在有一个初始状态是下面这样的B树，然后进行删除操作。

<figure><img src="../../../.gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

* 删除15，这种情况是删除叶子节点的元素，如果删除之后，节点数还是大于`m/2`，这种情况只要直接删除即可。

<figure><img src="../../../.gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>

* 接着，我们把22删除，这种情况的规则：22是非叶子节点，**对于非叶子节点的删除，我们需要用后继key（元素）覆盖要删除的key，然后在后继key所在的子支中删除该后继key**。对于删除22，需要将后继元素24移到被删除的22所在的节点。

<figure><img src="../../../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

此时发现26所在的节点只有一个元素，小于2个（m/2），这个节点不符合要求，这时候的规则（向兄弟节点借元素）：**如果删除叶子节点，如果删除元素后元素个数少于（m/2），并且它的兄弟节点的元素大于（m/2），也就是说兄弟节点的元素比最少值m/2还多，将先将父节点的元素移到该节点，然后将兄弟节点的元素再移动到父节点**。这样就满足要求了。

我们看看操作过程就更加明白了。

<figure><img src="../../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

* 接着删除28，**删除叶子节点**，删除后不满足要求，所以，我们需要考虑向兄弟节点借元素，但是，兄弟节点也没有多的节点（2个），借不了，怎么办呢？如果遇到这种情况，**首先，还是将先将父节点的元素移到该节点，然后，将当前节点及它的兄弟节点中的key合并，形成一个新的节点**。

<figure><img src="../../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

移动之后，跟兄弟节点合并。

<figure><img src="../../../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

删除就只有上面的几种情况，根据不同的情况进行删除即可。

上面的这些介绍，相信对于B树已经有一定的了解了，接下来的一部分，我们接着讲解B+树，我相信加上B+树的对比，就更加清晰明了了。

#### 2 B+树 <a href="#item-2" id="item-2"></a>

**2.1 B+树概述**

B+树其实和B树是非常相似的，我们首先看看**相同点**。

* 根节点至少一个元素
* 非根节点元素范围：m/2 <= k <= m-1

**不同点**。

* B+树有两种类型的节点：内部结点（也称索引结点）和叶子结点。内部节点就是非叶子节点，内部节点不存储数据，只存储索引，数据都存储在叶子节点。
* 内部结点中的key都按照从小到大的顺序排列，对于内部结点中的一个key，左树中的所有key都小于它，右子树中的key都大于等于它。叶子结点中的记录也按照key的大小排列。
* 每个叶子结点都存有相邻叶子结点的指针，叶子结点本身依关键字的大小自小而大顺序链接。
* 父节点存有右孩子的第一个元素的索引。

下面我们看一个B+树的例子，感受感受它吧！

<figure><img src="../../../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

**2.2 插入操作**

对于插入操作很简单，只需要记住一个技巧即可：**当节点元素数量大于m-1的时候，按中间元素分裂成左右两部分，中间元素分裂到父节点当做索引存储，但是，本身中间元素还是分裂右边这一部分的**。

下面以一颗5阶B+树的插入过程为例，5阶B+树的节点最少2个元素，最多4个元素。

* 插入5，10，15，20

![](<../../../.gitbook/assets/image (30).png>)

* 插入25，此时元素数量大于4个了，分裂

<figure><img src="../../../.gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>

接着插入26，30，继续分裂

<figure><img src="../../../.gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

有了这几个例子，相信插入操作没什么问题了，下面接着看看删除操作。

**2.3 删除操作**

对于删除操作是比B树简单一些的，因为**叶子节点有指针的存在，向兄弟节点借元素时，不需要通过父节点了，而是可以直接通过兄弟节移动即可（前提是兄弟节点的元素大于m/2），然后更新父节点的索引；如果兄弟节点的元素不大于m/2（兄弟节点也没有多余的元素），则将当前节点和兄弟节点合并，并且删除父节点中的key**，下面我们看看具体的实例。

* 初始状态

<figure><img src="../../../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

* 删除10，删除后，不满足要求，发现左边兄弟节点有多余的元素，所以去借元素，最后，修改父节点索引

<figure><img src="../../../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

* 删除元素5，发现不满足要求，并且发现左右兄弟节点都没有多余的元素，所以，可以选择和兄弟节点合并，最后修改父节点索引

<figure><img src="../../../.gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>

* 发现父节点索引也不满足条件，所以，需要做跟上面一步一样的操作

<figure><img src="../../../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

这样，B+树的删除操作也就完成了，是不是看完之后，觉得非常简单！

#### 3 B树和B+树总结 <a href="#item-3" id="item-3"></a>

B+树相对于B树有一些自己的优势，可以归结为下面几点。

* 单一节点存储的元素更多，使得查询的IO次数更少，所以也就使得它更适合做为数据库MySQL的底层数据结构了。
* 所有的查询都要查找到叶子节点，查询性能是稳定的，而B树，每个节点都可以查找到数据，所以不稳定。
* 所有的叶子节点形成了一个有序链表，更加便于查找。

---

---
description: 知识点总结
---

# 🤣 Pre:  B+Tree

## B+树 <a href="#b-shu" id="b-shu"></a>

### 为什么需要B+树 <a href="#wei-shen-me-xu-yaobshu" id="wei-shen-me-xu-yaobshu"></a>

B+树本质上是一个索引数据结构。比如我们要用某个给定的ID去检索某个student记录，如果没有索引的话，我们可能从第一条记录开始遍历每一个student记录，直到找到某个ID和我们给定的ID一致的记录。可想而知，这是非常耗时的。\
如果我们已经维护了一个以ID为KEY的索引结构，我们可以向索引查询这个ID对应的记录所在的位置，然后直接从这个位置读取这个记录。从索引查询某个ID对应的位置，这个操作需要高效，B+树能保证以O(log n)的时间复杂度完成。

### B+树的性质 <a href="#b-shu-de-xing-zhi" id="b-shu-de-xing-zhi"></a>

B+树由叶子节点和内部节点组成，和其它树结构差不多，但是对(KEY, VALUE)的个数和排列顺序有要求。

#### 叶子节点： <a href="#ye-zi-jie-dian" id="ye-zi-jie-dian"></a>

格式如下：

```markdown
 *  ---------------------------------------------------------------------------
 * | HEADER | KEY(1) + RID(1) | KEY(2) + RID(2) | ... | KEY(n) + RID(n) 
 *  ---------------------------------------------------------------------------
```

假设叶子结点最多能容纳个n个(KEY, RID)对，那么该叶子节点任何时候都不能少于n/2向上取整个(KEY, RID)对。假设(KEY, RID)对个数为x，那么x必须满足：

```markdown
ceil(n/2) <= x <= n
```

ceil 表示向上取整。\
KEY是search key，RID是该KEY对应的记录的位置。(KEY, RID)对按照KEY的増序进行排列。\
HEADER的结构如下：

```markdown
 * ----------------------------------------------------------------------------------------
 * | PageType (4) | LSN (4) | CurrentSize (4) | MaxSize (4) | ParentPageId (4) | PageId(4) |
 * ---------------------------------------------------------------------------------------
```

`ParentPageId` 指向父节点。

#### 内部节点 <a href="#nei-bu-jie-dian" id="nei-bu-jie-dian"></a>

```markdown
 *  ----------------------------------------------------------------------------------------
 * | HEADER | INVALID_KEY+PAGE_ID(1) | KEY(2)+PAGE_ID(2) | ... | KEY(n)+PAGE_ID(n) |
 *  ----------------------------------------------------------------------------------------
```

假设内部节点最多容纳n个(KEY, PAGE\_ID)对，和叶子节点一样，x必须满足：

```
ceil(n/2) <= x <= n
```

KEY表示search key，PAGE\_ID指的是子节点的ID。\
(KEY, PAGE\_ID)对按照KEY的増序进行排列。\
第一个KEY是无效的。\
假设PAGE\_ID(i)对应的子树中的KEY用SUB\_KEY表示，那么SUBKEY都满足：`KEY(i) <= SUB_KEY < KEY(i+1)`。\


<figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_1_page_node.PNG" alt=""><figcaption></figcaption></figure>

### 查找操作 <a href="#cha-zhao-cao-zuo" id="cha-zhao-cao-zuo"></a>

课本p489给出了find的伪代码。总结来说就是先找到KEY应该出现的叶子节点，然后在该叶子节点中，查找KEY对应的RID。\
如下图：

<figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_2_find.PNG" alt=""><figcaption></figcaption></figure>

\
假如我们希望查找的KEY为38，第一步在根节点A查找38应该出现在哪个子节点中，根据之前的性质，38应该出现在以B为根的子树中，继续查找节点B，以此类推，最终38应该出现在H的叶子节点中。最后我们在H中查找38。\
所以对于内部节点，我们需要一个`Lookup(const KeyType &key, const KeyComparator &comparator)`方法，查找key应该出现在哪个子节点对应的子树中。

{% code overflow="wrap" lineNumbers="true" %}
```cpp
INDEX_TEMPLATE_ARGUMENTS
ValueType
B_PLUS_TREE_INTERNAL_PAGE_TYPE::Lookup(const KeyType &key,
                                       const KeyComparator &comparator) const {
    assert(GetSize() >= 2);
    // 先找到第一个array[index].first大于等于key的index（从index 1开始）
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

    // key比array中所有key都要大
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

因为KEY是已排序的，所以可以先二分查找第一个大于或等于KEY的下标targetIndex，如果targetIndex对应的KEY就是我们要找的KEY，那么targetIndex对应的value就是下一步要搜索的节点，否则targetIndex-1对应的value是下一步应该搜索的节点。

### 插入操作 <a href="#cha-ru-cao-zuo" id="cha-ru-cao-zuo"></a>

课本p494给出了完整的insert(key, value)操作的伪代码。\
思路就是：

1. 先找到key应该出现的叶子节点，将(key, value)插入到该叶子节点中。
2.  如果插入后该叶子节点中键值对超出了最大值，则进行分裂。如果插入后没有超出最大限制，那么就完成任务了。\
    \
    如上图准备插入(7, 'g')，但是插入前p1叶子结点已经满了，那么先插入，然后将插入后的节点，分裂出新的节点p3，将p1原来一半的元素挪到p3，然后将(6, p3)插入到父节点p2中，其中6是新创建的节点p3第一个key。\
    同样的，如果我们在父节点p2中插入了(6, p3)导致了p2超过最大限制，p2也需要分裂，以此类推，这个过程可能产生新的根节点。

    <figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_3_insert.png" alt=""><figcaption><p>img.png</p></figcaption></figure>

### 删除操作 <a href="#shan-chu-cao-zuo" id="shan-chu-cao-zuo"></a>

#### 思路： <a href="#shan-chu-cao-zuo" id="shan-chu-cao-zuo"></a>

1. 先找到key应该出现的叶子节点，删除该叶子节点中key对应的键值对。
2.  删除后如果个数少于规定最少个数，那么有两个措施，如果当前节点个数和兄弟节点个数总和不超过允许的最大个数，那么进行并合。否则，从兄弟节点中借一个元素。

    <figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_4_delete.png" alt=""><figcaption></figcaption></figure>

上图第一种情况：\
删除(7, 'g')后，p3只有一个元素，少于最少允许的个数（2），于是将(6, 'f')已到兄弟节点p1, 删除p3节点，并且删除父节点p2中的(6, p3)，如果p2也少于最少允许个数，递归进行。\
第二种请求：\
删除p3的(8, 'h')后，p3只有一个元素，于是从兄弟节点p1借一个元素(6, f)，然后将父节点(7, 'g')修改为(6, 'f')，这种情况不需要递归。

## 支持并发操作 <a href="#zhi-chi-bing-fa-cao-zuo" id="zhi-chi-bing-fa-cao-zuo"></a>

最粗暴的方式就是在find, insert, delete开始就加锁，执行完毕后解锁，这样逻辑上没有问题，但是并发效率很低，相当于串行执行。

### crabbing协议 <a href="#crabbing-xie-yi" id="crabbing-xie-yi"></a>

该协议允许多个线程同时访问修改B+树。

### 基本算法 <a href="#ji-ben-suan-fa" id="ji-ben-suan-fa"></a>

1. 对于查询操作，从根节点开始，首先获取根节点的**读锁**，然后在根节点中查找key应该出现的孩子节点，获取孩子节点的**读锁**，然后释放根节点的**读锁**，以此类推，直到找到目标叶子节点，此时该叶子节点获取了读锁。
2. 对于删除和插入操作，也是从根节点开始，先获取根节点的**写锁**，一旦孩子节点也获取了**写锁**，检查根节点是否安全，如果安全释放孩子节点所有祖先节点的**写锁**，以此类推，直到找到目标叶子节点。节点安全定义如下：如果对于插入操作，如果再插入一个元素，不会产生分裂，或者对于删除操作，如果再删除一个元素，不会产生并合。

举个查找过程的例子，查找key=38：\


<figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_5_crabbing_protol_find.png" alt=""><figcaption></figcaption></figure>

举个插入过程的例子，插入25：\


<figure><img src="https://blog-1253119293.cos.ap-beijing.myqcloud.com/cmu-15445/lab2/lab2_6-crabbing_protol_insert.png" alt=""><figcaption></figcaption></figure>

crab有螃蟹的意思，了解完crabbing协议加锁的过程，应该不难理解为什么叫crabbing协议了吧。

### 需要注意的地方 <a href="#xu-yao-zhu-yi-de-di-fang" id="xu-yao-zhu-yi-de-di-fang"></a>

我们需要保护根节点id。\
考虑下面这种情况：\
两个线程同时执行插入操作，插入前B+树只有一个节点，线程一插入当前key后将分裂，生成一个新的根节点。另一个线程在线程一分裂前读取了旧的根节点，从而将key插入到了错误的叶子节点中。\
解决办法：\
在访问，修改root\_page\_id\_的地方加锁，访问或者修改完毕root\_page\_id\_后释放锁。root\_page\_id\_指向的是该B+树的根节点，会保存在内存中，以便快速查找。

## 实验遇到的坑和解决方案 <a href="#shi-yan-yu-dao-de-keng-he-jie-jue-fang-an" id="shi-yan-yu-dao-de-keng-he-jie-jue-fang-an"></a>

1. 前文提到我们需要保护root\_page\_id\_这个变量，可以用一个mutex，访问或修改前加锁，访问或者修改后释放锁。一次加锁只能对应一次解锁，如果多调用了一次unlock()，同样起不到保护的作用。unlock()调用分别在各个函数中，很可能不小心就多调用了次，所以千万要小心。
2. 必须先释放Page上的锁，然后才能unpin该Page。为什么？我们知道unpin后，如果pin\_count为0，那么这个Page将被送到LRUReplacer，当没有足够的Page时，将从LRUReplacer中取Page，将该Page的内容保存到磁盘后用于保存其它其它页的内容。考虑下面这个场景：在插入25的过程中，查找到目标叶子节点，这时该叶子节点肯定被加上了写锁，如果我们执行完插入后，先unpin了该Page，然后才释放该Page的锁。可能出现这种情况，在unpin完后，释放锁前，这个Page被送到了LRUReplacer，另一个线程请求访问页面1，但是所有的Page都被占用了，LRUReplacer选择这个淘汰带锁的这个Page来保存页面1，因为该Page的锁还没释放，所以另一个线程可以直接访问或者修改，这是回到原来的线程，再释放已经晚了。
3. lab本身提供的测试case是完全不够的，就算全部通过了，也不能保证代码是正确的。我自己加入了很多测试，涵盖多个线程的，根节点分裂等case。原代码只有对BPlusTree的测试，所以我添加了对BPlusTreeInternalPage和BPlusTreeLeafPage单独的测试，这样在用BPlusTreeInternalPage和BPlusTreeLeafPage构建BPlusTree前能保证自己是正确的。
4. 在使用完一个Page后应该立刻unpin掉，不能忘记unpin，如果忘记unpin的话，那么这个Page将永远不能用于保存其它页，当所有Page都被占用后，系统将无法继续运行。这个问题一度困扰我很久，一定要非常仔细。
5. 本lab的一个难点是调试，多使用assert和log。
