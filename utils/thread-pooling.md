---
description: 线程池C++11实现
---

# 🥰 Thread Pooling

### Introduction

线程池是一种多线程处理形式，处理过程中将任务添加到队列，然后在创建线程后自动启动这些任务。线程池线程都是后台线程。每个线程都使用默认的堆栈大小，以默认的优先级运行，并处于多线程单元中。

#### **线程池的组成部分：**

* 线程池管理器（ThreadPoolManager）:用于创建并管理线程池
* 工作线程（WorkThread）: 线程池中线程
* 任务接口（Task）:每个任务必须实现的接口，以供工作线程调度任务的执行。
* 任务队列:用于存放没有处理的任务。提供一种缓冲机制。

<mark style="color:purple;">**idea**</mark>**：管理一个任务队列，一个线程队列，然后每次取一个任务分配给一个线程去做，循环往复**

**让每一个 thread 都去执行调度函数：循环获取一个 task，然后执行之。**

### 👉[<mark style="color:purple;">`源码`</mark>](https://github.dev/progschj/ThreadPool)分析

#### **首先打基础：**

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

通过新建一个线程池类，以类来管理资源。该类包含3个公有成员函数与5个私有成员：构造函数与析构函数即满足(RAII:Resource Acquisition Is Initialization)。

* 构造函数接受一个size\_t类型的数，表示连接数
* `enqueue`表示线程池部分中的任务管道，是一个模板函数
* **`workers`**是一个成员为thread的vector，用来监视线程状态
* **`tasks`**表示线程池部分中的任务队列，提供缓冲机制
* `queue_mutex`表示互斥锁
* `condition`表示条件变量(互斥锁，条件变量以及stop将在后面通过例子说明)

#### <mark style="background-color:green;">**奇淫技巧详解：**</mark>

< thread>: 是 C++ 11的新特性，主要包含了线程对象std::thread的构造。&#x20;

< mutex>: C++ 11新特性，主要包含各种Mutex的类的构造，主要是std::mutex。&#x20;

< condition\_variable>: C++ 11新特性， 包含多线程中常用的条件变量的声明，例如notify\_one、wait、wait\_for等等。

< future>: C++ 11新特性，**可以获取异步任务的结果，可用来实现同步**。包括std::sync和std::future。

< functional>: C++ 11增加了一些新特性，简单来说可以**实现函数到对象的绑定**，如bind()函数。

**构造函数ThreadPool(size\_t):**

* 声明为inline，会建议编译器把函数以直接展开的形式放入目标代码而不是以入栈调用的形式
* 省略了参数
* **`emplace_back`**相当于**`push_back`**但比**`push_back`**更为高效（更适合用来传递对象，因为它可以**避免对象作为参数被传递时**在拷贝成员上的开销）
* wokers压入了一个 <mark style="background-color:purple;"></mark>  [<mark style="background-color:purple;">**lamda表达式**</mark>](https://www.cnblogs.com/DswCnblog/p/5629165.html)  <mark style="background-color:purple;">****</mark>  (即一个匿名函数)，表示一个任务(线程)，使用for的无限循环，task表示函数对象，线程池中的函数接口在enqueue传入的参数之中，`condition.wait(lock, bool)`，当bool为false的时候，线程将会被堵塞挂起，被堵塞时需要`notify_one`来唤醒线程才能继续执行，详细说就是：
  * `workers.emplace_back([this]{…});`在本代码中的lambda表达式是作为一个线程放入workers\[]中。 这个线程是个for(;;)循环。
  * `for(;;)`里面: 每次循环首先声明一个`std::function< void()> task`，task是一个可以被封装成对象的函数，在此作为**最小任务单位**。然后用{}添加了一个作用域。
  * 作用域里面: 在这个作用域中进行了一些线程上锁和线程状态的判断。
  * `lock(this->queue_mutex)`: 声明上锁原语
  *   `condition.wait(lock, [this]{…})`: 使当前线程进入阻塞状态： 当第二个参数为false时，wait()会阻塞当前线程，为true时解除阻塞；

      在本例中的条件就是，当线程池运行或者任务列表为空时，线程进入阻塞态。 然后判断，如果线程池运行或者任务列表为空则继续后续操作，否则退出这个`[this]{…}`线程函数。 `std::move()`是移动构造函数，相当于效率更高的拷贝构造函数。最后将`tasks[]`任务队列的第一个任务出栈。
  * 离开作用域: 然后执行task()，当前一轮循环结束。&#x20;

**任务队列函数enqueue(F&& f, Args&&… args) 即 `template < class F, class… Args> auto enqueue(F&& f, Args&&… args) -> std::future< typename std::result_of< F(Args…)>::type>;`**

* 这类多参数模板的格式就是如此首先，这是一个函数模板，而不是类模板
* template<> 部分: `template < class F, class… Args>` 其中 `class… Args`代表接受多个参数
* 返回类型: `auto`&#x20;
* 函数名： `enqueue`&#x20;
* 形参表： `(F&& f, Args&&… args)`。&&是C++ 11新特性，代表右值引用。&#x20;
*   不明觉厉: `-> std::future< typename std::result_of< F(Args…)>::type>`

    这个`->`符号其实用到了C++ 11中的 <mark style="background-color:purple;"></mark>  [<mark style="background-color:purple;">**lamda表达式**</mark>](https://www.cnblogs.com/DswCnblog/p/5629165.html)  <mark style="background-color:purple;">****</mark>  ，后面的内容代表函数的返回类型。（-> 尾置限定符，语法就是如此，用来推断auto类型）

    `result_of`用来得到返回类型的对象，它有一个成员`::type`
*   总的来说就是，这句话声明了一个名为`enqueue()`的函数模板，它的模板类型为`class F`以及多个其他类型`Args`，它的形参是一个F&&类型的f以及多个Args&&类型的args，最后这个函数返回类型是`std::future`<mark style="color:red;">`<`</mark>`typename std::result_of`<mark style="color:green;">`<`</mark>`F(Args…)`<mark style="color:green;">`>`</mark>`::type`<mark style="color:red;">`>`</mark>。&#x20;

    对于这个冗长的返回类型，又可以继续分析： `std::future`在前面提到过了，它本身是一个模板，包含在 `< future>`中。通过`std::future`可以返回这个`A`类型的异步任务的结果。 `std::result_of::type`就是这段代码中的A类型。`result_of`获取了someTask的执行结果的类型。 `F(Args…)`就是这段代码的someTask，即函数`F(Args…)`。 所以最后这个模板函数`enqueue()`的返回值类型就是`F(Args…)`的异步执行结果类型。&#x20;
* 推荐看看👉 [**typename与class的区别**](typename-or-class.md)****
* 看看函数体内：
* `using … = typename …;` 功能类似typedef。将`return_type`声明为一个`result_of< F(Args…)>::type`类型，即函数F(Args…)的返回值类型。
*   `make_shared < packaged_task < >>(bind())`: 又是复杂的嵌套。&#x20;

    `make_shared` : 开辟()个类型为<>的内存&#x20;

    `packaged_task` : 把任务打包,这里打包的是return\_type&#x20;

    `bind` : 绑定函数f, 参数为args…&#x20;

    `forward` : 使()转化为<>相同类型的左值或右值引用&#x20;

    简单来说，这句话相当于把函数f和它的参数args…打包为一个模板内定义的task，便于后续操作。
* `res = task->get_future()`: 与模板函数的返回类型一致，是函数异步的执行结果。
* 新作用域: 先是一个加锁原语`lock()`。 然后是个异常处理，如果停止的话抛出一个运行时异常。 最后，向任务列表插入这个任务`task{(*task)();}`。
* `condition.notify_one()`: 解除一个正在等待唤醒的线程的阻塞态。
* 返回异步结果res&#x20;

**析构函数\~ThreadPool()**

* 通过`notify_all`可以唤醒线程竞争任务的执行，从而使所有任务不被遗漏
* 当所有线程执行完毕时返回主线程`worker.join()`
