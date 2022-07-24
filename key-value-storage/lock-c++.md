# 😇 Lock(c++ API)

## **互斥锁（Mutex）**

为了避免多个线程在某一时刻同时操作一个共享资源

例如线程池中的有多个空闲线程和一个任务队列, 任何一个线程都要使用互斥锁互斥访问任务队列，以避免多个线程同时访问任务队列以发生错乱。

在某一时刻，只有一个线程可以获取互斥锁，在释放互斥锁之前其他线程都不能获取该互斥锁。如果其他线程想要获取这个互斥锁，那么这个线程只能以阻塞方式进行等待。

构造`std::mutex`的实例创建互斥元，调用成员函数`lock()`来锁定它，调用`unlock()`来解锁

不过一般不推荐这种做法，标准C++库提供了`std::lock_guard`类模板，实现了互斥元的`RAII`惯用语法。`std::mutex`和`std::lock _ guard`

```cpp
#include <mutex>
#include <list> // 互斥元保护列表

std::list<int> this_list;
std::mutex this_mutex;

void add_to_list (int value) {
    std::lock_guard<std::mutex> guard(this_mutex);
    this_list.push_back(value);
}
```

这里注意死锁：多个线程争夺共享资源导致每个线程都不能取得自己所需的全部资源，从而程序无法向下执行

* 互斥（资源同一时刻只能被一个进程使用）
* 请求并保持（进程在请资源时，不释放自己已经占有的资源）
* 不剥夺（进程已经获得的资源，在进程使用完前，不能强制剥夺）
* 循环等待（进程间形成环状的资源循环等待关系）

#### **直接操作 mutex，即直接调用 mutex 的 lock / unlock 函数**

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

std::mutex mutex_1;
int count_1 = 0;

void counter() {
    mutex_1.lock();
    
    int i = ++count_1;
    ……
    
    mutex_1.unlock();
}

int main() {
    const std::size_t SIZE = 4;
    // create a group of threads
    std::vector<std::thread> v;
    v.reverse(SIZE);
    for (std::size_t i = 0; i < SIZE; ++i) {
        v.emplace_back(&counter);
    }
    
    // waiting for the end of all threads
    for (std::thread& t : v) {
        t.join();
    }
    return 0;
}
```

#### **lock\_guard**

使用 `lock_guard` 自动加锁、解锁。原理是 RAII，和智能指针类似。

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

std::mutex mutex_2;
int count_2 = 0;

void counter() {
    // lock_guard 在构造函数里加锁，在析构函数里解锁。
    std::lock_guard<std::mutex> lock(mutex_2);
    int i = ++count_2;
    ……
}

int main() {
    const std::size_t SIZE = 4;
    std::vector<std::thread> v;
    v.reverse(SIZE);
    for (std::size_t i = 0; i < SIZE; ++i) {
        v.emplace_back(&counter);
    }
    
    // waiting for the end of all threads
    for (std::thread& t : v) {
        t.join();
    }
    return 0;
}
}
```

#### **unique\_lock**

使用 `unique_lock` 自动加锁、解锁。 `unique_lock` 与 `lock_guard` 原理相同，但是提供了更多功能（比如可以结合条件变量使用）。 注意：`mutex::scoped_lock` 其实就是 `unique_lock` 的 `typedef`!

`counter` 函数体：

```cpp
void counter() {
    std::unique_lock<std::mutex> lock(mutex_3);
    int i = ++count_3;
    ……
}
```

#### **std::recursive\_mutex**

就像互斥锁（`mutex`）一样，递归互斥锁（`recursive_mutex`）是可锁定的对象，但它**允许同一线程获得对互斥锁对象的多级所有权（多次lock）**。

这允许从已经锁定它的线程锁定（或尝试锁定）互斥对象，从而获得对互斥对象的新所有权级别：

**互斥对象实际上将保持对该线程的锁定，直到调用其成员 unlock 的次数与此所有权级别的次数相同**。

1. 调用线程从成功调用 lock 或 try\_lock 开始占有recursive\_mutex， 期间线程可以进行对 lock 或 try\_lock的附加调用，所有权在线程调用 unlock 匹配次数时结束。
2. 线程占有recursive\_mutex时，若其他线程要求recursive\_mutex所有权，调用lock将被阻塞，调用try\_lock将返回false.
3. 可锁定recursive\_mutex的最大次数未指定的，但到达该数后，对 lock 的调用将抛出 std::system\_error 而对 try\_lock 的调用返回false;
4. 若recursive\_mutex在仍被线程占有时被销毁，则程序行为未定义。recursive\_mutex满足 mutex 和 标准布局类型的所有要求。

```cpp
#include <iostream>
#include <thread>
#include <mutex> 

std::recursive_mutex mtx;           

void print_block (int n, char c) {
  mtx.lock();
  mtx.lock();
  mtx.lock();
  
  for (int i=0; i<n; ++i) { 
      std::cout << c; 
  }
  std::cout << '\n';
  
  mtx.unlock();
  mtx.unlock();
  mtx.unlock();
}

int main () {
  std::thread th1 (print_block,50,'*');
  std::thread th2 (print_block,50,'$');

  th1.join();
  th2.join();

  return 0;
}
```

#### **std::timed\_mutex**

定时互斥锁是一个可时间锁定的对象，旨在通知何时关键代码需要独占访问，就像常规互斥锁一样，但还支持定时尝试锁定请求。

| lock             | 调用线程将锁定timed\_mutex，并在必要时进行阻塞（其行为与 mutex 完全相同） |
| ---------------- | ---------------------------------------------- |
| try\_lock        | 调用线程将锁定timed\_mutex，并在必要时进行阻塞（其行为与 mutex 完全相同） |
| try\_lock\_for   | 尝试锁定 timed\_mutex， 最多阻塞 rel\_time 时间           |
| try\_lock\_until | 尝试锁定 timed\_mutex，最多阻塞到 abs\_time 时间点          |
| unlock           | 解锁 timed\_mutex，释放对其的所有权（其行为与 mutex 相同）        |

#### **std::recursive\_timed\_mutex**

递归定时互斥锁将 `recursive_timed` 和 `timed_mutex` 的功能结合到一个类中：

* 它既支持通过单个线程获取多个锁定级别
* 又支持定时的 try\_lock 请求。

成员函数与 `timed_mutex` 相同。

#### **once\_flag、call\_once使用**

在多线程中，有一种场景是**某个任务只需要执行一次**，可以用C++11中的`std::call_once`函数配合`std::once_flag`来实现。

**多个线程同时调用某个函数**，**`std::call_once`可以保证多个线程对该函数只调用一次**

实现**线程安全的单例模式**

```cpp
//h文件
#pragma once
#include <thread>
#include <iostream>
#include <mutex>
#include <memory>

class Task {
private:
	Task();
public:
	static Task* task;
	static Task* getInstance();
	void fun();
};
```

```cpp
// cpp文件
Task* Task::task;
Task::Task() {
	std::cout << "构造函数" << std::endl;
}

Task* Task::getInstance() {
	static std::once_flag flag;
	std::call_once(flag, []	{
		task = new Task();
	});
	return task;
}

void Task::fun() {
	std::cout << "hello world!"<< std::endl;
}
```

## **条件锁**

条件锁就是所谓的条件变量, 不是用来管理互斥量的，它的作用是用来同步线程，它的用法相当于编程中常见的flag标志（A、B两个人约定flag=true为行动号角，默认flag为false,A不断的检查flag的值,只要B将flag修改为true，A就开始行动）

某一个线程因为某个条件未满足时，可以使用条件变量使改程序处于阻塞状态。

一旦条件满足，则以“信号量”的方式唤醒一个因为该条件而被阻塞的线程。

最为常见就是在线程池中，起初没有任务时任务队列为空，此时线程池中的线程因为“任务队列为空”这个条件处于阻塞状态。一旦有任务进来，就会以信号量的方式唤醒一个线程来处理这个任务。

类型：

* `std::condition_variable`（只和`std::mutex`一起工作）
* `std::condition_variable_any`（符合类似互斥元的最低标准的任何东西一起工作）

```cpp
// std::condition_variable waiting for data
#include <condition_variable>
#include <mutex>
#include <queue>
……

std::mutex mut;
std::queue<data_chunck> data_queue;
std::condition_variable data_con;

void data_preparing_thread () {
    while (more_data_to_prepare()) {
        data_chunck const data = prepare_data();
        std::lock_guard<std::mutex> lk(mut);
        data_queue.push(data);
        data_con.notify_one();
    }
}

void data_processing_thread () {
    while (true) {
        std::unique_lock<std::mutex> lk(mut); //这里使用unique_lock是为了后面方便解锁
        data.con.wait(lk, {
            []return !data_queue.empty();
        });
        data_chunck data = data_queue.front();
        data_queue.pop();
        lk.unlock();
        process(data);
        if (is_last_chunck(data)) {
            break;
        }
    }
}
```

**eg 2:**

```cpp
#include <iostream>
#include <thread>
#include <string>
#include <mutex>
#include <condition_variable>
#include <deque>
#include <chrono>

std::deque<int> q;
std::mutex mu;
std::condition_variable condi;

void function_1() {
	int count = 10;
	while (count > 0) {
		std::unique_lock<std::mutex> locker(mu);
		q.push_back(count);
		locker.unlock();
		condi.notify_one();			// 通知一个等待线程激活   condi.notify_all()激活所有线程
		count--;
		std::this_thread::sleep_for(std::chrono::seconds(1));
	}
}

void function_2() {
	int data = 100;
	while (data > 1) {
		std::unique_lock<std::mutex> locker(mu);
		condi.wait(locker,			// 解锁locker,并进入休眠  收到notify时又重新加锁
			[]() { return !q.empty(); });   // 如果q不为空 线程才会被激活
		data = q.front();
		q.pop_front();
		locker.unlock();

		std::cout << data << std::endl;
	}
}
int main() {
	std::thread t1(function_1);
	std::thread t2(function_2);

	t1.join();
	t2.join();
	
	return 0;
}
```

`cond.notify_one()`: 随机唤醒一个等待的线程

`cond.notify_all()`: 唤醒所有等待的线程

`wait()`的实现: 检查条件，并在满足时返回。

两个重载：

```cpp
void wait( std::unique_lock<std::mutex>& lock );                  //  (1)	(since C++11)

template< class Predicate >
void wait( std::unique_lock<std::mutex>& lock, Predicate pred );  //  (2)	(since C++11)
```

如果条件不满足，`wait()`解锁互斥元，并将该线程置于阻塞或等待状态。

当来自数据准备线程中对`notify_one()`的调用通知条件变量时，线程从睡眠状态中苏醒（解除其阻塞），重新获得互斥元上的锁，并再次检查条件，如果条件已经满足，就从`wait()`返回值，互斥元仍被锁定。如果条件不满足，该线程解锁互斥元，并恢复等待。

*   `void wait( std::unique_lockstd::mutex& lock )`

    先unlock之前获得的mutex，然后阻塞当前的执行线程。

    把当前线程添加到等待线程列表中，该线程会持续 block 直到被 notify\_all() 或 notify\_one() 唤醒。

    被唤醒后，该thread会重新获取mutex，获取到mutex后执行后面的动作。

    线程block时候也可能被意外或者错误唤醒。
*   `template< class Predicate > void wait( std::unique_lockstd::mutex& lock, Predicate pred );`

    该重载设置了第二个参数 Predicate， 只有当pred为false时，wait才会阻塞当前线程。

    该情况下，线程被唤醒后，先重新判断pred的值。

    如果pred为false，则会释放mutex并重新阻塞在wait。

    因此，该mutex必须有pred的权限。该重载消除了意外唤醒的影响。

如果等待线程只打算等待一次，那么当条件为`true`时它就不会再等待这个条件变量了，

条件变量未必是同步机制的最佳选择。如果等待的条件是一个特定数据块的可用性时，这尤其正确。在这个场景中，使用期值（future）更合适。使用future等待一次性事件。

## **自旋锁**

假设我们有一个两个处理器core1和core2计算机，现在在这台计算机上运行的程序中有两个线程：T1和T2分别在处理器core1和core2上运行，两个线程之间共享着一个资源。

首先我们说明互斥锁的工作原理，互斥锁是是一种sleep-waiting的锁。假设线程T1获取互斥锁并且正在core1上运行时，此时线程T2也想要获取互斥锁（pthread\_mutex\_lock），但是由于T1正在使用互斥锁使得T2被阻塞。当T2处于阻塞状态时，T2被放入到等待队列中去，处理器core2会去处理其他任务而不必一直等待（忙等）。也就是说处理器不会因为线程阻塞而空闲着，它去处理其他事务去了。

而自旋锁就不同了，自旋锁是一种busy-waiting的锁。也就是说，如果T1正在使用自旋锁，而T2也去申请这个自旋锁，此时T2肯定得不到这个自旋锁。

与互斥锁相反的是，此时运行T2的处理器core2会一直不断地循环检查锁是否可用（自旋锁请求），直到获取到这个自旋锁为止。

从“自旋锁”的名字也可以看出来，如果一个线程想要获取一个被使用的自旋锁，那么它会一致占用CPU请求这个自旋锁使得CPU不能去做其他的事情，直到获取这个锁为止，这就是“自旋”的含义。

当发生阻塞时，互斥锁可以让CPU去处理其他的任务；而自旋锁让CPU一直不断循环请求获取这个锁。通过两个含义的对比可以我们知道“自旋锁”是比较耗费CPU的。

```cpp
// use std::atomic_flag
class spinlock_mutex {
    std::atomic_flag flag;
    public:
    spinlock_mutex(): flag(ATOMIC_FLAG_INIT) {
    }
    void lock() {
        while (flag.test_and_set(std::memory_order_aquire));
    }
    void unlock() {
        flag.clear(std::memory_order_release);
    }
}
```

## **读写锁**

我们允许在数据库上同时执行多个“读”操作，但是某一时刻只能在数据库上有一个“写”操作来更新数据。这就是一个简单的读者-写者模型。

头文件：`boost/thread/shared_mutex.cpp` 类型：`boost::shared_lock`

提供两种访问权限的控制：共享性（shared）和排他性（exclusive）。

通过`lock / try_lock`获取排他性访问权限，通过`lock_shared / try_lock_shared`获取共享性访问权限。

这样的设置对于区分不同线程的读写操作特别有用。`shared_mutex`是`c++17`中引入的，使用时需要注意编译器版本。

```cpp
#include <iostream>
#include <mutex>  // For std::unique_lock
#include <shared_mutex>
#include <thread>

class ThreadSafeCounter {
 public:
  ThreadSafeCounter() = default;

  // Multiple threads/readers can read the counter's value at the same time.
  unsigned int get() const {
    std::shared_lock lock(mutex_);
    return value_;
  }

  // Only one thread/writer can increment/write the counter's value.
  void increment() {
    std::unique_lock lock(mutex_);
    value_++;
  }

  // Only one thread/writer can reset/write the counter's value.
  void reset() {
    std::unique_lock lock(mutex_);
    value_ = 0;
  }

 private:
  mutable std::shared_mutex mutex_;
  unsigned int value_ = 0;
};


int main() {
  ThreadSafeCounter counter;

  auto increment_and_print = [&counter]() {
    for (int i = 0; i < 3; i++) {
      counter.increment();
      std::cout << std::this_thread::get_id() << ' ' << counter.get() << '\n';

      // Note: Writing to std::cout actually needs to be synchronized as well
      // by another std::mutex. This has been omitted to keep the example small.
    }
  };

  std::thread thread1(increment_and_print);
  std::thread thread2(increment_and_print);

  thread1.join();
  thread2.join();
}
```
