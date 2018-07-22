---
title: C++11的thread代码分析
date: 2014-06-29 19:58:28
tags:
 - cpp
 - cpp11
 - posix


categories:
  - 技术

---

## 前言

本文分析的是llvm libc++的实现：http://libcxx.llvm.org/

## class thread


thread类直接包装了一个pthread_t，在linux下实际是unsigned long int。

```cpp
class  thread
{
    pthread_t __t_;
```

```cpp
    id get_id() const _NOEXCEPT {return __t_;}
}
```

用了一个std::unique_ptr来包装用户定义的线程函数：

创建线程用的是

```cpp
template <class _Fp>
void*
__thread_proxy(void* __vp)
{
    __thread_local_data().reset(new __thread_struct);
    std::unique_ptr<_Fp> __p(static_cast<_Fp*>(__vp));
    (*__p)();
    return nullptr;
}
 
template <class _Fp>
thread::thread(_Fp __f)
{
    std::unique_ptr<_Fp> __p(new _Fp(__f));
    int __ec = pthread_create(&__t_, 0, &__thread_proxy<_Fp>, __p.get());
    if (__ec == 0)
        __p.release();
    else
        __throw_system_error(__ec, "thread constructor failed");
}

```

## thread::joinable() , thread::join(), thread::detach() 

再来看下thread::joinable() , thread::join(), thread::detach() 函数。
也是相应调用了posix的函数。在调用join()之后，会把_t设置为0，这样再调用joinable()时就会返回false。**对于_t变量没有memory barrier同步，感觉可能会有问题。**

```cpp
bool joinable() const {return __t_ != 0;}
void
thread::join()
{
    int ec = pthread_join(__t_, 0);
    __t_ = 0;
}
 
void
thread::detach()
{
    int ec = EINVAL;
    if (__t_ != 0)
    {
        ec = pthread_detach(__t_);
        if (ec == 0)
            __t_ = 0;
    }
    if (ec)
        throw system_error(error_code(ec, system_category()), "thread::detach failed");
}
```

## thread::hardware_concurrency()

thread::hardware_concurrency()函数，获取的是当前可用的processor的数量。
调用的是sysconf(_SC_NPROCESSORS_ONLN)函数，据man手册：

```
        - _SC_NPROCESSORS_ONLN
              The number of processors currently online (available).
```

```cpp
unsigned
thread::hardware_concurrency() _NOEXCEPT
{
    long result = sysconf(_SC_NPROCESSORS_ONLN);
    // sysconf returns -1 if the name is invalid, the option does not exist or
    // does not have a definite limit.
    // if sysconf returns some other negative number, we have no idea
    // what is going on. Default to something safe.
    if (result < 0)
        return 0;
    return static_cast<unsigned>(result);
}
```

## thread::sleep_for和thread::sleep_until

sleep_for函数实际调用的是nanosleep函数：

```cpp
void
sleep_for(const chrono::nanoseconds& ns)
{
    using namespace chrono;
    if (ns > nanoseconds::zero())
    {
        seconds s = duration_cast<seconds>(ns);
        timespec ts;
        typedef decltype(ts.tv_sec) ts_sec;
        _LIBCPP_CONSTEXPR ts_sec ts_sec_max = numeric_limits<ts_sec>::max();
        if (s.count() < ts_sec_max)
        {
            ts.tv_sec = static_cast<ts_sec>(s.count());
            ts.tv_nsec = static_cast<decltype(ts.tv_nsec)>((ns-s).count());
        }
        else
        {
            ts.tv_sec = ts_sec_max;
            ts.tv_nsec = giga::num - 1;
        }
 
        while (nanosleep(&ts, &ts) == -1 && errno == EINTR)
            ;
    }
}
```

sleep_until函数用到了mutex, condition_variable, unique_lock，实际上调用的还是pthread_cond_timedwait函数：

```cpp
template <class _Clock, class _Duration>
void
sleep_until(const chrono::time_point<_Clock, _Duration>& __t)
{
    using namespace chrono;
    mutex __mut;
    condition_variable __cv;
    unique_lock<mutex> __lk(__mut);
    while (_Clock::now() < __t)
        __cv.wait_until(__lk, __t);
}
 
void
condition_variable::__do_timed_wait(unique_lock<mutex>& lk,
     chrono::time_point<chrono::system_clock, chrono::nanoseconds> tp) _NOEXCEPT
{
    using namespace chrono;
    if (!lk.owns_lock())
        __throw_system_error(EPERM,
                            "condition_variable::timed wait: mutex not locked");
    nanoseconds d = tp.time_since_epoch();
    if (d > nanoseconds(0x59682F000000E941))
        d = nanoseconds(0x59682F000000E941);
    timespec ts;
    seconds s = duration_cast<seconds>(d);
    typedef decltype(ts.tv_sec) ts_sec;
    _LIBCPP_CONSTEXPR ts_sec ts_sec_max = numeric_limits<ts_sec>::max();
    if (s.count() < ts_sec_max)
    {
        ts.tv_sec = static_cast<ts_sec>(s.count());
        ts.tv_nsec = static_cast<decltype(ts.tv_nsec)>((d - s).count());
    }
    else
    {
        ts.tv_sec = ts_sec_max;
        ts.tv_nsec = giga::num - 1;
    }
    int ec = pthread_cond_timedwait(&__cv_, lk.mutex()->native_handle(), &ts);
    if (ec != 0 && ec != ETIMEDOUT)
        __throw_system_error(ec, "condition_variable timed_wait failed");
}
```

## std::notify_all_at_thread_exit 的实现

先来看个例子，这个notify_all_at_thread_exit函数到底有什么用：

```cpp
#include <mutex>
#include <thread>
#include <condtion_variable>
 
std::mutex m;
std::condition_variable cv;
 
bool ready = false;
ComplexType result;  // some arbitrary type
 
void thread_func()
{
    std::unique_lock<std::mutex> lk(m);
    // assign a value to result using thread_local data
    result = function_that_uses_thread_locals();
    ready = true;
    std::notify_all_at_thread_exit(cv, std::move(lk));
} // 1. destroy thread_locals, 2. unlock mutex, 3. notify cv
 
int main()
{
    std::thread t(thread_func);
    t.detach();
 
    // do other work
    // ...
 
    // wait for the detached thread
    std::unique_lock<std::mutex> lk(m);
    while(!ready) {
        cv.wait(lk);
    }
    process(result); // result is ready and thread_local destructors have finished
}
```

可以看到std::notify_all_at_thread_exit 函数，实际上是注册了一对condition_variable，mutex，当线程退出时，notify_all。

下面来看下具体的实现：

这个是通过Thread-specific Data来实现的，具体可以参考：http://www.ibm.com/developerworks/cn/linux/thread/posix_threadapi/part2/

但我个人觉得这个应该叫线程特定数据比较好，因为它是可以被别的线程访问的，而不是某个线程”专有“的。

简而言之，std::thread在构造的时候，创建了一个__thread_struct_imp对象。

__thread_struct_imp对象里，用一个vector来保存了`pair<condition_variable*, mutex*>`：

```cpp
class  __thread_struct_imp
{
    typedef vector<__assoc_sub_state*,
                          __hidden_allocator<__assoc_sub_state*> > _AsyncStates;
<strong>    typedef vector<pair<condition_variable*, mutex*>,
               __hidden_allocator<pair<condition_variable*, mutex*> > > _Notify;</strong>
 
    _AsyncStates async_states_;
    _Notify notify_;
```

**当调用notify_all_at_thread_exit函数时，把condition_variable和mutex，push到vector里：**

```cpp
void
__thread_struct_imp::notify_all_at_thread_exit(condition_variable* cv, mutex* m)
{
    notify_.push_back(pair<condition_variable*, mutex*>(cv, m));
}
```

当线程退出时，会delete掉__thread_struct_imp，也就是会调用__thread_struct_imp的析构函数。

**而在析构函数里，会调用历遍vector，unlock每个mutex，和调用condition_variable.notify_all()函数：**

```cpp
__thread_struct_imp::~__thread_struct_imp()
{
    for (_Notify::iterator i = notify_.begin(), e = notify_.end();
            i != e; ++i)
    {
        i->second->unlock();
        i->first->notify_all();
    }
    for (_AsyncStates::iterator i = async_states_.begin(), e = async_states_.end();
            i != e; ++i)
    {
        (*i)->__make_ready();
        (*i)->__release_shared();
    }
}
```

更详细的一些封闭代码，我提取出来放到了gist上：https://gist.github.com/hengyunabc/d48fbebdb9bddcdf05e9

## 其它的一些东东

关于线程的yield, detch, join，可以直接参考man文档：

```
pthread_yield:

       pthread_yield() causes the calling thread to relinquish the CPU.  The
       thread is placed at the end of the run queue for its static priority
       and another thread is scheduled to run.  For further details, see
       sched_yield(2)
pthread_detach:
       The pthread_detach() function marks the thread identified by thread
       as detached.  When a detached thread terminates, its resources are
       automatically released back to the system without the need for
       another thread to join with the terminated thread.

       Attempting to detach an already detached thread results in
       unspecified behavior.
pthread_join:
       The pthread_join() function waits for the thread specified by thread
       to terminate.  If that thread has already terminated, then
       pthread_join() returns immediately.  The thread specified by thread
       must be joinable.
```

## 总结

个人感觉像 join, detach这两个函数实际没多大用处。绝大部分情况下，线程创建之后，都应该detach掉。

像join这种同步机制不如换mutex等更好。

## 参考

* http://en.cppreference.com/w/cpp/thread/notify_all_at_thread_exit
* http://man7.org/linux/man-pages/man3/pthread_detach.3.html
* http://man7.org/linux/man-pages/man3/pthread_join.3.html
* http://stackoverflow.com/questions/19744250/c11-what-happens-to-a-detached-thread-when-main-exits
* http://man7.org/linux/man-pages/man3/pthread_yield.3.html
* http://man7.org/linux/man-pages/man2/sched_yield.2.html
* http://www.ibm.com/developerworks/cn/linux/thread/posix_threadapi/part2/
* man pthread_key_create