---
layout:     post
title:      "EventLoop的跨线程调用函数"
subtitle:   " \"简析runInLoop()函数的机制\""
date:       2017-06-10 
author:     "xihoo"
tags:
    - C++
    - Linux网络编程
---

## runInLoop()函数

```c++

void EventLoop::runInLoop(const Functor& cb)
{
  if (isInLoopThread())
  {
    cb();
  }
  else
  {
    queueInLoop(cb);
  }
}

```

在当前IO线程时就调用用户回调；  
在其他线程调用runInLoop()函数时，cb被加入队列，IO线程被唤醒来调用cb；

***

## queueInLoop()函数

```c++

void EventLoop::queueInLoop(const Functor& cb)
{
  {
  MutexLockGuard lock(mutex_);
  pendingFunctors_.push_back(cb);
  }

  if (!isInLoopThread() || callingPendingFunctors_)
  {
    wakeup();
  }
}

```

`pendingFunctors_`是暴露在其他线程之下的，故需要用互斥量来保护。  
将cb放入队列之后，如果不在IO线程中或是在IO线程中但是正在调用pendingfunctors则会唤醒IO线程；  

***

## wakeup()函数

```c++

void EventLoop::wakeup()
{
  uint64_t one = 1;
  ssize_t n = ::write(wakeupFd_, &one, sizeof one);
  if (n != sizeof one)
  {
    LOG_ERROR << "EventLoop::wakeup() writes " << n << " bytes instead of 8";
  }
}

```

唤醒IO线程的机制是：向`wakeupFd_`这个文件描述符输入数据，这样阻塞在epoll上的`loop()`函数
监听到readable事件就可以唤醒IO线程。  

其中wakeupChannel_负责处理`wakeupFd_`上的readable事件，将事件分发至`handleRead()`函数  

`handleRead()`读取数据，**数据通过eventfd(Linux系统调用)进行线程间的通信**

## doPendingFunctors()函数

```c++

void EventLoop::doPendingFunctors()
{
  std::vector<Functor> functors;
  callingPendingFunctors_ = true;

  {
  MutexLockGuard lock(mutex_);
  functors.swap(pendingFunctors_);
  }

  for (size_t i = 0; i < functors.size(); ++i)
  {
    functors[i]();
  }
  callingPendingFunctors_ = false;
}

```

唤醒IO线程之后执行队列中的回调函数；  
**注意**：doPendingFunctors()不是简单的在临界区依次调用函数，而是将其`swap`到局部变量functors中
1. 减小了临界区长度，不会阻塞其他线程调用`queueInLoop()`函数；
2. 避免死锁，因为回调可能**再次**调用`queueInLoop()`函数；