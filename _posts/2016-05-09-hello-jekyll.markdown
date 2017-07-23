---
layout:     post
title:      "基于Reactor模式的网络库(一)"
subtitle:   " \"EventLoop事件循环\""
date:       2017-05-09 10:25:07 +0800
author:     "xihoo"
tags:
    - C++
    - Linux网络编程
---


### Reactor模式

  >Wikipedia:"The reactor design pattern is an event handling pattern for handling service requests delivered concurrently by one or more inputs. The service handler then demultiplexes the incoming requests and dispatches them synchronously to associated request handlers."

  Reactor模式由事件驱动，有一个或多个并发输入源，有一个Service Handler，有多个Request Handlers；这个Service Handler会同步的将输入的请求（Event）多路复用的分发给相应的Request Handler

  ![](/img/Reactor_Simple.png)

***

### Reactor模式最基本的class:EvenLoop

``` c++
// Network/reactor/src_code
#ifndef xihoo_NET_EVENTLOOP_H
#define xihoo_NET_EVENTLOOP_H

#include "datetime/Timestamp.h"
#include "thread/Mutex.h"
#include "thread/Thread.h"
#include "Callbacks.h"
#include "TimerId.h"

#include <boost/scoped_ptr.hpp>
#include <vector>

namespace xihoo
{

class Channel;
class EPoller;
class TimerQueue;

class EventLoop : boost::noncopyable
{
 public:
  typedef boost::function<void()> Functor;
  EventLoop();
  ~EventLoop();
  void loop();
  void quit();
  Timestamp pollReturnTime() const { return pollReturnTime_; }
  void runInLoop(const Functor& cb);
  void queueInLoop(const Functor& cb);
  TimerId runAt(const Timestamp& time, const TimerCallback& cb);
  TimerId runAfter(double delay, const TimerCallback& cb);
  TimerId runEvery(double interval, const TimerCallback& cb);
  void cancel(TimerId timerId);
  void wakeup();
  void updateChannel(Channel* channel);
  void removeChannel(Channel* channel);

  void assertInLoopThread()
  {
    if (!isInLoopThread())
    {
      abortNotInLoopThread();
    }
  }

  bool isInLoopThread() const { return threadId_ == CurrentThread::tid(); }

 private:

  void abortNotInLoopThread();
  void handleRead();  
  void doPendingFunctors();

  typedef std::vector<Channel*> ChannelList;

  bool looping_; 
  bool quit_; 
  bool callingPendingFunctors_; 
  const pid_t threadId_;
  Timestamp pollReturnTime_;
  boost::scoped_ptr<EPoller> poller_;
  boost::scoped_ptr<TimerQueue> timerQueue_;
  int wakeupFd_;
  boost::scoped_ptr<Channel> wakeupChannel_;
  ChannelList activeChannels_;
  MutexLock mutex_;
  std::vector<Functor> pendingFunctors_;
};

}

#endif  


```
EventLoop: 事件循环，一个线程一个事件循环即one loop per thread，其主要功能是运行事件循环如等待事件发生然后处理发生的事件

***

### EventLoop成员解释

* **loop()函数**：EventLoop的主体,用于事件循环，`Eventloop::loop()->Poller::Poll()`获得就绪的事件集合并通过Channel::handleEvent()执行就绪事件回调
* **quit()函数**：终止事件循环，通过设定标志位所以有一定延迟
* **assertInLoopThread()函数**：若运行线程不拥有EventLoop则退出，保证每个线程有一个事件循环
* **runInLoop()函数**：用于IO线程执行用户回调(如EventLoop由于执行事件回调阻塞了，此时用户希望唤醒EventLoop执行用户指定的任务)
* **queueInLoop()函数**：唤醒IO线程(拥有此EventLoop的线程)并将用户指定的任务回调放入队列
* **wakeup()函数**：唤醒IO线程
* **updateChannel()函数**：更新事件分发器Channel，完成文件描述符fd向事件集合注册事件及事件回调函数
* **abortNotInLoopThread()函数**：在不拥有EventLoop线程中终止
* **doPendingFunctors()函数**：执行队列pendingFunctors中的用户任务回调
* **handleRead()函数**：timerfd上可读事件回调
* **ChannelList**：事件分发器Channel容器，一个Channel只负责一个文件描述符fd的事件分发
* **looping_**：事件循环主体loop是运行标志
* **quit_**：取消循环主体标志
* **callingPendingFunctors_**：是否有用户任务回调标志
* **threadID_**：EventLoop的附属线程ID
* **poller_**：IO复用器Poller用于监听事件集合
* **activeChannels_**：类似与poll的就绪事件集合，这里集合换成Channel(事件分发器具备就绪事件回调功能)
* **wakeupFd_**：eventfd用于唤醒EventLoop所在线程
* **wakeupChannel_** ：通过wakeupChannel_观察wakeupFd_上的可读事件，当可读时表明需要唤醒EventLoop所在线程执行用户回调 
* **mutex_**：互斥量用以保护队列
* **pendingFunctors_**：用户任务回调队列
* **timerQueue_**：定时器队列用于存放定时器

***
### Evenloop原理解析

每个线程有且只能有一个Eventloop对象，Eventloop的核心功能是loop()函数
``` c++

void EventLoop::loop()
{
  assert(!looping_);
  assertInLoopThread();
  looping_ = true;
  quit_ = false;

  while (!quit_)
  {
    activeChannels_.clear();
    pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);
    for (ChannelList::iterator it = activeChannels_.begin();
        it != activeChannels_.end(); ++it)
    {
      (*it)->handleEvent(pollReturnTime_);
    }
    doPendingFunctors();
  }

  LOG_TRACE << "EventLoop " << this << " stop looping";
  looping_ = false;
}

```

**loop()函数使用IO多路复用监听事件，事件发生后调用Channel对象进行事件分发处理**



