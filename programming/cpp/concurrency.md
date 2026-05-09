---
description: https://changkun.de/modern-cpp/en-us/07-thread/#7-1-Basic-of-Parallelism
---

# 🤣 Parallelism and Concurrency

### Basic

`std::thread` is used to create a thread of execution, so it is the foundation of C++ concurrent programming. To use it, include the `<thread>` header. It provides many basic thread operations, such as `get_id()` for obtaining the ID of the created thread and `join()` for joining a thread. For example:

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

`std::mutex` is the most basic mutex class in C++11. Instantiating `std::mutex` creates a mutex. Its member function `lock()` locks the mutex, and `unlock()` unlocks it. In real code, however, it is better not to call these member functions directly, because then every exit path of the critical section must call `unlock()`, including exception paths.

For this reason, C++11 provides the RAII-style template class `std::lock_guard` for mutexes. RAII keeps code concise while providing good exception safety.

With RAII, the mutex guard for a critical section only needs to be created at the beginning of the scope:

```cpp
#include <iostream>
#include <mutex>
#include <thread>

int v = 1;

void critical_section(int change_v) {
    static std::mutex mtx;
    std::lock_guard<std::mutex> lock(mtx);

    // Execute the competing operation.
    v = change_v;

    // mtx is released after leaving this scope.
}

int main() {
    std::thread t1(critical_section, 2), t2(critical_section, 3);
    t1.join();
    t2.join();

    std::cout << v << std::endl;
    return 0;
}
```

Because C++ guarantees that stack objects are destroyed when their lifetime ends, this code is exception-safe. Whether `critical_section()` returns normally or throws an exception in the middle, stack unwinding destroys the guard and calls `unlock()` automatically.

**`std::unique_lock`** is a more flexible lock-management tool than `std::lock_guard`. A `std::unique_lock` object manages lock and unlock operations on a `mutex` with exclusive ownership: no other `unique_lock` object owns that mutex at the same time. For simple scoped locking, prefer `std::lock_guard`. Use `std::unique_lock` when delayed locking, manual `unlock`, moving the lock object, or use with a condition variable is required.

`std::lock_guard` cannot explicitly call `lock` or `unlock`, while `std::unique_lock` can call them at arbitrary points after construction. This can reduce the locked scope and provide higher concurrency.

If you use `std::condition_variable::wait`, you must use `std::unique_lock` as the argument.

### Future

A future, represented by `std::future`, provides a way to access the result of an asynchronous operation. This sentence is not immediately intuitive. To understand this feature, first consider multi-threaded behavior before C++11.

Suppose main thread A wants to create a new thread B to execute an expected task and return a result. At that moment, thread A may be busy with other work and cannot care about B's result immediately. Naturally, we want to obtain thread B's result at some specific later time.

Before C++11 introduced `std::future`, a common approach was to create a thread, start task B inside it, send an event when it was ready, and store the result in a global variable. The main thread continued doing other work and called a thread-waiting function when it needed the result.

`std::future` in C++11 simplifies this process and can be used to obtain the result of an asynchronous task. Naturally, it can also serve as a simple thread-synchronization mechanism, namely a **barrier**.

To see an example, use `std::packaged_task`. It can wrap any callable target and can therefore be used to implement asynchronous calls:

```cpp
#include <iostream>
#include <future>
#include <thread>

int main() {
    // Wrap a lambda expression that returns 7 into task.
    // The template parameter of std::packaged_task is the function type to wrap.
    std::packaged_task<int()> task([](){return 7;});
    // Obtain the future of task.
    std::future<int> result = task.get_future(); // Execute task in a thread.
    std::thread(std::move(task)).detach(); // move: move constructor
    std::cout << "waiting...";
    result.wait(); // Set a barrier here and block until the future is ready.
    // Print the execution result.
    std::cout << "done!" << std:: endl << "future result is "
              << result.get() << std::endl;
    return 0;
}
```

After wrapping the target to call, use `get_future()` to obtain a `std::future` object for later thread synchronization.

### Condition Variable

`std::condition_variable` lets a thread block while a condition has not yet been satisfied and be awakened after another thread modifies the state. Its core value is avoiding busy-waiting and correctly handling the synchronization pattern "wait until a condition changes"; it is not specifically designed to "solve deadlocks". `std::condition_variable::notify_one()` wakes one waiting thread, while `notify_all()` notifies all waiting threads. The following is a producer-consumer example:

```cpp
#include <queue>
#include <chrono>
#include <mutex>
#include <thread>
#include <iostream>
#include <condition_variable>


int main() {
    std::queue<int> produced_nums;
    std::mutex mtx;
    std::condition_variable cv;
    bool notified = false;  // notification flag

    // Producer
    auto producer = [&]() {
        for (int i = 0; ; i++) {
            std::this_thread::sleep_for(std::chrono::milliseconds(900));
            std::unique_lock<std::mutex> lock(mtx);
            std::cout << "producing " << i << std::endl;
            produced_nums.push(i);
            notified = true;
            cv.notify_all(); // notify_one could also be used here.
        }
    };
    // Consumer
    auto consumer = [&]() {
        while (true) {
            std::unique_lock<std::mutex> lock(mtx);
            while (!notified) {  // avoid spurious wakeups
                cv.wait(lock);
            }
            // Temporarily release the lock so the producer can keep producing
            // before the consumer drains the queue.
            lock.unlock();
            // Consumer is slower than producer.
            std::this_thread::sleep_for(std::chrono::milliseconds(1000));
            lock.lock();
            while (!produced_nums.empty()) {
                std::cout << "consuming " << produced_nums.front() << std::endl;
                produced_nums.pop();
            }
            notified = false;
        }
    };

    // Run in separate threads.
    std::thread p(producer);
    std::thread cs[2];
    for (int i = 0; i < 2; ++i) {
        cs[i] = std::thread(consumer);
    }
    p.join();
    for (int i = 0; i < 2; ++i) {
        cs[i].join();
    }
    return 0;
}
```

It is worth noting that although `notify_one()` could be used in the producer, it is not recommended in this specific example. With multiple consumers, the consumer implementation simply releases the lock for a while, allowing other consumers to compete for it and better exploit concurrency between consumers. That said, because `std::mutex` is exclusive, we still cannot expect multiple consumers to truly consume the queue contents in parallel. Finer-grained techniques are still needed.

### Atomic Operations and the Memory Model

Careful readers may wonder whether the producer-consumer example in the previous section could be broken by compiler optimization. For example, the boolean `notified` is not marked `volatile`, so the compiler might optimize it, perhaps keeping it as a register value and causing the consumer thread to never observe changes to it. This is a good question. To explain it clearly, we need to discuss the memory model introduced in C++11. First look at this question: what does the following code print?

```cpp
#include <thread>
#include <iostream>

int main() {
    int a = 0;
    int flag = 0;

    std::thread t1([&]() {
        while (flag != 1);

        int b = a;
        std::cout << "b = " << b << std::endl;
    });

    std::thread t2([&]() {
        a = 5;
        flag = 1;
    });

    t1.join();
    t2.join();
    return 0;
}
```

Intuitively, `a = 5;` in `t2` appears to execute before `flag = 1;`, and `while (flag != 1)` in `t1` appears to ensure that `std::cout << "b = " << b << std::endl;` will not execute before the flag changes. Logically, `b` seems like it should be 5. In reality, the situation is much more complex. More precisely, this program has undefined behavior, because `a` and `flag` are read and written from two parallel threads without synchronization, creating data races. In addition, even if we ignored the racing reads and writes, CPU out-of-order execution and compiler instruction reordering could still cause `a = 5` to occur after `flag = 1`, so `b` might print 0.

#### Atomic Operations

`std::mutex` can solve the concurrent read/write problem above, but a mutex is an operating-system-level facility. A typical mutex implementation involves two basic principles:

1. It provides an automatic inter-thread state transition, namely the "locked" state.
2. It ensures that memory operated on while holding the mutex is isolated from memory outside the critical section during mutex operations.

These are very strong synchronization conditions. In other words, once compiled into CPU instructions, they may translate into many instructions. We will later look at how to implement a simple mutex. For a variable that only needs an atomic operation, meaning it has no intermediate state, this seems too heavy.

Synchronization conditions have been studied for a very long time, and this note does not expand on that history. Readers should understand that modern CPU architectures provide instruction-level atomic operations. Therefore, C++11 introduced the `std::atomic` template for shared variable reads and writes under multi-threading. It lets us instantiate an atomic type and reduce an atomic read/write operation from a group of instructions to a single CPU-level atomic operation when the platform supports it. For example:

```cpp
std::atomic<int> counter;
```

For atomic integer or floating-point types, C++ provides basic numeric member functions such as `fetch_add` and `fetch_sub`. It also overloads operators to conveniently provide corresponding `+` and `-` forms. For example:

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> count = {0};

int main() {
    std::thread t1([](){
        count.fetch_add(1);
    });
    std::thread t2([](){
        count++;        // equivalent to fetch_add
        count += 1;     // equivalent to fetch_add
    });
    t1.join();
    t2.join();
    std::cout << count << std::endl;
    return 0;
}
```

Of course, not every type can provide lock-free atomic operations. Whether atomic operations are feasible depends on the specific CPU architecture and whether the instantiated type layout satisfies that architecture's memory-alignment requirements. Therefore, we can use `std::atomic<T>::is_lock_free` to check whether an atomic type is lock-free, for example:

```cpp
#include <atomic>
#include <iostream>

struct A {
    float x;
    int y;
    long long z;
};

int main() {
    std::atomic<A> a;
    std::cout << std::boolalpha << a.is_lock_free() << std::endl;
    return 0;
}
```

#### Consistency Models

At a coarse level, multiple parallel threads can be viewed roughly as a distributed system. In a distributed system, any communication and even local operation takes some time, and communication can even be unreliable.

If we force operations on a variable `v` across multiple threads to be atomic, and require every other thread to **synchronously** observe changes to `v` as soon as any thread finishes operating on it, then for `v`, the program behaves as if it executed sequentially. It gains no efficiency from introducing multiple threads. How can this be accelerated appropriately? The answer is to weaken synchronization conditions between threads for atomic operations.

In principle, each thread can correspond to a cluster node, and communication between threads is almost equivalent to communication between cluster nodes. When weakening inter-thread synchronization conditions, we usually consider four different consistency models:

<mark style="background-color:green;">**Linearizability**</mark>: also called strong consistency or atomic consistency. It requires every read operation to read the most recent write of the data, and the operation order of all threads must be consistent with global-clock order.

```
        x.store(1)      x.load()
T1 ---------+----------------+------>


T2 -------------------+------------->
                x.store(2)
```

In this situation, the two writes to `x` by threads `T1` and `T2` are atomic. `x.store(1)` strictly happens before `x.store(2)`, and `x.store(2)` strictly happens before `x.load()`. It is worth noting that the global-clock requirement of linearizability is difficult to implement. This is why people continue to study algorithms under weaker consistency conditions.

<mark style="background-color:green;">**Sequential consistency**</mark>: this also requires **every read operation** to read the most recent write of the data, but it does not require consistency with global-clock order.

```
        x.store(1)  x.store(3)   x.load()
T1 ---------+-----------+----------+----->


T2 ---------------+---------------------->
              x.store(2)

or

        x.store(1)  x.store(3)   x.load()
T1 ---------+-----------+----------+----->


T2 ------+------------------------------->
      x.store(2)
```

Under sequential consistency, `x.load()` must read the latest written data. Therefore, there is no ordering guarantee between `x.store(2)` and `x.store(1)`; it is only required that `T2`'s `x.store(2)` happen before `x.store(3)`.

<mark style="background-color:green;">**Causal consistency**</mark>: the requirement is further weakened. Only the order of causally related operations must be preserved. Operations without causal relationship do not have an ordering requirement.

```
      a = 1      b = 2
T1 ----+-----------+---------------------------->


T2 ------+--------------------+--------+-------->
      x.store(3)         c = a + b    y.load()

or

      a = 1      b = 2
T1 ----+-----------+---------------------------->


T2 ------+--------------------+--------+-------->
      x.store(3)          y.load()   c = a + b

or

     b = 2       a = 1
T1 ----+-----------+---------------------------->


T2 ------+--------------------+--------+-------->
      y.load()            c = a + b  x.store(3)

```

All three examples above are causally consistent, because in the whole process only `c` depends on `a` and `b`, while `x` and `y` appear unrelated in this example. In real situations, however, more detailed information is needed to determine whether `x` and `y` are truly unrelated.

<mark style="background-color:green;">**Eventual consistency**</mark>: this is the **weakest** consistency requirement. It only guarantees that an operation will be observed at some future time, without specifying when it will be observed. We can strengthen this condition slightly, for example by requiring that the time for an operation to be observed is always bounded. But that is outside the scope of this discussion.

```
    x.store(3)  x.store(4)
T1 ----+-----------+-------------------------------------------->


T2 ---------+------------+--------------------+--------+-------->
         x.read      x.read()           x.read()   x.read()
```

In the situation above, if we assume the initial value of `x` is 0, the four `x.read()` results in `T2` may include, but are not limited to, the following:

| <pre><code>3 4 4 4 // writes to x are observed quickly
0 3 3 4 // observing writes to x has some delay
0 0 0 4 // the final read observes the final value of x,
        // but earlier changes were not observed
0 0 0 0 // no writes to x are observed in the current time period,
        // but at some future time x must be observed as 4
</code></pre> |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

#### Memory Ordering

To pursue maximum performance and implement consistency requirements of different strengths, C++11 defines six different **memory order** options under `std::memory_order` for atomic operations. These express four synchronization models between threads:

<mark style="background-color:green;">**Relaxed model**</mark>: under this model, atomic operations within a single thread execute in order and instruction reordering is not allowed for that thread's atomic operations, but the order of atomic operations between **different threads** is arbitrary. This is specified with `std::memory_order_relaxed`. Consider this example:

```cpp
std::atomic<int> counter = {0};
std::vector<std::thread> vt;
for (int i = 0; i < 100; ++i) {
    vt.emplace_back([&](){
        counter.fetch_add(1, std::memory_order_relaxed);
    });
}

for (auto& t : vt) {
    t.join();
}
std::cout << "current counter:" << counter << std::endl;
```

<mark style="background-color:green;">**Release/consume model**</mark>: in this model, we begin to constrain operation order between threads. If one thread modifies a value and another thread has a dependency on one operation on that value, then the latter depends on the former. Specifically, suppose thread A performs three writes to `x`, and thread B depends only on A's third write to `x`, not on the first two writes. When A performs `x.release()`, meaning it uses `std::memory_order_release`, the `std::memory_order_consume` option ensures that B observes A's third write to `x` when B calls `x.load()`. Here is an example:

```cpp
// Initialize to nullptr to prevent the consumer thread from reading from a wild pointer.
std::atomic<int*> ptr(nullptr);
int v;
std::thread producer([&]() {
    int* p = new int(42);
    v = 1024;
    ptr.store(p, std::memory_order_release);
});
std::thread consumer([&]() {
    int* p;
    while(!(p = ptr.load(std::memory_order_consume)));

    std::cout << "p: " << *p << std::endl;
    std::cout << "v: " << v << std::endl;
});
producer.join();
consumer.join();
```

<mark style="background-color:green;">**Release/acquire model**</mark>: this model further tightens ordering constraints for atomic operations across different threads. It defines an ordering between release, `std::memory_order_release`, and acquire, `std::memory_order_acquire`: **all** writes before the release operation are visible to any acquire operation in another thread, establishing a happens-before relationship.

`std::memory_order_release` ensures that writes before it do not occur after the release operation; it is a backward barrier. `std::memory_order_acquire` ensures that reads and writes after it do not occur before the acquire operation; it is a forward barrier. `std::memory_order_acq_rel` combines both properties and defines a memory barrier that prevents the current thread's memory reads and writes from being reordered across this operation in either direction.

Consider this example:

```cpp
std::vector<int> v;
std::atomic<int> flag = {0};
std::thread release([&]() {
    v.push_back(42);
    flag.store(1, std::memory_order_release);
});
std::thread acqrel([&]() {
    int expected = 1; // must be before compare_exchange_strong
    while(!flag.compare_exchange_strong(expected, 2, std::memory_order_acq_rel))
        expected = 1; // must be after compare_exchange_strong
    // flag has changed to 2
});
std::thread acquire([&]() {
    while(flag.load(std::memory_order_acquire) < 2);

    std::cout << v.at(0) << std::endl; // must be 42
});
release.join();
acqrel.join();
acquire.join();
```

This example uses the compare-and-swap primitive `compare_exchange_strong`. It has a weaker version, `compare_exchange_weak`, which may return `false` even when the exchange could succeed. This is due to spurious failure on some platforms, specifically inconsistencies that can arise when the CPU performs a context switch and another thread loads the same address. In addition, `compare_exchange_strong` may have slightly worse performance than `compare_exchange_weak`. In most cases, given the complexity of its use, `compare_exchange_weak` should be considered first when the code is prepared to retry in a loop.

<mark style="background-color:green;">**Sequentially consistent model**</mark>: in this model, atomic operations satisfy sequential consistency, which may reduce performance. It can be specified explicitly with `std::memory_order_seq_cst`. Finally, consider this example:

```cpp
std::atomic<int> counter = {0};
std::vector<std::thread> vt;
for (int i = 0; i < 100; ++i) {
    vt.emplace_back([&](){
        counter.fetch_add(1, std::memory_order_seq_cst);
    });
}

for (auto& t : vt) {
    t.join();
}
std::cout << "current counter:" << counter << std::endl;
```

This example is essentially the same as the first **relaxed model** example. The only difference is that the memory order of the atomic operation is changed to `memory_order_seq_cst`. Interested readers can write a program to measure the performance difference caused by these two memory orders.
