# 😩 Skiplist

### Import:

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

![小优化](<../.gitbook/assets/image (4).png>)

同理继续骚下去，如果我们继续增加节点上指针的个数，立个小目标，设置![\[公式\]](https://www.zhihu.com/equation?tex=%5Clog+n)个指针，完全可以在![\[公式\]](https://www.zhihu.com/equation?tex=%5Clog+n)的时间内完成元素的查找，这就是SkipList的精髓。

但是捏由于查找需要，我们还要保证元素的有序性。有个前提：元素添加的顺序时无序的。

跳表又引入了新的骚操作：随机深度 -> 一个节点的指针数量是随机的

![随机深度](<../.gitbook/assets/image (3).png>)

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

