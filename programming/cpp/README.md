# C++

这一节收集 C++ 语言特性、资源管理、并发工具和常用数据结构笔记。

建议阅读顺序：

1. [typename or class?](templates/typename-or-class.md)、[RAII](raii.md)、[Smart Pointers](smart-pointers.md)：先看类型、资源管理和智能指针。
2. [Parallelism and Concurrency](concurrency.md)、[Thread Pooling](thread-pool.md)：再看并发基础和线程池。
3. [Lock V1](locks-cpp.md)、[Lock V2](locks-pthread.md)：补充 C++/pthread 锁与条件变量。
4. [Skiplist](skiplist.md)、[Miscellaneous Of C++](miscellaneous.md)：最后作为专题补充。

内容状态：以学习札记和局部源码示例为主。涉及并发、内存模型和标准库行为时，以 C++ 标准和目标平台实现为准。
