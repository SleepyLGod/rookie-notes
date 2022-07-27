---
description: 线程池C++11实现
---

# 🥰 Thread Pooling

线程池是一种多线程处理形式，处理过程中将任务添加到队列，然后在创建线程后自动启动这些任务。线程池线程都是后台线程。每个线程都使用默认的堆栈大小，以默认的优先级运行，并处于多线程单元中。

#### **线程池的组成部分：**

* 线程池管理器（ThreadPoolManager）:用于创建并管理线程池
* 工作线程（WorkThread）: 线程池中线程
* 任务接口（Task）:每个任务必须实现的接口，以供工作线程调度任务的执行。
* 任务队列:用于存放没有处理的任务。提供一种缓冲机制。

管理一个任务队列，一个线程队列，然后每次取一个任务分配给一个线程去做，循环往复

让每一个 thread 都去执行调度函数：循环获取一个 task，然后执行之。

#### 👉[<mark style="color:purple;">`源码`</mark>](https://github.dev/progschj/ThreadPool)分析

通过新建一个线程池类，以类来管理资源。该类包含3个公有成员函数与5个私有成员：构造函数与析构函数即满足(RAII:Resource Acquisition Is Initialization)。

* 构造函数接受一个size\_t类型的数，表示连接数
* `enqueue`表示线程池部分中的任务管道，是一个模板函数
* **`workers`**是一个成员为thread的vector，用来监视线程状态
* **`tasks`**表示线程池部分中的任务队列，提供缓冲机制
* `queue_mutex`表示互斥锁
* `condition`表示条件变量(互斥锁，条件变量以及stop将在后面通过例子说明)

互斥到底是什么意思？为什么需要一个bool量来控制？条件变量condition又是什么？👇\
[<mark style="background-color:purple;">**多线程的生产者与消费者模型**</mark>](https://segmentfault.com/a/1190000024444906)\
同时附上👉[<mark style="background-color:purple;">**condition\_variable详解**</mark>](http://www.cnblogs.com/haippy/p/3252041.html)<mark style="background-color:purple;">****</mark>

这里来个生产者消费者的示例：

```cpp
#include <iostream>              // std::cout
#include <thread>                // std::thread, std::this_thread::yield
#include <mutex>                 // std::mutex, std::unique_lock
#include <condition_variable>    // std::condition_variable

std::mutex mtx;
std::condition_variable cv;

int cargo = 0;
bool shipment_available() {
    return cargo != 0;
}

// 消费者线程
void consume(int n) {
    for (int i = 0; i < n; ++i) {
        std::unique_lock <std::mutex> lck(mtx);
        cv.wait(lck, shipment_available);
        std::cout << cargo << '\n';
        cargo = 0;
    }
}

int main() {
    std::thread consumer_thread(consume, 10); // 消费者线程.

    // 主线程为生产者线程, 生产 10 个物品.
    for (int i = 0; i < 10; ++i) {
        while (shipment_available()) {
            std::this_thread::yield();
        }
        std::unique_lock <std::mutex> lck(mtx);
        cargo = i + 1;
        cv.notify_one();
    }

    consumer_thread.join();

    return 0;
}
```

执行结果：

```
concurrency ) ./ConditionVariable-wait 
1
2
3
4
5
6
7
8
9
10
```



**构造函数ThreadPOOL(size\_t):**

* 省略了参数
* **`emplace_back`**相当于**`push_back`**但比**`push_back`**更为高效
* wokers压入了一个lambda表达式(即一个匿名函数)，表示一个任务(线程)，使用for的无限循环，task表示函数对象，线程池中的函数接口在enqueue传入的参数之中，`condition.wait(lock, bool)`，当bool为false的时候，线程将会被堵塞挂起，被堵塞时需要`notify_one`来唤醒线程才能继续执行

**任务队列函数enqueue(F&& f, Args&&… args)**

* 这类多参数模板的格式就是如此
* \-> 尾置限定符，语法就是如此，用来推断auto类型
* [typename与class的区别](http://blog.csdn.net/zhouxuguang236/article/details/7911285)
* `result_of`用来得到返回类型的对象，它有一个成员`::type`

**析构函数\~ThreadPool()**

* 通过`notify_all`可以唤醒线程竞争任务的执行，从而使所有任务不被遗漏
