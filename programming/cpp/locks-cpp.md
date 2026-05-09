---
description: C++ API
---

# Lock V1

## **Mutex**

Mutexes prevent multiple threads from operating on the same shared resource at the same time.

For example, a thread pool may contain multiple idle worker threads and one task queue. Any worker that accesses the task queue should use a mutex so that multiple threads do not modify or read the queue concurrently and corrupt its state.

At any moment, only one thread can acquire a mutex. Before that thread releases the mutex, no other thread can acquire it. If another thread tries to acquire the same mutex, it waits in a blocking manner.

Constructing an instance of `std::mutex` creates a mutex object. Calling `lock()` locks it, and calling `unlock()` unlocks it.

However, directly calling `lock()` and `unlock()` is generally not recommended. The standard C++ library provides the `std::lock_guard` class template, which implements the `RAII` idiom for mutex ownership.

```cpp
#include <mutex>
#include <list> // the mutex protects the list

std::list<int> this_list;
std::mutex this_mutex;

void add_to_list (int value) {
    std::lock_guard<std::mutex> guard(this_mutex);
    this_list.push_back(value);
}
```

Be careful about deadlocks: multiple threads may compete for shared resources in a way that prevents every thread from obtaining all the resources it needs, so the program cannot make progress.

* Mutual exclusion: a resource can be used by only one thread at a time.
* Hold and wait: a thread requests more resources while still holding resources it already owns.
* No preemption: resources already acquired by a thread cannot be forcibly taken away before the thread releases them.
* Circular wait: threads form a cycle in which each waits for a resource held by the next thread.

#### **Directly Operating on a Mutex with `lock` / `unlock`**

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
    v.reserve(SIZE);
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

Use `lock_guard` to lock and unlock automatically. The principle is RAII, similar to how smart pointers manage resources.

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

std::mutex mutex_2;
int count_2 = 0;

void counter() {
    // lock_guard locks in its constructor and unlocks in its destructor.
    std::lock_guard<std::mutex> lock(mutex_2);
    int i = ++count_2;
    ……
}

int main() {
    const std::size_t SIZE = 4;
    std::vector<std::thread> v;
    v.reserve(SIZE);
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

Use `unique_lock` to lock and unlock automatically. `unique_lock` follows the same RAII principle as `lock_guard`, but it provides more functionality, such as delayed locking, manual unlock/relock, move ownership, and condition-variable support. In older Boost APIs, `mutex::scoped_lock` is essentially a typedef-style wrapper around a unique-lock type; in standard C++, use `std::unique_lock`.

`counter` function body:

```cpp
void counter() {
    std::unique_lock<std::mutex> lock(mutex_3);
    int i = ++count_3;
    ……
}
```

#### **std::recursive\_mutex**

Like a regular `mutex`, `recursive_mutex` is a lockable object, but it **allows the same thread to acquire multiple ownership levels of the mutex object**, meaning the same thread can lock it multiple times.

This allows a thread that already owns the mutex to lock, or attempt to lock, the same mutex again, thereby acquiring another ownership level:

**The mutex remains owned by that thread until `unlock()` has been called the same number of times as the successful lock acquisitions**.

1. The calling thread starts owning a `recursive_mutex` after a successful call to `lock()` or `try_lock()`. During ownership, the same thread may make additional `lock()` or `try_lock()` calls. Ownership ends only after matching `unlock()` calls.
2. While one thread owns a `recursive_mutex`, another thread calling `lock()` blocks, and another thread calling `try_lock()` returns `false`.
3. The maximum number of times a `recursive_mutex` can be locked is unspecified. After that limit is reached, `lock()` throws `std::system_error`, and `try_lock()` returns `false`.
4. Destroying a `recursive_mutex` while it is still owned by a thread causes undefined behavior. `recursive_mutex` satisfies the requirements of a mutex type and a standard-layout type.

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

`std::timed_mutex` is a timed lockable object. Like a regular mutex, it provides exclusive access to critical code, but it also supports timed attempts to acquire the lock.

| lock             | The calling thread locks the `timed_mutex` and blocks if necessary, behaving like `mutex` |
| ---------------- | ---------------------------------------------- |
| try\_lock        | Attempts to lock the `timed_mutex` without blocking; returns immediately on failure |
| try\_lock\_for   | Attempts to lock the `timed_mutex`, blocking for at most `rel_time` |
| try\_lock\_until | Attempts to lock the `timed_mutex`, blocking until at most the `abs_time` time point |
| unlock           | Unlocks the `timed_mutex` and releases ownership, behaving like `mutex` |

#### **std::recursive\_timed\_mutex**

`std::recursive_timed_mutex` combines the features of `recursive_mutex` and `timed_mutex` in one class:

* It supports multiple ownership levels by a single thread.
* It also supports timed `try_lock` requests.

Its member functions are the same style as `timed_mutex`.

#### **Using `once_flag` and `call_once`**

In multithreaded code, one common scenario is that **a task should be executed exactly once**. This can be implemented with C++11's `std::call_once` and `std::once_flag`.

When **multiple threads call a function concurrently**, **`std::call_once` guarantees that the target function is called only once**.

This can be used to implement a **thread-safe singleton**.

```cpp
// header file
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
// cpp file
Task* Task::task;
Task::Task() {
	std::cout << "constructor" << std::endl;
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

## **Condition Variable**

The so-called "condition lock" is usually a condition variable. It is not used to manage a mutex directly; its purpose is thread synchronization. Its usage is similar to a common flag in programming: two actors agree that `flag = true` is the signal to proceed. The default value is `false`; A keeps checking the flag, and once B changes it to `true`, A starts acting.

When a thread cannot proceed because a condition is not satisfied, it can use a condition variable to block.

Once the condition is satisfied, one of the threads blocked on that condition can be woken up in a signal-like manner.

The most common example is a thread pool. Initially, when there is no task, the task queue is empty, so worker threads block because the "task queue is empty" condition is true. Once a task arrives, one thread is notified to process it.

Types:

* `std::condition_variable`: works only with `std::mutex`.
* `std::condition_variable_any`: works with any lock type that satisfies the minimal mutex-like requirements.

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
        std::unique_lock<std::mutex> lk(mut); // use unique_lock so it can be unlocked later
        data_con.wait(lk, [] {
            return !data_queue.empty();
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
		condi.notify_one();			// wake one waiting thread; condi.notify_all() wakes all waiting threads
		count--;
		std::this_thread::sleep_for(std::chrono::seconds(1));
	}
}

void function_2() {
	int data = 100;
	while (data > 1) {
		std::unique_lock<std::mutex> locker(mu);
		condi.wait(locker,			// unlock locker and sleep; relock after notification
			[]() { return !q.empty(); });   // the thread proceeds only when q is not empty
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

`cond.notify_one()`: wakes one waiting thread.

`cond.notify_all()`: wakes all waiting threads.

`wait()` behavior: check the condition and return when it is satisfied.

Two overloads:

```cpp
void wait( std::unique_lock<std::mutex>& lock );                  //  (1)	(since C++11)

template< class Predicate >
void wait( std::unique_lock<std::mutex>& lock, Predicate pred );  //  (2)	(since C++11)
```

If the condition is not satisfied, `wait()` unlocks the mutex and puts the thread into a blocked or waiting state.

When a data-preparation thread calls `notify_one()` on the condition variable, the waiting thread wakes from sleep, reacquires the mutex lock, and checks the condition again. If the condition is satisfied, `wait()` returns while the mutex is still locked. If the condition is not satisfied, the thread unlocks the mutex and waits again.

*   `void wait( std::unique_lock<std::mutex>& lock )`

    First unlocks the previously acquired mutex, then blocks the current execution thread.

    It adds the current thread to the waiting-thread list. The thread remains blocked until it is woken by `notify_all()` or `notify_one()`.

    After waking, the thread reacquires the mutex. Once the mutex is acquired, it continues with the following actions.

    A blocked thread can also wake spuriously.
*   `template< class Predicate > void wait( std::unique_lock<std::mutex>& lock, Predicate pred );`

    This overload adds the second parameter `Predicate`. The current thread blocks only while `pred` evaluates to `false`.

    In this case, after the thread wakes, it checks `pred` again first.

    If `pred` is still `false`, it releases the mutex and blocks in `wait` again.

    Therefore the predicate must be checked while protected by the same mutex. This overload removes the effect of spurious wakeups from user code.

If a waiting thread only needs to wait once, then after the condition becomes `true` it will no longer wait on this condition variable.

A condition variable is not always the best synchronization mechanism. This is especially true when the condition being waited for is the availability of one specific data block. In that scenario, a future is often more appropriate. Use `future` for one-shot events.

## **Spinlock**

Suppose we have a computer with two processor cores, `core1` and `core2`. A program running on this machine has two threads, `T1` and `T2`, running on `core1` and `core2` respectively. The two threads share one resource.

First consider how a mutex works. A mutex is a sleep-waiting lock. Suppose thread `T1` acquires the mutex and is running on `core1`. At the same time, thread `T2` also wants to acquire the mutex through `pthread_mutex_lock`, but because `T1` is using the mutex, `T2` is blocked. When `T2` is blocked, it is placed into a wait queue, and processor `core2` can handle other work instead of busy-waiting. In other words, the processor does not stay idle merely because the thread is blocked; it can run other tasks.

A spinlock is different. A spinlock is a busy-waiting lock. If `T1` is using a spinlock and `T2` tries to acquire it, `T2` cannot acquire it immediately.

Unlike with a mutex, the processor running `T2`, `core2`, repeatedly checks whether the lock is available until the spinlock is acquired.

The name "spinlock" reflects this behavior. If a thread wants to acquire a spinlock that is already held, it continuously consumes CPU while requesting the lock, preventing that CPU from doing other work until the lock is acquired. This is the meaning of "spinning."

When blocking occurs, a mutex can let the CPU run other work, while a spinlock keeps the CPU looping and trying to acquire the lock. From this comparison, it is clear that spinlocks can be expensive in CPU time.

```cpp
// use std::atomic_flag
class spinlock_mutex {
    std::atomic_flag flag;
    public:
    spinlock_mutex(): flag(ATOMIC_FLAG_INIT) {
    }
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire));
    }
    void unlock() {
        flag.clear(std::memory_order_release);
    }
}
```

## **Read-Write Lock**

We may allow multiple read operations on a database at the same time, while allowing only one write operation at any moment to update the data. This is a simple reader-writer model.

Header: `boost/thread/shared_mutex.hpp`. Type: `boost::shared_lock`.

It provides two access modes: shared and exclusive.

Use `lock` / `try_lock` to acquire exclusive access, and `lock_shared` / `try_lock_shared` to acquire shared access.

This setup is useful for distinguishing read and write operations across threads. `std::shared_mutex` was introduced in `C++17`, so pay attention to compiler and standard-version support.

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
