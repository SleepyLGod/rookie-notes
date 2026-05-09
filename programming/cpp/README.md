# C++

This section collects notes on C++ language features, resource management, concurrency tools, and common data structures.

Suggested reading order:

1. [typename or class?](templates/typename-or-class.md), [RAII](raii.md), [Smart Pointers](smart-pointers.md): start with types, resource management, and smart pointers.
2. [Parallelism and Concurrency](concurrency.md), [Thread Pooling](thread-pool.md): then read concurrency fundamentals and thread pools.
3. [Lock V1](locks-cpp.md), [Lock V2](locks-pthread.md): supplement with C++/pthread locks and condition variables.
4. [Skiplist](skiplist.md), [Miscellaneous Of C++](miscellaneous.md): use these as additional topic notes.

Content status: these are mostly learning notes and local source-code examples. For concurrency, memory model, and standard library behavior, use the C++ standard and the target platform implementation as the source of truth.
