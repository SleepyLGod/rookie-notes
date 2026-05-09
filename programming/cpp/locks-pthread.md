---
description: pthread
---

# Lock V2

## **Mutexes, Condition Variables, Read-Write Locks, and Spinlocks**

A lock is a common synchronization concept. We often talk about locking and unlocking; the more formal terms are acquire and release.

![img](https://s2.loli.net/2022/07/24/3gexCDnSULmpQqw.png)

`pthread` provides APIs for these synchronization primitives, while `C++11` includes only some of them. The following notes explain the concepts through the `pthread` APIs.

### _**Mutex**_

`mutex` stands for mutual exclusion. It is the usual mutex lock. Although the name does not contain the word "lock," it is still reasonable to call it a lock.

`mutex` is probably the most common synchronization mechanism in multithreaded programs. The idea is simple: multiple threads share one mutex, and the threads compete to acquire it.

The thread that acquires the lock can enter the critical section and execute the protected code.

```c
// declare a mutex
pthread_mutex_t mtx;
// initialize
pthread_mutex_init(&mtx, NULL);
// lock
pthread_mutex_lock(&mtx);
// unlock
pthread_mutex_unlock(&mtx);
// destroy
pthread_mutex_destroy(&mtx); 
```

`mutex` is a sleep-waiting lock.

When a thread fails to acquire a mutex, the thread may sleep.

The advantage is that it saves CPU resources. The disadvantage is that sleeping and waking have some overhead. Since Linux 2.6, mutex implementations have relied heavily on [**`futex`**](https://www.zhihu.com/search?q=futex\&search\_source=Entity\&hybrid\_search\_source=Entity\&hybrid\_search\_extra=\{%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1267625567}) APIs, which greatly reduce the cost of system calls in common cases.

It is worth noting that pthread locks usually provide a `trylock` function. For a mutex:

```c
ret = pthread_mutex_trylock(&mtx);
if (0 == ret) { // lock acquired
    ... 
    pthread_mutex_unlock(&mtx);
} else if (EBUSY == ret) { // the lock is already in use
    ... 
}
```

`pthread_mutex_trylock` requests a mutex in non-blocking mode. Just as many I/O functions provide a non-blocking mode, locking also has a similar non-blocking form.

When a thread calls the regular lock operation and the mutex is already held by another thread, it blocks until it can acquire the mutex. Sometimes we do not want that behavior. `pthread_mutex_trylock` returns a special error code when the mutex is already locked by another thread. It returns `0` only when the lock is successfully acquired, and only in that success case should the caller later unlock it.

C++11 introduced a standard threading library, including the mutex API **`std::mutex`**.

Depending on whether the same thread may acquire the same mutex multiple times, mutexes can be divided into two categories:

* Yes: a recursive mutex, also called a **reentrant lock**.
* No: a non-recursive mutex, also called a **non-reentrant mutex**.

If the same thread locks a [non-recursive](https://www.zhihu.com/search?q=%E9%9D%9E%E9%80%92%E5%BD%92\&search\_source=Entity\&hybrid\_search\_source=Entity\&hybrid\_search\_extra=\{%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1267625567}) mutex multiple times, it may deadlock. A recursive mutex avoids that particular risk. C++11 provides **`std::recursive_mutex`**. With pthreads, a recursive mutex can be created by setting the `PTHREAD_MUTEX_RECURSIVE` attribute:

```c
// declare a mutex
pthread_mutex_t mtx;
// declare a mutex attribute object
pthread_mutexattr_t mtx_attr;

// initialize the mutex attribute object
pthread_mutexattr_init(&mtx_attr);
// set the recursive-mutex attribute
pthread_mutexattr_settype(&mtx_attr, PTHREAD_MUTEX_RECURSIVE);

// apply the attributes to the mutex
pthread_mutex_init(&mtx, &mtx_attr);
```

However, **recursive mutexes**, or **reentrant locks**, should be used with restraint. Stevens wrote in *APUE* that using them correctly is tricky and that they should be used only when there is no other solution \[[1\]](locks-pthread.md#ref\_1).

The popularity of the term "reentrant lock" is probably due largely to Java.

### Condition Variable

What is sometimes called a "condition lock" usually refers to a condition variable. A condition variable is not a lock; it is a communication mechanism between threads and is almost always used together with a mutex. Therefore mutexes and condition variables usually appear as a pair. C++11 also provides a condition-variable API: **`std::condition_variable`**.

For pthreads:

```c
// declare a mutex
pthread_mutex_t mtx;
// declare a condition variable
pthread_cond_t cond;
...

// initialize
pthread_mutex_init(&mtx, NULL);
pthread_cond_init(&cond, NULL);

// lock
pthread_mutex_lock(&mtx);
// after locking successfully, wait for the condition variable
pthread_cond_wait(&cond, &mtx);

...
// lock
pthread_mutex_lock(&mtx);
pthread_cond_signal(&cond);
...

// unlock
pthread_mutex_unlock(&mtx);
// destroy
pthread_mutex_destroy(&mtx);
```

`pthread_cond_wait` receives both the condition variable and the mutex. In multithreaded use, the condition variable and mutex must correspond consistently. Do not wait on the same condition variable with different mutexes in different threads; doing so has undefined behavior.

Whether to unlock the mutex before notifying the condition variable is a larger topic. One argument says that unlocking first and then notifying can reduce unnecessary context switches and improve efficiency. That argument assumes an implementation where notifying first may wake another waiting thread while the mutex is still locked, causing the awakened thread to sleep again. Some implementations, such as Linux, address this through **wait morphing** \[[2\]](locks-pthread.md#ref\_2). Therefore notify-before-unlock is not inherently wrong.

**One slightly counterintuitive pattern when using condition variables** is to use `while`, not `if`, to test whether the condition is satisfied. There are two reasons:

1. Avoid thundering-herd effects.
2. Handle **spurious wakeups**, where a thread may stop blocking even without a corresponding `pthread_cond_signal`.

For example, in a **half-sync/half-reactor** network model, a worker thread consuming an fd queue might be written like this:

```c
while (1) {
    if (pthread_mutex_lock(&mtx) != 0) { // lock
        ... // error handling
    }
    while (queue.empty()) {
        if (pthread_cond_wait(&cond, &mtx) != 0) {
            ... // error handling
        }
    }
    auto data = queue.pop();
    if (pthread_mutex_unlock(&mtx) != 0) { // unlock
        ... // error handling
    }
    process(data); // processing flow, business logic
}
```

### Read-Write Lock

As the name suggests, a read-write lock distinguishes reads from writes in a critical section. In read-heavy and write-light workloads, using an ordinary mutex without distinguishing reads and writes can be wasteful. This is where read-write locks are useful.

A read-write lock is also called a shared-exclusive lock. The name "shared-exclusive lock" alone does not say whether reads or writes are shared or exclusive, which can make it sound more general than it is. The term "read-write lock" is precise: reads are shared, writes are exclusive.

Read-write lock behavior:

* When a read-write lock is held in write mode, other threads trying to acquire either a read lock or a write lock will **block** rather than simply fail.
* When a read-write lock is held in read mode, other threads trying to acquire a write lock will **block**, while other readers can acquire the read lock successfully.

Therefore, read-write locks are suitable for workloads with many reads and few writes.

```c
// declare a read-write lock
pthread_rwlock_t rwlock;
...
// acquire read mode before reading
pthread_rwlock_rdlock(&rwlock);

... read the shared resource

// release the lock after reading
pthread_rwlock_unlock(&rwlock);

// acquire write mode before writing
pthread_rwlock_wrlock(&rwlock); 

... write the shared resource

// release the lock after writing
pthread_rwlock_unlock(&rwlock);

// destroy the read-write lock
pthread_rwlock_destroy(&rwlock);
```

The phrases "read lock" and "write lock" can be misleading because they may sound like two separate locks. A read-write lock is one lock. More precisely, we acquire the read-write lock in read mode or in write mode.

Like mutexes, read-write locks also provide `trylock` functions that request the lock in non-blocking mode:

```c
 pthread_rwlock_tryrdlock(&rwlock)
 pthread_rwlock_trywrlock(&rwlock)
```

C++11 provides mutexes and condition variables, but it did not introduce a standard read-write lock. C++17 added **`std::shared_mutex`**, which can be used as a read-write lock. Example code is available on cppreference:

[**std::shared\_mutex - cppreference.comen.cppreference.com/w/cpp/thread/shared\_mute**x](https://link.zhihu.com/?target=https%3A//en.cppreference.com/w/cpp/thread/shared\_mutex)

In some read-heavy/write-light scenarios, special data structures can reduce lock usage:

* Multi-reader, single-writer linear data. Use an array-based [**ring buffer**](https://www.zhihu.com/search?q=%E7%8E%AF%E5%BD%A2%E9%98%9F%E5%88%97\&search\_source=Entity\&hybrid\_search\_source=Entity\&hybrid\_search\_extra=\{%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1267625567}) instead of dynamically expanding structures such as `vector`. The single writer appends at the tail and may avoid locking; readers consume from the head, and because there are multiple readers and duplicate consumption must be avoided, a simple mutex may still be needed.
* Multi-reader, single-writer KV data. A [**double-buffer**](https://www.zhihu.com/search?q=%E5%8F%8C%E7%BC%93%E5%86%B2\&search\_source=Entity\&hybrid\_search\_source=Entity\&hybrid\_search\_extra=\{%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1267625567}) data structure can be used. "Double buffer" is an overloaded term; here it means switching between a foreground buffer and a [background](https://www.zhihu.com/search?q=backgroud\&search\_source=Entity\&hybrid\_search\_source=Entity\&hybrid\_search\_extra=\{%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1267625567}) buffer. This is useful for dynamic or hot-loaded configuration files. A short mutex may still be needed during the switch, but the design can be treated as mostly lock-free in the steady state.

In short, this is the classic space-for-time tradeoff.

### Spinlock

The term "spin" can sound mysterious at first, but like many computer-science terms that appear to complicate a simple concept, the essence of a spinlock is very simple.

To understand a spinlock, first understand spinning. **What is spinning? A more ordinary term is busy waiting. In the most direct sense, it is essentially a loop that keeps checking.**

At the API level, the code looks similar to mutex usage. The difference is that a spinlock does not put the thread to sleep. When the shared resource is not available, the spinlock continuously loops and checks the state. Because it does not sleep and instead busy-waits, it does not need a condition variable.

This is both an advantage and a disadvantage. Avoiding sleep avoids context-switch overhead, but busy waiting wastes CPU time.

```c
// declare a spinlock variable
pthread_spinlock_t spinlock;

// initialize
pthread_spin_init(&spinlock, 0);

// lock
pthread_spin_lock(&spinlock);

// unlock
pthread_spin_unlock(&spinlock);

// destroy
pthread_spin_destroy(&spinlock);
```

The second argument of `pthread_spin_init` is named `pshared` and has type `int`. It indicates whether the spinlock can be shared between processes. This is called thread process-shared synchronization. Mutexes can also be configured as process-shared through attributes. `pshared` has two enum values:

* **`PTHREAD_PROCESS_PRIVATE`**: only threads in the same process can use the spinlock.
* **`PTHREAD_PROCESS_SHARED`**: threads in different processes can use the spinlock.

**In glibc on Linux, these two enum values are `0` and `1` respectively. This is not true on every platform, such as macOS. That is why code sometimes passes `0` directly. Hard-coding a number instead of using the macro is still not a good habit; it is a magic number. But there is also an interesting detail:** not every implementation supports both `pshared` modes for spinlocks. For example \[[3\]](locks-pthread.md#ref\_3):

```c
int pthread_spin_init (pthread_spinlock_t *lock, int pshared) {
    /* Relaxed MO is fine because this is an initializing store.  */
    atomic_store_relaxed (lock, 0);
    return 0;
}
```

So passing `0` directly may work in practice on such implementations, even though using the symbolic constant is clearer.

Which is better, a spinlock or a mutex plus condition variable? It depends on the specific workload.

If you do not know which one to use in your scenario, use a mutex. You can also decide through stress testing, but in most ordinary cases a pthread spinlock is not needed. More concrete spinlock use cases would require workload-specific evidence.

This topic has many details, so corrections are welcome if any point is inaccurate.

### **References**

1. [^](locks-pthread.md#ref\_1\_0)1 **https://en.wikipedia.org/wiki/Reentrant\_mutex#Practical\_use**
2. [^](locks-pthread.md#ref\_2\_0)2 \[**https://books.google.com.hk/books?id=Ps2SH727eCIC\&pg=PA647\&lpg=PA647\&dq=linux+programming+interface+wait+morphing\&source=bl\&ots=kMKcz2zPC7\&sig=ACfU3U1ZSbxBegrQhuVkfNAMTRkY-YavvA\&hl=en\&sa=X\&redir\_esc=y\&hl=zh-CN\&sourceid=cndr#v=onepage\&q=linux%20programming%20interface%20wait%20morphing\&f=false**]\(https://books.google.com.hk/books?id=Ps2SH727eCIC\&pg=PA647\&lpg=PA647\&dq=linux+programming+interface+wait+morphing\&source=bl\&ots=kMKcz2zPC7\&sig=ACfU3U1ZSbxBegrQhuVkfNAMTRkY-YavvA\&hl=en\&sa=X\&redir\_esc=y\&hl=zh-CN\&sourceid=cndr#v=onepage\&q=linux programming interface wait morphing\&f=false)
3. [^](locks-pthread.md#ref\_3\_0)3 [glibc `pthread_spin_init.c`](https://github.com/bminor/glibc/blob/master/nptl/pthread_spin_init.c)

## **C++11**
