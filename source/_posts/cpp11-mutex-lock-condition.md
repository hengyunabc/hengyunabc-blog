---
title: C++11中的mutex, lock，condition variable实现分析
date: 2014-06-22 19:58:28
tags:
 - cpp
 - cpp11


categories:
  - 技术

---

## 前言

本文分析的是llvm libc++的实现：http://libcxx.llvm.org/

C++11中的各种mutex, lock对象，实际上都是对posix的mutex，condition的封装。不过里面也有很多细节值得学习。

## std::mutex

先来看下std::mutex：

包增了一个pthread_mutex_t __m_，很简单，每个函数该干嘛就干嘛。

```cpp
class mutex
{
    pthread_mutex_t __m_;
 
public:
     mutex() _NOEXCEPT {__m_ = (pthread_mutex_t)<strong>PTHREAD_MUTEX_INITIALIZER</strong>;}
     ~mutex();
private:
    mutex(const mutex&);// = delete;
    mutex& operator=(const mutex&);// = delete;
public:
    void lock();
    bool try_lock() _NOEXCEPT;
    void unlock() _NOEXCEPT;
 
    typedef pthread_mutex_t* native_handle_type;
    _LIBCPP_INLINE_VISIBILITY native_handle_type native_handle() {return &__m_;}
};
 
mutex::~mutex()
{
    pthread_mutex_destroy(&__m_);
}
 
void mutex::lock()
{
    int ec = pthread_mutex_lock(&__m_);
    if (ec)
        __throw_system_error(ec, "mutex lock failed");
}
 
bool mutex::try_lock() _NOEXCEPT
{
    return pthread_mutex_trylock(&__m_) == 0;
}
 
void mutex::unlock() _NOEXCEPT
{
    int ec = pthread_mutex_unlock(&__m_);
    (void)ec;
    assert(ec == 0);
}
```

## 三种锁状态：std::defer_lock, std::try_to_lock, std::adopt_lock

这三个是用于标识锁在传递到一些包装类时，锁的状态：

* std::defer_lock，还没有获取到锁
* std::try_to_lock，在包装类构造时，尝试去获取锁
* std::adopt_lock，调用者已经获得了锁

这三个东东，实际上是用于偏特化的，是三个空的struct：

```cpp
struct  defer_lock_t {};
struct  try_to_lock_t {};
struct  adopt_lock_t {};
constexpr defer_lock_t  defer_lock  = defer_lock_t();
constexpr try_to_lock_t try_to_lock = try_to_lock_t();
constexpr adopt_lock_t  adopt_lock  = adopt_lock_t();
```

在下面的代码里，就可以看到这三个东东是怎么用的了。

## std::lock_guard

这个类比较重要，因为我们真正使用lock的时候，大部分都是要用这个。

这个类其实很简单：

* 在构造函数里调用 mutext.lock()，
* 在释构函数里，调用了mutex.unlock() 函数。

因为C++会在函数抛出异常时，自动调用作用域内的变量的析构函数，**所以使用std::lock_guard可以在异常时自动释放锁，这就是为什么要避免直接使用mutex的函数，而是要用std::lock_guard的原因了。**

```cpp
template <class _Mutex>
class lock_guard
{
public:
    typedef _Mutex mutex_type;
private:
    mutex_type& __m_;
public:
    explicit lock_guard(mutex_type& __m)
        : __m_(__m) {__m_.lock();}
    lock_guard(mutex_type& __m, adopt_lock_t)
        : __m_(__m) {}
    ~lock_guard() {__m_.unlock();}
private:
    lock_guard(lock_guard const&);// = delete;
    lock_guard& operator=(lock_guard const&);// = delete;
};
```

注意，std::lock_guard的两个构造函数，当只传递mutex时，会在构造函数时调用mutext.lock()来获得锁。

当传递了adopt_lock_t时，说明调用者已经拿到了锁，所以不再尝试去获得锁。

## std::unique_lock

unique_lock实际上也是一个包装类，起名为unique可能是和std::lock函数区分用的。
注意，多了一个owns_lock函数和release()函数，这两个在std::lock函数会用到。

owns_lock函数用于判断是否拥有锁；

release()函数则放弃了对锁的关联，当析构时，不会去unlock锁。
再看下unique_lock的实现，可以发现，**上面的三种类型就是用来做偏特化用的**：

```cpp
template <class _Mutex>
class unique_lock
{
public:
    typedef _Mutex mutex_type;
private:
    mutex_type* __m_;
    bool __owns_;
 
public:
    unique_lock() _NOEXCEPT : __m_(nullptr), __owns_(false) {}
    explicit unique_lock(mutex_type& __m)
        : __m_(&__m), __owns_(true) {__m_->lock();}
    unique_lock(mutex_type& __m, defer_lock_t) _NOEXCEPT
        : __m_(&__m), __owns_(false) {}
    unique_lock(mutex_type& __m, try_to_lock_t)    //偏特化
        : __m_(&__m), __owns_(__m.try_lock()) {}
    unique_lock(mutex_type& __m, adopt_lock_t)     //偏特化
        : __m_(&__m), __owns_(true) {}
    template <class _Clock, class _Duration>
        unique_lock(mutex_type& __m, const chrono::time_point<_Clock, _Duration>& __t)
            : __m_(&__m), __owns_(__m.try_lock_until(__t)) {}
    template <class _Rep, class _Period>
        unique_lock(mutex_type& __m, const chrono::duration<_Rep, _Period>& __d)
            : __m_(&__m), __owns_(__m.try_lock_for(__d)) {}
    ~unique_lock()
    {
        if (__owns_)
            __m_->unlock();
    }
 
private:
    unique_lock(unique_lock const&); // = delete;
    unique_lock& operator=(unique_lock const&); // = delete;
 
public:
    unique_lock(unique_lock&& __u) _NOEXCEPT
        : __m_(__u.__m_), __owns_(__u.__owns_)
        {__u.__m_ = nullptr; __u.__owns_ = false;}
    unique_lock& operator=(unique_lock&& __u) _NOEXCEPT
        {
            if (__owns_)
                __m_->unlock();
            __m_ = __u.__m_;
            __owns_ = __u.__owns_;
            __u.__m_ = nullptr;
            __u.__owns_ = false;
            return *this;
        }
 
    void lock();
    bool try_lock();
 
    template <class _Rep, class _Period>
    bool try_lock_for(const chrono::duration<_Rep, _Period>& __d);
    template <class _Clock, class _Duration>
    bool try_lock_until(const chrono::time_point<_Clock, _Duration>& __t);
 
    void unlock();
    void swap(unique_lock& __u) _NOEXCEPT
    {
        _VSTD::swap(__m_, __u.__m_);
        _VSTD::swap(__owns_, __u.__owns_);
    }
    mutex_type* release() _NOEXCEPT
    {
        mutex_type* __m = __m_;
        __m_ = nullptr;
        __owns_ = false;
        return __m;
    }
    bool owns_lock() const _NOEXCEPT {return __owns_;}
    operator bool () const _NOEXCEPT {return __owns_;}
    mutex_type* mutex() const _NOEXCEPT {return __m_;}
};
```

## std::lock和std::try_lock函数

上面的都是类对象，这两个是函数。
std::lock和std::try_lock函数用于在同时使用多个锁时，防止死锁。这个实际上很重要的，因为手写代码来处理多个锁的同步问题，很容易出错。

要注意的是std::try_lock函数的返回值：

* 当成功时，返回-1；
* 当失败时，返回第几个锁没有获取成功，以0开始计数；

首先来看下只有两个锁的情况，代码虽然看起来比较简单，但里面却有大文章：

```cpp
template <class _L0, class _L1>
void
lock(_L0& __l0, _L1& __l1)
{
    while (true)
    {
        {
            unique_lock<_L0> __u0(__l0);
            if (__l1.try_lock())  //已获得锁l0，再尝试获取l1
            {
                __u0.release();   //l0和l1都已获取到，因为unique_lock在释构时会释放l0，所以要调用release()函数，不让它释放l0锁。
                break;
            }
        }//如果同时获取l0,l1失败，这里会释放l0。
        sched_yield();  //把线程放到同一优先级的调度队列的尾部，CPU切换到其它线程执行
        {
            unique_lock<_L1> __u1(__l1); //因为上面尝试先获取l1失败，说明有别的线程在持有l1，那么这次先尝试获取锁l1（只有前面的线程释放了，才可能获取到）
            if (__l0.try_lock())
            {
                __u1.release();
                break;
            }
        }
        sched_yield();
    }
}
template <class _L0, class _L1>
int
try_lock(_L0& __l0, _L1& __l1)
{
    unique_lock<_L0> __u0(__l0, try_to_lock);
    if (__u0.owns_lock())
    {
        if (__l1.try_lock()) //注意try_lock返回值的定义，否则这里无法理解
        {
            __u0.release();
            return -1;
        }
        else
            return 1;
    }
    return 0;
}
```

上面的lock函数用尝试的办法防止了死锁。

上面是两个锁的情况，那么在多个参数的情况下呢？
先来看下std::try_lock函数的实现：

**里面递归地调用了try_lock函数自身，如果全部锁都获取成功，则依次把所有的unique_lock都release掉。**

如果有失败，则计数失败的次数，最终返回。

```cpp
template <class _L0, class _L1, class _L2, class... _L3>
int
try_lock(_L0& __l0, _L1& __l1, _L2& __l2, _L3&... __l3)
{
    int __r = 0;
    unique_lock<_L0> __u0(__l0, try_to_lock);
    if (__u0.owns_lock())
    {
        __r = try_lock(__l1, __l2, __l3...);
        if (__r == -1)
            __u0.release();
        else
            ++__r;
    }
    return __r;
}
```

再来看多参数的std::lock的实现：

```cpp
template <class _L0, class _L1, class _L2, class ..._L3>
void
__lock_first(int __i, _L0& __l0, _L1& __l1, _L2& __l2, _L3& ...__l3)
{
    while (true)
    {
        switch (__i)  //__i用来标记上一次获取参数里的第几个锁失败，从0开始计数
        {
        case 0:   //第一次执行时，__i是0
            {
                unique_lock<_L0> __u0(__l0);
                __i = try_lock(__l1, __l2, __l3...);
                if (__i == -1)  //获取到l0之后，如果尝试获取后面的锁也成功了，即全部锁都获取到了，则设置unique_lock为release，并返回
                {
                    __u0.release();
                    return;
                }
            }
            ++__i;  //因为__i表示是获取第几个锁失败，而上面的try_lock(__l1,__l2__l3,...)是从l1开始的，因此这里要+1，调整到没有获取成功的锁上，下次先从它开始获取。
            sched_yield();
            break;
        case 1:   //说明上次获取l1失败，这次先获取到l1。
            {
                unique_lock<_L1> __u1(__l1);    
                __i = try_lock(__l2, __l3..., __l0);   //把前一次的l0放到最后。这次先获取到了l1，再尝试获取后面的锁。
                if (__i == -1)
                {
                    __u1.release();
                    return;
                }
            }
            if (__i == sizeof...(_L3) + 1)   //说明把l0放到最后面时，最后获取l0时失败了。那么说明现在有其它线程持有l0，那么下一次要从l0开始获取。
                __i = 0;
            else
                __i += 2; //因为__i表示是获取第几个锁失败，而上面的try_lock(__l2,__l3..., __l0)是从l2开始的，因此这里要+2
            sched_yield();
            break;
        default:
            __lock_first(__i - 2, __l2, __l3..., __l0, __l1);    //因为这里是从l2开始的，因此__i要减2。
            return;
        }
    }
}
 
template <class _L0, class _L1, class _L2, class ..._L3>
inline _LIBCPP_INLINE_VISIBILITY
void
lock(_L0& __l0, _L1& __l1, _L2& __l2, _L3& ...__l3)
{
    __lock_first(0, __l0, __l1, __l2, __l3...);
}
```

可以看到多参数的std::lock的实现是：

* 先获取一个锁，然后再调用std::try_lock去获取剩下的锁，如果失败了，则下次先获取上次失败的锁。
* 重复上面的过程，直到成功获取到所有的锁。

上面的算法用比较巧妙的方式实现了参数的轮转。

## std::timed_mutex

std::timed_mutex   是里面封装了mutex和condition，这样就两个函数可以用：

* try_lock_for
* try_lock_until 

实际上是posix的mutex和condition的包装。

```cpp
class timed_mutex
{
    mutex              __m_;
    condition_variable __cv_;
    bool               __locked_;
public:
     timed_mutex();
     ~timed_mutex();
private:
    timed_mutex(const timed_mutex&); // = delete;
    timed_mutex& operator=(const timed_mutex&); // = delete;
public:
    void lock();
    bool try_lock() _NOEXCEPT;
    template <class _Rep, class _Period>
        _LIBCPP_INLINE_VISIBILITY
        bool try_lock_for(const chrono::duration<_Rep, _Period>& __d)
            {return try_lock_until(chrono::steady_clock::now() + __d);}
    template <class _Clock, class _Duration>
        bool try_lock_until(const chrono::time_point<_Clock, _Duration>& __t);
    void unlock() _NOEXCEPT;
};
 
template <class _Clock, class _Duration>
bool
timed_mutex::try_lock_until(const chrono::time_point<_Clock, _Duration>& __t)
{
    using namespace chrono;
    unique_lock<mutex> __lk(__m_);
    bool no_timeout = _Clock::now() < __t;
    while (no_timeout && __locked_)
        no_timeout = __cv_.wait_until(__lk, __t) == cv_status::no_timeout;
    if (!__locked_)
    {
        __locked_ = true;
        return true;
    }
    return false;
}
```

## std::recursive_mutex和std::recursive_timed_mutex

这两个实际上是std::mutex和std::timed_mutex 的recursive模式的实现，即锁得获得者可以重复多次调用lock()函数。

和posix mutex里的recursive mutex是一样的。

看下std::recursive_mutex的构造函数就知道了。

```cpp
recursive_mutex::recursive_mutex()
{
    pthread_mutexattr_t attr;
    int ec = pthread_mutexattr_init(&attr);
    if (ec)
        goto fail;
    ec = pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
    if (ec)
    {
        pthread_mutexattr_destroy(&attr);
        goto fail;
    }
    ec = pthread_mutex_init(&__m_, &attr);
    if (ec)
    {
        pthread_mutexattr_destroy(&attr);
        goto fail;
    }
    ec = pthread_mutexattr_destroy(&attr);
    if (ec)
    {
        pthread_mutex_destroy(&__m_);
        goto fail;
    }
    return;
fail:
    __throw_system_error(ec, "recursive_mutex constructor failed");
}

```

## std::cv_status

这个用来表示condition等待返回的状态的，和上面的三个表示lock的状态的用途差不多。

```cpp
enum cv_status
{
    no_timeout,
    timeout
};
```

## std::condition_variable

包装了posix condition variable。

```cpp
class condition_variable
{
    pthread_cond_t __cv_;
public:
    condition_variable() {__cv_ = (pthread_cond_t)PTHREAD_COND_INITIALIZER;}
    ~condition_variable();
private:
    condition_variable(const condition_variable&); // = delete;
    condition_variable& operator=(const condition_variable&); // = delete;
public:
    void notify_one() _NOEXCEPT;
    void notify_all() _NOEXCEPT;
 
    void wait(unique_lock<mutex>& __lk) _NOEXCEPT;
    template <class _Predicate>
        void wait(unique_lock<mutex>& __lk, _Predicate __pred);
 
    template <class _Clock, class _Duration>
        cv_status
        wait_until(unique_lock<mutex>& __lk,
                   const chrono::time_point<_Clock, _Duration>& __t);
 
    template <class _Clock, class _Duration, class _Predicate>
        bool
        wait_until(unique_lock<mutex>& __lk,
                   const chrono::time_point<_Clock, _Duration>& __t,
                   _Predicate __pred);
 
    template <class _Rep, class _Period>
        cv_status
        wait_for(unique_lock<mutex>& __lk,
                 const chrono::duration<_Rep, _Period>& __d);
 
    template <class _Rep, class _Period, class _Predicate>
        bool
        wait_for(unique_lock<mutex>& __lk,
                 const chrono::duration<_Rep, _Period>& __d,
                 _Predicate __pred);
 
    typedef pthread_cond_t* native_handle_type;
    _LIBCPP_INLINE_VISIBILITY native_handle_type native_handle() {return &__cv_;}
 
private:
    void __do_timed_wait(unique_lock<mutex>& __lk,
       chrono::time_point<chrono::system_clock, chrono::nanoseconds>) _NOEXCEPT;
};
```

里面的函数都是符合直觉的实现，值得注意的是：
cv_status是通过判断时间而确定的，如果超时的则返回cv_status::timeout，如果没有超时，则返回cv_status::no_timeout。

condition_variable::wait_until函数可以传入一个predicate，即一个用户自定义的判断是否符合条件的函数。这个也是很常见的模板编程的方法了。

```cpp
template <class _Clock, class _Duration>
cv_status
condition_variable::wait_until(unique_lock<mutex>& __lk,
                               const chrono::time_point<_Clock, _Duration>& __t)
{
    using namespace chrono;
    wait_for(__lk, __t - _Clock::now());
    return _Clock::now() < __t ? cv_status::no_timeout : cv_status::timeout;
}
 
template <class _Clock, class _Duration, class _Predicate>
bool
condition_variable::wait_until(unique_lock<mutex>& __lk,
                   const chrono::time_point<_Clock, _Duration>& __t,
                   _Predicate __pred)
{
    while (!__pred())
    {
        if (wait_until(__lk, __t) == cv_status::timeout)
            return __pred();
    }
    return true;
}
```

## std::condition_variable_any

std::condition_variable_any的接口和std::condition_variable一样，不同的是 **`std::condition_variable`只能使用`std::unique_lock<std::mutex>`，而`std::condition_variable_any`可以使用任何的锁对象。**

下面来看下为什么`std::condition_variable_any`可以使用任意的锁对象。

```cpp
class _LIBCPP_TYPE_VIS condition_variable_any
{
    condition_variable __cv_;
    shared_ptr<mutex>  __mut_;
public:
    condition_variable_any();
 
    void notify_one() _NOEXCEPT;
    void notify_all() _NOEXCEPT;
 
    template <class _Lock>
        void wait(_Lock& __lock);
    template <class _Lock, class _Predicate>
        void wait(_Lock& __lock, _Predicate __pred);
 
    template <class _Lock, class _Clock, class _Duration>
        cv_status
        wait_until(_Lock& __lock,
                   const chrono::time_point<_Clock, _Duration>& __t);
 
    template <class _Lock, class _Clock, class _Duration, class _Predicate>
        bool
        wait_until(_Lock& __lock,
                   const chrono::time_point<_Clock, _Duration>& __t,
                   _Predicate __pred);
 
    template <class _Lock, class _Rep, class _Period>
        cv_status
        wait_for(_Lock& __lock,
                 const chrono::duration<_Rep, _Period>& __d);
 
    template <class _Lock, class _Rep, class _Period, class _Predicate>
        bool
        wait_for(_Lock& __lock,
                 const chrono::duration<_Rep, _Period>& __d,
                 _Predicate __pred);
};
```

可以看到，在s`td::condition_variable_any`里，用`shared_ptr<mutex>  __mut_`来包装了mutex。所以一切都明白了，**回顾`std::unique_lock<std::mutex>`，它包装了mutex，当析构时自动释放mutex。在`std::condition_variable_any`里，这份工作让`shared_ptr<mutex>`来做了。**

因此，也可以很轻松得出`std::condition_variable_any`会比`std::condition_variable`稍慢的结论了。

## 其它的东东

sched_yield()函数的man手册：

> sched_yield() causes the calling thread to relinquish the CPU.  The thread is moved to the end of the queue for its static priority and a new thread gets to run.  

在C++14里还有`std::shared_lock`和`std::shared_timed_mutex`，但是libc++里还没有对应的实现，因此不做分析。

## 总结

llvm libc++中的各种mutex, lock, condition variable实际上是封闭了posix里的对应实现。封装的技巧和一些细节值得细细推敲学习。

看完了实现源码之后，对于如何使用就更加清晰了。

## 参考

* http://en.cppreference.com/w/cpp
* http://libcxx.llvm.org/