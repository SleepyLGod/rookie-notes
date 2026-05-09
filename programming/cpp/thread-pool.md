---
description: >-
  C++11 implementation references: https://github.com/progschj/ThreadPool AND
  https://github.com/vit-vit/ctpl
---

# 🥰 Thread Pooling

### Introduction

A thread pool is a form of multithreaded processing. Tasks are added to a queue, and already-created worker threads take tasks from the queue and execute them. Thread-pool threads are usually background worker threads. Each worker commonly runs with the default stack size and default priority, depending on the platform and threading library.

#### **Components of a Thread Pool**

* ThreadPoolManager: creates and manages the thread pool.
* WorkThread: the worker threads inside the thread pool.
* Task interface: the interface or callable abstraction that each task must satisfy so worker threads can execute it.
* Task queue: stores tasks that have not yet been processed and provides buffering.

<mark style="color:purple;">**Idea**</mark>**: manage a task queue and a worker-thread set. Each worker repeatedly takes one task and executes it.**

**Each thread runs a scheduling function: repeatedly fetch a task, then execute it.**

### 👉 [<mark style="color:purple;">`Source Code`</mark>](https://github.com/SleepyLGod/miscellaneous/blob/main/thread%20pooling) Analysis

#### **First, the Basics**

What does mutual exclusion mean? Why is a boolean flag needed for control? What is a condition variable? See:\
[<mark style="background-color:purple;">**Producer-Consumer Model in Multithreading**</mark>](https://segmentfault.com/a/1190000024444906)\
Also see 👉 [<mark style="background-color:purple;">**Detailed Explanation of `condition_variable`**</mark>](http://www.cnblogs.com/haippy/p/3252041.html)<mark style="background-color:purple;">****</mark>

Here is a producer-consumer example:

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

// consumer thread
void consume(int n) {
    for (int i = 0; i < n; ++i) {
        std::unique_lock <std::mutex> lck(mtx);
        cv.wait(lck, shipment_available);
        std::cout << cargo << '\n';
        cargo = 0;
    }
}

int main() {
    std::thread consumer_thread(consume, 10); // consumer thread

    // The main thread is the producer thread and produces 10 items.
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

Execution result:

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

Create a thread-pool class and let the class manage resources. The class contains three public member functions and five private members. The constructor and destructor satisfy the RAII principle: Resource Acquisition Is Initialization.

* The constructor accepts a `size_t` value that represents the number of worker threads.
* **`enqueue`** is the task submission pipeline of the thread pool. It is a function template.
* **`workers_`** is a `vector` whose elements are `thread` objects, used to hold and later join worker threads.
* **`tasks_`** is the task queue in the thread pool and provides buffering.
* `queue_mutex_` is the mutex.
* `condition_` is the condition variable. The mutex, condition variable, and `stop_` flag are explained later through examples.

#### <mark style="background-color:green;">**Implementation Details**</mark>

`<thread>`: a C++11 header that mainly provides `std::thread` and related thread operations.&#x20;

`<mutex>`: a C++11 header that provides mutex-related classes, especially `std::mutex`.&#x20;

`<condition_variable>`: a C++11 header that provides condition-variable APIs commonly used in multithreaded code, such as `notify_one`, `wait`, and `wait_for`.

`<future>`: a C++11 header used to **obtain asynchronous task results and implement synchronization**. It includes facilities such as `std::async` and `std::future`.

`<functional>`: includes function-object utilities. In simple terms, it can **bind functions and arguments into callable objects**, such as through `std::bind()`.

**Constructor `ThreadPool(size_t)`**

```cpp
// the constructor just launches some amount of workers
inline ThreadPool::ThreadPool(size_t thread_number) : stop_(false) {
    for(size_t i = 0; i < thread_number; ++i) {
        workers_.emplace_back( // push into the vector
            [this] { // lambda expression capturing this
                for(;;) { // infinite loop inside the worker thread
                    std::function< void() > task;
                    { // scope controls temporary lifetime, especially the lock duration
                        std::unique_lock<std::mutex> lock(this->queue_mutex_);
                        this->condition_.wait(lock,
                            [this] {
                                // when stop_ == false and tasks_ is empty, this thread blocks
                                return this->stop_ || !this->tasks_.empty(); 
                            }); 
                        if (this->stop_ && this->tasks_.empty()) {
                            return; // stop when the pool is stopped and there is no remaining task
                        }
                        task = std::move(this->tasks_.front()); // move the front task
                        this->tasks_.pop();
                    }
                    task(); // execute the task
                }
            }
        );
    }
}
```

* Declaring the function `inline` suggests that the compiler may expand the function body directly at call sites instead of emitting a normal out-of-line call. The compiler is not required to do so.
* The lambda omits an explicit parameter list.
* **`emplace_back`** is similar to **`push_back`**, but can be more efficient for constructing objects in place. It is especially useful for objects that would otherwise incur copy or move overhead when passed as arguments.
* `workers_` receives a [<mark style="background-color:purple;">**lambda expression**</mark>](https://www.cnblogs.com/DswCnblog/p/5629165.html), that is, an anonymous function. This lambda represents the worker-thread body. It uses an infinite `for` loop. `task` represents a function object. Tasks submitted through `enqueue` eventually become these callable units. `condition.wait(lock, predicate)` blocks the thread when the predicate is `false`; a blocked thread needs `notify_one` or `notify_all` to wake it. More specifically:
  * `workers_.emplace_back([this]{...});` places the lambda into `workers_` as the body of a worker thread. The worker runs a `for(;;)` loop.
  * Inside `for(;;)`: each loop first declares `std::function<void()> task`. `task` is a function object and acts as the **smallest task unit**. Then a new scope is introduced with `{}`.
  * Inside the scope: the thread acquires the lock and checks the thread-pool state.
  * `lock(this->queue_mutex_)`: declares the lock object.
  *   `condition_.wait(lock, [this]{...})`: puts the current thread into a blocked state when the predicate is not satisfied.&#x20;

      When the predicate returns `false`, `wait()` blocks the current thread. When it returns `true`, the thread can proceed.

      In this example, the thread blocks while the pool is not stopped and the task queue is empty.&#x20;

      Then it checks whether the pool is stopped and the task queue is empty. If both are true, the worker exits the `[this]{...}` thread function; otherwise it continues.

      `std::move()` does not move by itself; it casts the task expression to an rvalue so the task can be moved rather than copied.

      Finally, it pops the first task from the `tasks_` queue.
  * After leaving the scope: execute `task()`, and the current loop iteration ends.&#x20;

**Task-queue function `enqueue(F&& f, Args&&... args)`, namely `template <class F, class... Args> auto enqueue(F&& f, Args&&... args) -> std::future<typename std::result_of<F(Args...)>::type>;`**

```cpp
// add new work item to the pool
template<class F, class... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args)  // forwarding references for a callable and its arguments
    -> std::future<typename std::result_of<F(Args...)>::type> {
    using return_type = typename std::result_of<F(Args...)>::type;
    // packaged_task is an abstraction for a task. We pass it a function to construct it,
    // then submit the task to any worker thread. The future obtained through
    // packaged_task.get_future() is used to retrieve the task result.
    auto task = std::make_shared< std::packaged_task<return_type()> > ( // smart pointer to the packaged callable
            std::bind(std::forward<F>(f), std::forward<Args>(args)...) // bind the callable and arguments
        );
    // future represents the eventual result; get_future retrieves the result handle
    std::future<return_type> res = task->get_future(); // get the future object; get() blocks if the task is not ready
    {
        std::unique_lock<std::mutex> lock(this->queue_mutex_); // preserve mutual exclusion while modifying the queue

        // don't allow enqueueing after stopping the pool
        if (this->stop_) {
            throw std::runtime_error("enqueue on stopped ThreadPool");
        }

        this->tasks_.emplace([task](){ 
            (*task)(); 
        });
        // Submit the task for worker execution.
        // std::packaged_task overloads operator(), and the overloaded operator() executes the function.
        // Therefore (*task)() can be wrapped into the queue's function<void()> task type.
    }
    this->condition_.notify_one(); // wake one waiting worker so it can acquire the lock and execute a task
    return res;
} // notify_one does not guarantee which waiting thread wakes; the predicate still protects correctness
```

* This is the syntax for a variadic function template. It is a function template, not a class template.
* The `template<>` part is `template <class F, class... Args>`, where `class... Args` means the function accepts a parameter pack.
* Return type: `auto`.&#x20;
* Function name: `enqueue`.&#x20;
* Parameter list: `(F&& f, Args&&... args)`. In this template context, these are forwarding references, not simply ordinary rvalue references.&#x20;
*   The verbose part: `-> std::future<typename std::result_of<F(Args...)>::type>`

    The `->` syntax is the trailing return type syntax introduced in C++11. It is not specific to [<mark style="background-color:purple;">**lambda expressions**</mark>](https://www.cnblogs.com/DswCnblog/p/5629165.html), although lambdas also use a similar trailing-return syntax. The part after `->` is the function's return type and is needed here because the return type depends on template parameters.

    `result_of` is used to obtain the result type of calling the function; the type is exposed through its `::type` member. In modern C++, note that `std::result_of` was deprecated in C++17 and removed in C++20; `std::invoke_result` is the replacement. This example keeps `result_of` because it is explaining a C++11 implementation.
*   In summary, this line declares a function template named `enqueue()`. Its template parameters are `class F` and the parameter pack `Args`. Its function parameters are `f` of type `F&&` and `args` of types `Args&&...`. The function returns `std::future`<mark style="color:red;">`<`</mark>`typename std::result_of`<mark style="color:green;">`<`</mark>`F(Args...)`<mark style="color:green;">`>`</mark>`::type`<mark style="color:red;">`>`</mark>.&#x20;

    This long return type can be unpacked further. `std::future` is a template from `<future>`. `std::future<A>` represents the asynchronous result of type `A`. In this code, `std::result_of<...>::type` is that `A` type. `result_of` obtains the result type of the task expression. `F(Args...)` represents calling function `F` with arguments `Args...`. Therefore, the return type of `enqueue()` is the future result type of asynchronously executing `F(Args...)`.&#x20;
* Recommended related note: [**Difference Between `typename` and `class`**](templates/typename-or-class.md)****
* Now look inside the function body:
* `using ... = typename ...;` is similar to `typedef`. It declares `return_type` as `result_of<F(Args...)>::type`, namely the return type of calling `F(Args...)`.
*   `make_shared<packaged_task<...>>(bind(...))`: another nested expression.&#x20;

    `make_shared`: allocates and constructs an object of the specified type and returns a `shared_ptr`.&#x20;

    `packaged_task`: packages a callable task. Here, the packaged task returns `return_type`.&#x20;

    `bind`: binds function `f` with arguments `args...`.&#x20;

    `forward`: preserves the original value category of `f` and `args...` during forwarding.&#x20;

    In simple terms, this line packages function `f` and its arguments `args...` into a task object for later execution.
* `res = task->get_future()`: matches the function template's return type and represents the asynchronous result of the function.
* New scope: first acquire the lock. Then perform error handling by throwing a runtime exception if the pool has stopped. Finally, insert this task into the task queue as `task{(*task)();}`.
* `condition_.notify_one()`: wakes one blocked worker thread.
* Return the asynchronous result `res`.&#x20;

**Destructor `~ThreadPool()`**

```cpp
// the distructor joins all threads
inline ThreadPool::~ThreadPool() {
    {
        std::unique_lock<std::mutex> lock(queue_mutex_);
        this->stop_ = true;
    }
    condition_.notify_all(); // wake all waiting worker threads
    for (std::thread &worker: workers_) { // for-each loop
        worker.join(); // wait for each worker thread to finish
    }
}
```

* `notify_all` wakes worker threads so they can observe `stop_` and finish any remaining work.
* `worker.join()` waits until each worker thread finishes before returning to the main thread.
