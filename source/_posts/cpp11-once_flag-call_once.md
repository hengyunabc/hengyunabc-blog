---
title: C++11中once_flag，call_once实现分析
date: 2014-06-02 19:58:28
tags:
 - cpp
 - cpp11


categories:
  - 技术

---

## 前言

本文的分析基于llvm的libc++，而不是gun的libstdc++，因为libstdc++的代码里太多宏了，看起来蛋疼。

在多线程编程中，有一个常见的情景是某个任务只需要执行一次。在C++11中提供了很方便的辅助类once_flag，call_once。

## 声明

首先来看一下once_flag和call_once的声明：

```cpp
struct once_flag
{
    constexpr once_flag() noexcept;
    once_flag(const once_flag&) = delete;
    once_flag& operator=(const once_flag&) = delete;
};
template<class Callable, class ...Args>
  void call_once(once_flag& flag, Callable&& func, Args&&... args);
 
}  // std
```

可以看到once_flag是不允许修改的，拷贝构造函数和operator=函数都声明为delete，这样防止程序员乱用。
另外，call_once也是很简单的，只要传进一个once_flag，回调函数，和参数列表就可以了。

## 示例

看一个示例：

http://en.cppreference.com/w/cpp/thread/call_once

```cpp
#include <iostream>
#include <thread>
#include <mutex>
 
std::once_flag flag;
 
void do_once()
{
    std::call_once(flag, [](){ std::cout << "Called once" << std::endl; });
}
 
int main()
{
    std::thread t1(do_once);
    std::thread t2(do_once);
    std::thread t3(do_once);
    std::thread t4(do_once);
 
    t1.join();
    t2.join();
    t3.join();
    t4.join();
}
```

保存为main.cpp，如果是用g++或者clang++来编绎：

```
g++ -std=c++11 -pthread main.cpp
clang++ -std=c++11 -pthread main.cpp
./a.out 
```

可以看到，只会输出一行

```
Called once
```

值得注意的是，如果在函数执行中抛出了异常，那么会有另一个在once_flag上等待的线程会执行。

比如下面的例子：

```cpp
#include <iostream>
#include <thread>
#include <mutex>
 
std::once_flag flag;
 
inline void may_throw_function(bool do_throw)
{
  // only one instance of this function can be run simultaneously
  if (do_throw) {
    std::cout << "throw\n"; // this message may be printed from 0 to 3 times
    // if function exits via exception, another function selected
    throw std::exception();
  }
 
  std::cout << "once\n"; // printed exactly once, it's guaranteed that
      // there are no messages after it
}
 
inline void do_once(bool do_throw)
{
  try {
    std::call_once(flag, may_throw_function, do_throw);
  }
  catch (...) {
  }
}
 
int main()
{
    std::thread t1(do_once, true);
    std::thread t2(do_once, true);
    std::thread t3(do_once, false);
    std::thread t4(do_once, true);
 
    t1.join();
    t2.join();
    t3.join();
    t4.join();
}
```

输出的结果可能是0到3行throw，和一行once。

**实际上once_flag相当于一个锁，使用它的线程都会在上面等待，只有一个线程允许执行。如果该线程抛出异常，那么从等待中的线程中选择一个，重复上面的流程。**

## 实现分析

once_flag实际上只有一个unsigned long __state_的成员变量，把call_once声明为友元函数，这样call_once能修改__state__变量：

```cpp
struct once_flag
{
        once_flag() _NOEXCEPT : __state_(0) {}
private:
    once_flag(const once_flag&); // = delete;
    once_flag& operator=(const once_flag&); // = delete;
 
    unsigned long __state_;
 
    template<class _Callable>
    friend void call_once(once_flag&, _Callable);
};
```

call_once则用了一个__call_once_param类来包装函数，很常见的模板编程技巧。

```cpp
template <class _Fp>
class __call_once_param
{
    _Fp __f_;
public:
    explicit __call_once_param(const _Fp& __f) : __f_(__f) {}
    void operator()()
    {
        __f_();
    }
};
template<class _Callable>
void call_once(once_flag& __flag, _Callable __func)
{
    if (__flag.__state_ != ~0ul)
    {
        __call_once_param<_Callable> __p(__func);
        __call_once(__flag.__state_, &__p, &__call_once_proxy<_Callable>);
    }
}
```

最重要的是__call_once函数的实现：

```cpp
static pthread_mutex_t mut = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t  cv  = PTHREAD_COND_INITIALIZER;
 
void
__call_once(volatile unsigned long& flag, void* arg, void(*func)(void*))
{
    pthread_mutex_lock(&mut);
    while (flag == 1)
        pthread_cond_wait(&cv, &mut);
    if (flag == 0)
    {
#ifndef _LIBCPP_NO_EXCEPTIONS
        try
        {
#endif  // _LIBCPP_NO_EXCEPTIONS
            flag = 1;
            pthread_mutex_unlock(&mut);
            func(arg);
            pthread_mutex_lock(&mut);
            flag = ~0ul;
            pthread_mutex_unlock(&mut);
            pthread_cond_broadcast(&cv);
#ifndef _LIBCPP_NO_EXCEPTIONS
        }
        catch (...)
        {
            pthread_mutex_lock(&mut);
            flag = 0ul;
            pthread_mutex_unlock(&mut);
            pthread_cond_broadcast(&cv);
            throw;
        }
#endif  // _LIBCPP_NO_EXCEPTIONS
    }
    else
        pthread_mutex_unlock(&mut);
}
```

里面用了全局的mutex和condition来做同步，还有异常处理的代码。
其实当看到mutext和condition时，就明白是如何实现的了。里面有一系列的同步操作，可以参考另外一篇blog：

* http://blog.csdn.net/hengyunabc/article/details/27969613   并行编程之条件变量（posix condition variables）

尽管代码看起来很简单，但是要仔细分析它的各种时序也比较复杂。

有个地方比较疑惑的：

* 对于同步的__state__变量，并没有任何的memory order的保护，会不会有问题？

因为在JDK的代码里LockSupport和逻辑和上面的__call_once函数类似，但是却有memory order相关的代码：

OrderAccess::fence();

## 其它的东东

有个东东值得提一下，在C++中，static变量的初始化，并不是线程安全的。

比如

```cpp
void func(){
    static int value = 100;
    ...
}
```

实际上相当于这样的代码：

```cpp
int __flag = 0
void func(){
    static int value;
    if(!__flag){
        value = 100;
        __flag = 1;
    }
    ...
}
```

## 总结

还有一件事情要考虑：所有的once_flag和call_once都共用全局的mutex和condition会不会有性能问题？

首先，像call_once这样的需求在一个程序里不会太多。另外，临界区的代码是比较很少的，只有判断各自的flag的代码。

如果有上百上千个线程在等待once_flag，那么pthread_cond_broadcast可能会造成“惊群”效果，但是如果有那么多的线程都上等待，显然程序设计有问题。

还有一个要注意的地方是 **once_flag的生命周期，它必须要比使用它的线程的生命周期要长。所以通常定义成全局变量比较好。**

## 参考

* http://libcxx.llvm.org/
* http://en.cppreference.com/w/cpp/thread/once_flag
* http://en.cppreference.com/w/cpp/thread/call_once