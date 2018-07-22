---
title: 并行编程之条件变量（posix condition variables）
date: 2014-06-02 19:58:28
tags:
 - cpp
 - cpp11
 - condition
 - posix


categories:
  - 技术

---

## 前言

在整理Java LockSupport.park()的东东，看到了个"Spurious wakeup"，重新梳理下。

首先来个《UNIX环境高级编程》里的例子：

```cpp
#include <pthread.h>
struct msg {
	struct msg *m_next;
	/* ... more stuff here ... */
};
struct msg *workq;
pthread_cond_t qready = PTHREAD_COND_INITIALIZER;
pthread_mutex_t qlock = PTHREAD_MUTEX_INITIALIZER;
void process_msg(void) {
	struct msg *mp;
	for (;;) {
		pthread_mutex_lock(&qlock);
		while (workq == NULL)
			pthread_cond_wait(&qready, &qlock);
		mp = workq;
		workq = mp->m_next;
		pthread_mutex_unlock(&qlock);
		/* now process the message mp */
	}
}
void enqueue_msg(struct msg *mp) {
	pthread_mutex_lock(&qlock);
	mp->m_next = workq;
	workq = mp;
	pthread_mutex_unlock(&qlock);
	pthread_cond_signal(&qready);
}
```

一个简单的消息生产者和消费者的代码。它们之间用condition同步。
这个代码最容易让人搞混的是process_msg函数里的pthread_mutex_lock 和 pthread_mutex_unlock 是一对函数调用，前面加锁，后面解锁。的确，是加锁解锁，但是它们两不是一对的。它们的另一半在pthread_cond_wait函数里。

pthread_cond_wait函数可以认为它做了三件事：

* 把自身线程放到condition的等待队列里，把mutex解锁；
* 等待被唤醒（当其它线程调用pthread_cond_signal或者pthread_cond_broadcast时）；
* 被唤醒之后，对metex加锁，再返回。

mutex和condition实际上是绑定在一起的，一个condition只能对应一个mutex。在Java的代码里，Condition对象只能通过lock.newCondition()的函数来获取。


## Spurious wakeup

所谓的spurious wakeup，指的是一个线程调用pthread_cond_signal()，却有可能不止一个线程被唤醒。为什么会出现这种情况？wiki和其它的一些文档都只是说在多核的情况下，简化实现允许出现这种spurious wakeup。。

在man文档里给出了一个可能的实现，然后解析为什么会出现。

假定有三个线程，线程A正在执行pthread_cond_wait，线程B正在执行pthread_cond_signal，线程C正准备执行pthread_cond_wait函数。

```cpp
              pthread_cond_wait(mutex, cond):
                  value = cond->value; /* 1 */
                  pthread_mutex_unlock(mutex); /* 2 */
                  pthread_mutex_lock(cond->mutex); /* 10 */
                  if (value == cond->value) { /* 11 */
                      me->next_cond = cond->waiter;
                      cond->waiter = me;
                      pthread_mutex_unlock(cond->mutex);
                      unable_to_run(me);
                  } else
                      pthread_mutex_unlock(cond->mutex); /* 12 */
                  pthread_mutex_lock(mutex); /* 13 */
 
 
              pthread_cond_signal(cond):
                  pthread_mutex_lock(cond->mutex); /* 3 */
                  cond->value++; /* 4 */
                  if (cond->waiter) { /* 5 */
                      sleeper = cond->waiter; /* 6 */
                      cond->waiter = sleeper->next_cond; /* 7 */
                      able_to_run(sleeper); /* 8 */
                  }
                  pthread_mutex_unlock(cond->mutex); /* 9 */
```

线程A执行了第1,2步，这时它释放了mutex，然后线程B拿到了这个mutext，并且pthread_cond_signal函数时执行并返回了。于是线程B就是一个所谓的“spurious wakeup”。

为什么pthread_cond_wait函数里一进入，就释放了mutex？没有找到什么解析。。

查看了glibc的源代码，大概可以看出上面的一些影子，但是太复杂了，也没有搞明白为什么。。

* /build/buildd/eglibc-2.19/nptl/pthread_cond_wait.c
* /build/buildd/eglibc-2.19/nptl/pthread_cond_signal.c

不过从上面的解析，可以发现《UNIX高级编程》里的说明是错误的（可能是因为太久了）。

> The  caller passes it locked to the function, which then atomically places the calling thread on the list of threads waiting for the condition and unlocks the mutex. 

上面的伪代码，一进入pthread_cond_wait函数就释放了mutex，明显和书里的不一样。

## wait morphing优化

在《UNIX环境高级编程》的示例代码里，是先调用pthread_mutex_unlock，再调用pthread_cond_signal。

```cpp
void enqueue_msg(struct msg *mp) {
	pthread_mutex_lock(&qlock);
	mp->m_next = workq;
	workq = mp;
	pthread_mutex_unlock(&qlock);
	pthread_cond_signal(&qready);
}
```

有的地方给出的是先调用pthread_cond_signal，再调用pthread_mutex_unlock：

```cpp
void enqueue_msg(struct msg *mp) {
	pthread_mutex_lock(&qlock);
	mp->m_next = workq;
	workq = mp;
	pthread_cond_signal(&qready);
	pthread_mutex_unlock(&qlock);
}
```

先unlock再signal，这有个好处，就是调用enqueue_msg的线程可以再次参与mutex的竞争中，这样意味着可以连续放入多个消息，这个可能会提高效率。类似Java里ReentrantLock的非公平模式。
网上有些文章说，先singal再unlock，有可能会出现一种情况是被singal唤醒的线程会因为不能马上拿到mutex（还没被释放），从而会再次休眠，这样影响了效率。从而会有一个叫“wait morphing”优化，就是如果线程被唤醒但是不能获取到mutex,则线程被转移(morphing)到mutex的等待队列里。

但是我查看了下glibc的源代码，貌似没有发现有这种“wait morphing”优化。

man文档里提到：

> The pthread_cond_broadcast() or pthread_cond_signal() functions may be called by a thread whether or not it currently owns the mutex that  threads  calling  pthread_cond_wait()  or pthread_cond_timedwait() have associated with the condition variable during their waits; however, if predictable scheduling behavior is required, then that mutex shall be locked by the thread calling pthread_cond_broadcast() or pthread_cond_signal().

可见在调用singal之前，可以不持有mutex，除非是“predictable scheduling”，可预测的调度行为。这种可能是实时系统才有这种严格的要求。

## 为什么要用while循环来判断条件是否成立？

```cpp
		while (workq == NULL)
			pthread_cond_wait(&qready, &qlock);
```

而不用if来判断？

```cpp
		if (workq == NULL)
			pthread_cond_wait(&qready, &qlock);
```

**一个原因是spurious wakeup，但即使没有spurious wakeup，也是要用While来判断的。**

比如线程A，线程B在pthread_cond_wait函数中等待，然后线程C把消息放到队列里，再调用pthread_cond_broadcast，然后线程A先获取到mutex，处理完消息完后，这时workq就变成NULL了。这时线程B才获取到mutex，那么这时实际上是没有资源供线程B使用的。所以从pthread_cond_wait函数返回之后，还是要判断条件是否成功，如果成立，再进行处理。

## pthread_cond_signal和pthread_cond_broadcast

在这篇文章里，http://www.cppblog.com/Solstice/archive/2013/09/09/203094.html

给出的示例代码7里，认为调用pthread_cond_broadcast来唤醒所有的线程是比较好的写法。但是我认为pthread_cond_signal和pthread_cond_broadcast是两个不同东东，不能简单合并在同一个函数调用。只唤醒一个效率和唤醒全部等待线程的效率显然不能等同。典型的condition是用CLH或者MCS来实现的，要通知所有的线程，则要历遍链表，显然效率降低。另外，C++11里的condition_variable也提供了notify_one函数。

http://en.cppreference.com/w/cpp/thread/condition_variable/notify_one

## mutex，condition是不是公平(fair)的？

这个在参考文档里没有说明，在网上找了些资料，也没有什么明确的答案。

我写了个代码测试，发现mutex是公平的。condition的测试结果也是差不多。

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
 
volatile int mutexCount = 0;
void mutexFairTest(){
	int localCount = 0;
	while(1){
		pthread_mutex_lock(&lock);
		__sync_fetch_and_add(&mutexCount, 1);
		localCount += 1;
		if(mutexCount > 100000000){
			break;
		}
		pthread_mutex_unlock(&lock);
	}
	pthread_mutex_unlock(&lock);
	printf("localCount:%d\n", localCount);
}
 
int main() {
	pthread_mutex_lock(&lock);
	pthread_create(new pthread_t, NULL, (void * (*)(void *))&mutexFairTest, NULL);
	pthread_create(new pthread_t, NULL, (void * (*)(void *))&mutexFairTest, NULL);
	pthread_create(new pthread_t, NULL, (void * (*)(void *))&mutexFairTest, NULL);
	pthread_create(new pthread_t, NULL, (void * (*)(void *))&mutexFairTest, NULL);
	pthread_create(new pthread_t, NULL, (void * (*)(void *))&mutexFairTest, NULL);
	pthread_create(new pthread_t, NULL, (void * (*)(void *))&mutexFairTest, NULL);
	pthread_mutex_unlock(&lock);
 
	sleep(100);
}
```

输出结果是：

```
localCount:16930422
localCount:16525616
localCount:16850294
localCount:16129844
localCount:17329693
localCount:16234137
```

还特意在一个单CPU的虚拟机上测试了下。输出的结果差不多。操作系统是ububtu14.04。

## 连续调用pthread_cond_signal，会唤醒多少次/多少个线程？

比如线程a,b 在调用pthread_cond_wait之后等待，然后线程c, d同时调用pthread_cond_signal，那么a, b线程是否都能被唤醒？

会不会出现c, d, a 这种调用顺序，然后b一直在等待，然后死锁了？

根据文档：

> The pthread_cond_signal() function shall unblock at least one of the threads that are blocked on the specified condition variable cond (if any threads are blocked on cond).

**因此，如果有线程已经在调用pthread_cond_wait等待的情况下，pthread_cond_signal调用至少会唤醒等待中的一个线程。**

所以不会出现上面的线程b一直等待的情况。

但是，我们再仔细考虑下：

**如何确认线程a, b 调用pthread_cond_wait完成了？还是只是刚切换到内核态？显然是没有办法知道的。**

所以，我们平时编程肯定不会写这样的代码，应该是共享变量，在获取到锁之后，再修改变量。这样子来做同步。参考上面《UNIX环境高级编程》的例子。

不过，这个问题也是挺有意思的。

## 参考

* http://en.wikipedia.org/wiki/Spurious_wakeup
* http://siwind.iteye.com/blog/1469216
* http://www.cppblog.com/Solstice/archive/2013/09/09/203094.html
* http://www.cs.cmu.edu/afs/cs/academic/class/15492-f07/www/pthreads.html