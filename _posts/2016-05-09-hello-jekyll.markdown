---
layout:     post
title:      "基于Reactor模式的多线程网络库(一)"
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

### Reactor模式最基本的class--EvenLoop

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

