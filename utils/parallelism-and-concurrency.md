---
description: https://changkun.de/modern-cpp/en-us/07-thread/#7-1-Basic-of-Parallelism
---

# 🤣 Parallelism and Concurrency

### Basic

`std::thread` 用于创建一个执行的线程实例，所以它是一切并发编程的基础，使用时需要包含 `<thread>` 头文件， 它提供了很多基本的线程操作，例如 `get_id()` 来获取所创建线程的线程 ID，使用 `join()` 来加入一个线程等等，例如：

```cpp
#include <iostream>
#include <thread>

int main() {
    std::thread t([](){
        std::cout << "hello world." << std::endl;
    });
    t.join();
    return 0;
}
```

### Mutex and Critical Section

