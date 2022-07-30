# 😩 Skiplist

### Introduction:

SkipList是一个实现快速查找、增删数据的概率数据结构，可以做到![{\mathcal {O\}}(\log n)](https://wikimedia.org/api/rest\_v1/media/math/render/svg/74a9dfea91c47d1c6563e89bbcd891771b91acfa)复杂度的查找和![{\mathcal {O\}}(\log n)](https://wikimedia.org/api/rest\_v1/media/math/render/svg/74a9dfea91c47d1c6563e89bbcd891771b91acfa)的增删改。

从时间复杂度上来看，似乎和平衡树差不多，但是和平衡树比较起来，它的编码复杂度更低，实现起来更加简单。平衡树经常需要旋转操作来维护两边子树的平衡，不仅编码复杂，理解困难，而且debug也非常不方便。SkipList克服了这些缺点，原理简单，实现起来也非常方便。

首先你要知道链表，链表遍历是![\[公式\]](https://www.zhihu.com/equation?tex=O%28n%29)，但是当你骚操作一下：

```cpp
struct node {
    int value_;
    int* next_;
    bool is_even_;
    int* second_next_;
}
```

在当中一般的节点上增加一个指针，指向后面两个的元素。这样我们遍历的速度可以提升一倍，最快就可以在![\[公式\]](https://www.zhihu.com/equation?tex=O%28n%2F2%29)的时间内遍历完整个链表了。

![小优化](<../.gitbook/assets/image (4) (1).png>)

同理继续骚下去，如果我们继续增加节点上指针的个数，立个小目标，设置![\[公式\]](https://www.zhihu.com/equation?tex=%5Clog+n)个指针，完全可以在![\[公式\]](https://www.zhihu.com/equation?tex=%5Clog+n)的时间内完成元素的查找，这就是SkipList的精髓。

但是捏由于查找需要，我们还要保证元素的有序性。有个前提：元素添加的顺序时无序的。

跳表又引入了新的骚操作：随机深度 -> 一个节点的指针数量是随机的

![随机深度](<../.gitbook/assets/image (3) (2).png>)

每个节点的第`i`个指针（这里就不是指向后第`i`个元素的了）指向的元素一定是离它最近的、指针数量大于等于`i`的元素（的第`i`个指针）。

### Node:

我们首先说node:

由于我们需要一个字段来查找，一个字段存储结果，所以显然key和value是必须的字段。

另外就是每个节点会有一个指针列表，记录可以指向的位置。

Node类中的方法：

* 为第k个后向指针赋值
* 获取指定深度的指针指向的节点的key
* 获取指定深度的指针指向的节点

### Skip! Skip! Skip!

建议先写构造函数和随机生成节点深度的函数，后者在跳表中要设计一个概率P，随机一个0-1的浮点值，大于P则深度++，否则返回当前深度，为了防止极端情况——深度炸了，我们规定一个最大深度。

另外， 跳表中除了有head还有tail，由于要实现查询，就要实现元素有序，所以我们把head的key设置成无穷小，tail的key设置成无穷大，默认head的后向指针是全满的，全部指向tail。

#### query

类似贪心，每次都看本节点最大的key，若小于要查找的就跳，否则就从大到小遍历本节点的key

![](https://pic4.zhimg.com/v2-a23986f920c2fd924725bec42e94a0ff\_b.jpg)

比如上图当中，假设我们要查找20，首先我们在head的位置的最高点往后看，直接看到了正无穷，它是大于20的，说明我们看太远了，应该往下走一层。于是我们走到4层，这次我们看到了17，它是小于20的，所以就移动过去。

移动到了17之后，我们还是从4层开始看起，然后发现每一层看到的元素都大于等于20，那么说明17就是距离20最近的元素（有可能20不存在）。那么我们从17开始往后移动一格，就是20可能出现的位置，如果这个位置不是20，那么说明20不存在。

#### delete

首先找到要删除的节点X。

直接删掉不行，还要改要删掉的节点X前面的一些节点的后向指针（指向要删除节点X的指针）

不妨联系查找，还记得我们查找的时候，每次都看得尽量远的贪心法吗？

我们每次发生”下楼“操作的元素不就是最近的一个能看到X的位置吗？也就是说我们把查找过程中发生下楼的位置都记录下来即可。

#### insert

首先同样需要查找，因为我们要将元素放到正确的位置。

如果这个位置已经有元素了，那么我们直接修改它的value，其实这就是修改操作了，如果设计成禁止修改，也可以返回失败。

插入的过程同样会影响其他元素的指针指向的内容，我们分析一下就会发现，插入的过程和删除其实是相反的。删除的过程当中我们需要**将指向x的指向x指向的位置**，而插入则是相反，我们要把**指向x后面的指针指向x，并且也需要更新x指向的位置**，如果能理解delete，那么理解insert其实是板上钉钉的。

来个wiki上的动图：

![Inserting elements to skip list](../.gitbook/assets/Skip\_list\_add\_element-en.gif)

{% tabs %}
{% tab title="P.S." %}
<mark style="color:purple;">**这里放一个我自己写的**</mark>  [<mark style="color:purple;background-color:blue;">**C++示例**</mark>](https://github.com/SleepyLGod/miscellaneous/tree/main/skiplist) <mark style="color:purple;background-color:blue;">****</mark> 👈
{% endtab %}
{% endtabs %}
