# 😁 CMU 15-445

CMU 15-445 的 BusTub 项目笔记，按实现模块组织。

建议阅读顺序：

1. [Buffer Pool](buffer-pool/README.md)：页面、frame、LRU replacer、buffer pool manager。
2. [B+ Tree Index](b-plus-tree-index/README.md)：页面布局、插入、删除、iterator、并发索引。
3. [Query Execution](query-execution.md)：catalog、executor、scan、insert/update/delete、join。
4. [Concurrency Control](concurrency-control.md)：锁管理、隔离级别、事务状态。

内容状态：这些笔记强依赖课程项目版本。遇到接口名、测试命令或锁协议细节时，应以当年 BusTub handout 和源码为准。
