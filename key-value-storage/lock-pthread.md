# 😇 Lock(pthread)

## **互斥锁、条件锁、读写锁以及自旋锁**

锁是一个常见的同步概念，我们都听说过加锁（lock）或者解锁（unlock），当然学术一点地说法是获取（acquire）和释放（release）。

![img](https://s2.loli.net/2022/07/24/3gexCDnSULmpQqw.png)

恰好`pthread`包含这几种锁的API，而`C++11`只包含其中的部分。接下来我将通过`pthread`的API来展开回答。

### _**mutex（互斥量）**_

`mutex`（mutual exclusive）即互斥量（互斥体）。也便是常说的互斥锁。 尽管名称不含lock，但是称之为锁，也是没有太大问题的。

`mutex`无疑是最常见的多线程同步方式。其思想简单粗暴，多线程共享一个互斥量，然后线程之间去竞争。

得到锁的线程可以进入临界区执行代码。

```c
// 声明一个互斥量    
pthread_mutex_t mtx;
// 初始化 
pthread_mutex_init(&mtx, NULL);
// 加锁  
pthread_mutex_lock(&mtx);
// 解锁 
pthread_mutex_unlock(&mtx);
// 销毁
pthread_mutex_destroy(&mtx); 
```

`mutex`是睡眠等待（**sleep waiting**）类型的锁

当线程抢互斥锁失败的时候，线程会陷入休眠。

优点就是节省CPU资源，缺点就是休眠唤醒会消耗一点时间。另外自从Linux 2.6版以后，mutex完全用[**`futex`**](https://www.zhihu.com/search?q=futex\&search\_source=Entity\&hybrid\_search\_source=Entity\&hybrid\_search\_extra=\{%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1267625567})的API实现了，内部系统调用的开销大大减小。

值得一提的是，pthread的锁一般都有一个`trylock`的函数，比如对于互斥量：

```c
ret = pthread_mutex_trylock(&mtx);
if (0 == ret) { // 加锁成功
    ... 
    pthread_mutex_unlock(&mtx);
} else if (EBUSY == ret) { // 锁正在被使用;
    ... 
}
```

`pthread_mutex_trylock`用于以非阻塞的模式来请求互斥量。就好比各种IO函数都有一个noblock的模式一样，对于加锁这件事也有类似的非阻塞模式。

当线程尝试加锁时，如果锁已经被其他线程锁定，该线程就会阻塞住，直到能成功acquire。但有时候我们不希望这样。pthread\_mutex\_trylock在被其他线程锁定时，会返回特殊错误码。加锁成返回0，仅当成功但时候，我们才能解锁在后面进行解锁操作！

C++11开始引入了多线程库，其中也包含了互斥锁的API：**std::mutex** 。

此外，依据同一线程是否能多次加锁，把互斥量又分为如下两类：

* 是：称为『递归互斥量』recursive mutex ，也称『**可重入锁**』reentrant lock
* 否：即『非递归互斥量』non-recursive mute），也称『**不可重入锁**』non-reentrant mutex

若同一线程对[非递归](https://www.zhihu.com/search?q=%E9%9D%9E%E9%80%92%E5%BD%92\&search\_source=Entity\&hybrid\_search\_source=Entity\&hybrid\_search\_extra=\{%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1267625567})的互斥量多次加锁，可能会造成死锁。递归互斥量则无此风险。C++11中有递归互斥量的API：**std::recursive\_mutex**。对于pthread则可以通过给mutex添加PTHREAD\_MUTEX\_RECURSIVE 属性的方式来使用递归互斥量：

```c
// 声明一个互斥量
pthread_mutex_t mtx;
// 声明一个互斥量的属性变量
pthread_mutexattr_t mtx_attr;

// 初始化互斥量的属性变量
pthread_mutexattr_init(&mtx_attr);
// 设置递归互斥量的属性
pthread_mutexattr_settype(&mtx_attr, PTHREAD_MUTEX_RECURSIVE);

// 把属性赋值给互斥量
pthread_mutext_init(&mtx, &mutext_attr);
```

然而对于**递归互斥量**或者说**可重入锁**的使用则需要克制。Stevens大神生前在《APUE》中说『使用好它是十分tricky的，仅当没有其他解决方案时才使用』\[[1\]](lock-pthread.md#ref\_1)。

可重入锁这个概念和称呼的走俏多半是Java语言的功劳。

### condition variable（条件变量）

题主所谓的条件锁，我猜指的应该是条件变量。请注意条件变量不是锁，它是一种线程间的通讯机制，并且几乎总是和互斥量一起使用的。所以互斥量和条件变量二者一般是成套出现的。比如C++11中也有条件变量的API： **std::condition\_variable**。

对于pthread：

```c
// 声明一个互斥量     
pthread_mutex_t mtx;
// 声明一个条件变量
pthread_cond_t cond;
...

// 初始化 
pthread_mutex_init(&mtx, NULL);
pthread_cond_init(&cond, NULL);

// 加锁  
pthread_mutex_lock(&mtx);
// 加锁成功，等待条件变量触发
pthread_cond_wait(&cond, &mtx);

...
// 加锁  
pthread_mutex_lock(&mtx);
pthread_cond_signal(&cond);
...

// 解锁 
pthread_mutex_unlock(&mtx);
// 销毁
pthread_mutex_destroy(&mtx);
```

pthread\_cond\_wait函数会把条件变量和互斥量都传入。并且多线程调用的时候条件变量和互斥量一定要一一对应，不能一个条件变量在不同线程中wait的时候传入不同的互斥量。否则是未定义结果。

关于是先解锁互斥量还是先进行条件变量的通知，是另外一个比较大的议题。有种论断说：先解锁互斥量再通知条件变量可以减少多余的上下文切换，进而提高效率。这种说法是基于一种实现假设：先通知条件变量，再解锁。可能让其他等待条件变量的线程被唤醒了，但是此时互斥量还没解锁，从而再次陷入休眠。然而对于另外一些实现，比如Linux系统，则通过等待变形（**wait morphing**）\[[2\]](lock-pthread.md#ref\_2)解决了这一问题。所以先通知再解锁也没用问题。

**另外在使用条件变量的过程中有个稍微违反直觉的写法**：那就是使用while而不是if来做判断状态是否满足。这样做的原因有二：

1. 避免惊群；
2. 避免某些情况下线程被**虚假唤醒**（即没有pthread\_cond\_signal就解除了阻塞）。

比如**半同步/半reactor**的网络模型中，在工作线程消费fd队列的时候：

```c
while (1) {
    if (pthread_mutex_lock(&mtx) != 0) { // 加锁
        ... // 异常逻辑
    }
    while (queue.empty()) {
        if (pthread_cond_wait(&cond, &mtx) != 0) {
            ... // 异常逻辑
        }
    }
    auto data = queue.pop();
    if (pthread_mutex_unlock(&mtx) != 0) { // 解锁
        ... // 异常逻辑
    }
    process(data); // 处理流程，业务逻辑
}
```

### read-write lock（读写锁）

顾名思义『读写锁』就是对于临界区区分读和写。在读多写少的场景下，不加区分的使用互斥量显然是有点浪费的。此时便该上演读写锁的拿手好戏。

读写锁有一个别称叫『共享-独占锁』。不过单看『共享-独占锁』或者『读写锁』这两个名称，其实并未区分对于读和写，到底谁共享，谁独占。可能会让人误以为读写锁是一种更为泛化的称呼，其实不是。读写锁的含义是准确的：是一种 读共享，写独占的锁。

读写锁的特性：

* 当读写锁被加了写锁时，其他线程对该锁加读锁或者写锁都会**阻塞**（不是失败）。
* 当读写锁被加了读锁时，其他线程对该锁加写锁会**阻塞**，加读锁会成功。

因而适用于多读少写的场景。

```c
// 声明一个读写锁
pthread_rwlock_t rwlock;
...
// 在读之前加读锁
pthread_rwlock_rdlock(&rwlock);

... 共享资源的读操作

// 读完释放锁
pthread_rwlock_unlock(&rwlock);

// 在写之前加写锁
pthread_rwlock_wrlock(&rwlock); 

... 共享资源的写操作

// 写完释放锁
pthread_rwlock_unlock(&rwlock);

// 销毁读写锁
pthread_rwlock_destroy(&rwlock);
```

其实加读锁和加写锁这两个说法可能会造成误导，让人误以为是有两把锁，其实读写锁是一个锁。所谓加读锁和加写锁，准确的说法可能是『给读写锁加读模式的锁定和加写模式的锁定』。

读写锁和互斥量一样也有trylock函数，也是以非阻塞地形式来请求锁，不会导致阻塞。

```c
 pthread_rwlock_tryrdlock(&rwlock)
 pthread_rwlock_trywrlock(&rwlock)
```

C++11中有互斥量、条件变量但是并没有引入读写锁。而在C++17中出现了一种新锁：**std::shared\_mutex**。用它可以模拟实现出读写锁。demo代码可以直接参考cppreference👇：

[**std::shared\_mutex - cppreference.comen.cppreference.com/w/cpp/thread/shared\_mute**x](https://link.zhihu.com/?target=https%3A//en.cppreference.com/w/cpp/thread/shared\_mutex)

另外多读少写的场景有些特殊场景，可以用特殊的数据结构减少锁使用：

* 多读单写的线性数据。**用数组实现**[**环形队列**](https://www.zhihu.com/search?q=%E7%8E%AF%E5%BD%A2%E9%98%9F%E5%88%97\&search\_source=Entity\&hybrid\_search\_source=Entity\&hybrid\_search\_extra=\{%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1267625567})，避免vector等动态扩张的数据结构，写在结尾，由于单写因而可以不加锁；读在开头，由于多读（避免重复消费）所以需要加一下锁（互斥量就行）。
* 多读单写的KV。**可以使用**[**双缓冲**](https://www.zhihu.com/search?q=%E5%8F%8C%E7%BC%93%E5%86%B2\&search\_source=Entity\&hybrid\_search\_source=Entity\&hybrid\_search\_extra=\{%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1267625567})**（double buffer）的数据结构**来实现。double buffer同名的概念比较多，这里指的是foreground 和 [backgroud](https://www.zhihu.com/search?q=backgroud\&search\_source=Entity\&hybrid\_search\_source=Entity\&hybrid\_search\_extra=\{%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1267625567}) 两个buffer进行切换的『0 - 1切换』技术。比如实现动态加载（热加载）配置文件的时候。可能会在切换间隙加一个短暂的互斥量，但是基本可以认为是lock free的。

我一张口，你就会发现：无非也就是空间换时间的老套路了。

### spinlock（自旋锁）

自旋之名颇为玄妙，第一次听闻常让人略觉高大。但和无数个好似『故意把简单概念复杂化』的计算机术语一样，自旋锁的本质简单的难以置信。

要了解自旋锁，首先了解自旋。**什么是自旋（spin）呢？更为通俗的一个词是『忙等待』（busy waiting）。最最通俗的一个理解，其实就是死循环……**。

单看使用方法和使用互斥量的代码是差不多的。只不过自旋锁不会引起线程休眠。当共享资源的状态不满足的时候，自旋锁会不停地循环检测状态。因为不会陷入休眠，而是忙等待的方式也就不需要条件变量。

这是优点也是缺点。不休眠就不会引起上下文切换，但是会比较浪费CPU。

```c
// 声明一个自旋锁变量
pthread_spinlock_t spinlock;

// 初始化   
pthread_spin_init(&spinlock, 0);

// 加锁  
pthread_spin_lock(&spinlock);

// 解锁 
pthread_spin_unlock(&spinlock);

// 销毁  
pthread_spin_destroy(&spinlock);
```

pthread\_spin\_init函数的第二个参数名为pshared（int类型）。表示的是是否能进程间共享自旋锁。这被称之为Thread Process-Shared Synchronization。互斥量的通过属性也可以把互斥量设置成进程间共享的。pshared有两个枚举值：

* **PTHREAD\_PROCESS\_PRIVATE：仅同进程下读线程可以使用该自旋锁**
* **PTHREAD\_PROCESS\_SHARED：不同进程下的线程可以使用该自旋锁**

\*\*在Linux上的glibc中这两个枚举值分别是0和1（Mac上不是）。所以通常也会看到直接传0的代码。你可能觉得不使用宏，直接用数字硬编码不是一个好习惯。的确，妥妥的Magic Number，但还有一个有趣的事实你需要了解：\*\*并不是所有实现都支持自旋锁设置两种pshared。比如\[[3\]](lock-pthread.md#ref\_3)：

```c
int pthread_spin_init (pthread_spinlock_t *lock, int pshared) {
    /* Relaxed MO is fine because this is an initializing store.  */
    atomic_store_relaxed (lock, 0);
    return0;
}
```

所以直接传0可能也无伤大雅。

自旋锁 VS 互斥量+条件变量 孰优孰劣？肯定要看具体的使用场景。

当你不知道在你的使用场景下这两种锁该用哪个的时候，那就是用互斥量吧！或者通过压测的判断，不过大多数时候我们好像并不需要这么一个pthread的自旋锁，知友们可以提供一些自旋锁的使用参考。

内容太多，难免有误，望大家指教。

### **参考**

1. [^](lock-pthread.md#ref\_1\_0)1 **https://en.wikipedia.org/wiki/Reentrant\_mutex#Practical\_use**
2. [^](lock-pthread.md#ref\_2\_0)2 \[**https://books.google.com.hk/books?id=Ps2SH727eCIC\&pg=PA647\&lpg=PA647\&dq=linux+programming+interface+wait+morphing\&source=bl\&ots=kMKcz2zPC7\&sig=ACfU3U1ZSbxBegrQhuVkfNAMTRkY-YavvA\&hl=en\&sa=X\&redir\_esc=y\&hl=zh-CN\&sourceid=cndr#v=onepage\&q=linux%20programming%20interface%20wait%20morphing\&f=false**]\(https://books.google.com.hk/books?id=Ps2SH727eCIC\&pg=PA647\&lpg=PA647\&dq=linux+programming+interface+wait+morphing\&source=bl\&ots=kMKcz2zPC7\&sig=ACfU3U1ZSbxBegrQhuVkfNAMTRkY-YavvA\&hl=en\&sa=X\&redir\_esc=y\&hl=zh-CN\&sourceid=cndr#v=onepage\&q=linux programming interface wait morphing\&f=false)
3. [^](lock-pthread.md#ref\_3\_0)3 **https://github.com/lattera/glibc/blob/895ef79e04a953cac1493863bcae29ad85657ee1/nptl/pthread\_spin\_init.c**

## **C++11**
