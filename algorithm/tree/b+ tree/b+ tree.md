- B+树的特点
  - 在B+树中节点通常被表示为一组有序的元素和子指针。如果此B+树的序数（order）为m，则除了根之外的每个节点都包含至少floor(m/2)个元素，最多m-1个元素
  - 所有叶子节点中包含了全部关键字的信息，及指向含有这些关键字记录的指针，且叶子节点本身依关键字的大小自小而大的顺序链接（B树的叶子节点并没有包含全部需要查找的信息）
- B树与B+树的区别
  - B+树种只有叶子节点会带有指向记录的指针，而B树中所有节点都带有
  - B树中内存节点出现索引项不会再出现在叶子节点中
  - B+树中所有叶子节点都是通过指针连接在一起，而B树不是
- B+树优势
  - 非叶子节点没有带有指向记录的指针，这样一个块中可以容纳更多的索引项，一是可以降低树的高度，而是内部节点可以定位到更多的叶子节点
  - 叶子节点之间通过指针来连接，且有序，能够方便进行范围扫面
- 为何B+树相比B树更适合应用在操作系统的文件索引和数据库索引上？
  - B+树磁盘读写代价更低
    - 相比B树，其内部节点没有带有指向记录的指针，因此其内存节点比B树下，这样一个块中能容纳更多的索引项，一次性读入内存中的需要查找的关键字也就越多，能减低IO操作数
  - 方便扫库
    - B树必须用中序遍历的方式按序扫库，而B+树直接从叶子节点挨个扫一遍就可以了，并且还很方便用于范围查询