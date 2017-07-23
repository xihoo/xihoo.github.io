---
layout:     post
title:      "基于Reactor模式的网络库(三)"
subtitle:   " \"IO multiplexing的封装 EPoller class\""
date:       2017-05-11 
author:     "xihoo"
tags:
    - C++
    - Linux网络编程
---

## Reactor模式的关键结构之 EPoller class

``` c++

// Network/reactor/src_code/EPoller.h
#ifndef MUDUO_NET_EPOLLER_H
#define MUDUO_NET_EPOLLER_H

#include <map>
#include <vector>

#include "datetime/Timestamp.h"
#include "EventLoop.h"

struct epoll_event;

namespace muduo
{

class Channel;

class EPoller : boost::noncopyable
{
 public:
  typedef std::vector<Channel*> ChannelList;

  EPoller(EventLoop* loop);
  ~EPoller();

  Timestamp poll(int timeoutMs, ChannelList* activeChannels);
  void updateChannel(Channel* channel);
  void removeChannel(Channel* channel);

  void assertInLoopThread() { ownerLoop_->assertInLoopThread(); }

 private:
  static const int kInitEventListSize = 16;

  void fillActiveChannels(int numEvents,
                          ChannelList* activeChannels) const;
  void update(int operation, Channel* channel);

  typedef std::vector<struct epoll_event> EventList;
  typedef std::map<int, Channel*> ChannelMap;

  EventLoop* ownerLoop_;
  int epollfd_;
  EventList events_;
  ChannelMap channels_;
};

}
#endif  // MUDUO_NET_EPOLLER_H

```

***

## EPoller对象的成员解释

* **Poll(int timeoutMs,ChannelList* activeChannels)**：Poller的核心功能，通过poll系统调用将就绪事件集合通过activeChannels返回，并`EventLoop::loop()->Channel::handelEvent()`执行相应的就绪事件回调
* **updateChannel(Channel* channel)**：`Channel::update(this)->EventLoop::updateChannel(Channel*)->Poller::updateChannel(Channel*)`负责维护和更新pollfs_和channels_,更新或添加Channel到Poller的pollfds_和channels_中(主要是文件描述符fd对应的Channel可能想修改已经向poll注册的事件或者fd想向poll注册事件)
* **removeChannel(Channel* channel)**：通过EventLoop::removeChannel(Channel*)->Poller::removeChannle(Channel*)注销pollfds_和channels_中的Channel
* **fillActiveChannels(int numEvents,ChannelList* activeChannels) const**：遍历pollfds_找出就绪事件的fd填入activeChannls,这里不能一边遍历pollfds_一边执行`Channel::handleEvent()`因为后者可能添加或者删除Poller中含Channel的pollfds_和channels_(遍历容器的同时存在容器可能被修改是危险的),所以Poller仅仅是负责IO复用，不负责事件分发(交给Channel处理)
* **ownerLoop_**：隶属的EventLoop
* **pollfds_**：监听事件集合
* **channels_**：文件描述符fd到Channel的映射

***

## Epoller原理解析

Epoller的核心函数poll：

``` c++

Timestamp EPoller::poll(int timeoutMs, ChannelList* activeChannels)
{
  int numEvents = ::epoll_wait(epollfd_,
                               &*events_.begin(),
                               static_cast<int>(events_.size()),
                               timeoutMs);
  Timestamp now(Timestamp::now());
  if (numEvents > 0)
  {
    LOG_TRACE << numEvents << " events happended";
    fillActiveChannels(numEvents, activeChannels);
    if (implicit_cast<size_t>(numEvents) == events_.size())
    {
      events_.resize(events_.size()*2);
    }
  }
  else if (numEvents == 0)
  {
    LOG_TRACE << " nothing happended";
  }
  else
  {
    LOG_SYSERR << "EPoller::poll()";
  }
  return now;
}

```

* poll函数调用epoll获得当前活动的IO事件，然后填充调用方传入的activeChannels，并返回return的时刻