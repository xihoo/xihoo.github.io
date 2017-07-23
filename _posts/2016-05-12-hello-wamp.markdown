---
layout:     post
title:      "基于Reactor模式的网络库(二)"
subtitle:   " \"事件分发对象：Channel class\""
date:       2017-05-10 
author:     "xihoo"
tags:
    - C++
    - Linux网络编程
---

## Reactor模式的关键结构之 Channel class;

``` c++

// Network/reactor/src_code/Channel.h
#ifndef xihoo_NET_CHANNEL_H
#define xihoo_NET_CHANNEL_H

#include <boost/function.hpp>
#include <boost/noncopyable.hpp>

#include <datetime/Timestamp.h>

namespace xihoo
{

class EventLoop;

class Channel : boost::noncopyable
{
 public:
  typedef boost::function<void()> EventCallback;
  typedef boost::function<void(Timestamp)> ReadEventCallback;

  Channel(EventLoop* loop, int fd);
  ~Channel();

  void handleEvent(Timestamp receiveTime);
  void setReadCallback(const ReadEventCallback& cb)
  { readCallback_ = cb; }
  void setWriteCallback(const EventCallback& cb)
  { writeCallback_ = cb; }
  void setErrorCallback(const EventCallback& cb)
  { errorCallback_ = cb; }
  void setCloseCallback(const EventCallback& cb)
  { closeCallback_ = cb; }

  int fd() const { return fd_; }
  int events() const { return events_; }
  void set_revents(int revt) { revents_ = revt; }
  bool isNoneEvent() const { return events_ == kNoneEvent; }

  void enableReading() { events_ |= kReadEvent; update(); }
  void enableWriting() { events_ |= kWriteEvent; update(); }
  void disableWriting() { events_ &= ~kWriteEvent; update(); }
  void disableAll() { events_ = kNoneEvent; update(); }
  bool isWriting() const { return events_ & kWriteEvent; }

  // for Poller
  int index() { return index_; }
  void set_index(int idx) { index_ = idx; }

  EventLoop* ownerLoop() { return loop_; }

 private:
  void update();

  static const int kNoneEvent;
  static const int kReadEvent;
  static const int kWriteEvent;

  EventLoop* loop_;
  const int  fd_;
  int        events_;
  int        revents_;
  int        index_; 

  bool eventHandling_;

  ReadEventCallback readCallback_;
  EventCallback writeCallback_;
  EventCallback errorCallback_;
  EventCallback closeCallback_;
};

}
#endif  // xihoo_NET_CHANNEL_H

```
***

## Channel对象的成员解释

* **handleEvent()**：这是Channel的核心，当fd对应的事件就绪后Channel::handleEvent()执行相应的事件回调，如可读事件执行readCallback_()
* **setReadCallback**：可读事件回调
* **setWriteCallback**：可写事件回调
* **fd()**：返回Channel负责的文件描述符fd，即建立Channel到fd的映射
* **events()**：返回fd域注册的事件类型
* **set_revents()**：设定fd的就绪事件类型，再poll返回就绪事件后将就绪事件类型传给此函数，然后此函数传给handleEvent，handleEvent根据就绪事件的类型决定执行哪个事件回调函数
* **isNoneEvent()**：fd没有想要注册的事件
* **enableReading()**：fd注册可读事件
* **enableWriting()**：fd注册可写事件
* **index()**：index_是本Channel负责的fd在poll监听事件集合的下标，用于快速索引到fd的pollfd
* **update()**：`Channel::update(this)->EventLoop::updateChannel(Channel*)->Poller::updateChannel(Channel*)`最后Poller修改Channel，若Channel已经存在于Poller的`vector<pollfd> pollfds_`(其中`Channel::index_`是vector的下标)则表明Channel要重新注册事件，Poller调用`Channel::events()`获得事件并重置vector中的pollfd;若Channel没有在vector中则向Poller的vector添加新的文件描述符事件到事件表中，并将vector.size(),(vector每次最后追加)，给`Channel::set_index()`作为Channel记住自己在Poller中的位置
* **loop_**：Channel隶属的EventLoop(原则上EventLoop，Poller，Channel都是一个IO线程)
* **fd_**：每个Channel唯一负责的文件描述符，Channel不拥有fd
* **events_**：fd_注册的事件
* **revents_**：通过poll返回的就绪事件类型
* **index_**：在poll的监听事件集合pollfd的下标，用于快速索引到fd的pollfd

***

## Channel原理解析

handleEvent()函数：

``` c++

void Channel::handleEvent(Timestamp receiveTime)
{
  eventHandling_ = true;
  if (revents_ & POLLNVAL) {
    LOG_WARN << "Channel::handle_event() POLLNVAL";
  }

  if ((revents_ & POLLHUP) && !(revents_ & POLLIN)) {
    LOG_WARN << "Channel::handle_event() POLLHUP";
    if (closeCallback_) closeCallback_();
  }
  if (revents_ & (POLLERR | POLLNVAL)) {
    if (errorCallback_) errorCallback_();
  }
  if (revents_ & (POLLIN | POLLPRI | POLLRDHUP)) {
    if (readCallback_) readCallback_(receiveTime);
  }
  if (revents_ & POLLOUT) {
    if (writeCallback_) writeCallback_();
  }
  eventHandling_ = false;
}

```

* 每个Channel对象在其生命期内只属于一个eventloop，只负责一个文件描述符的IO事件分发
* Channel会把不同的IO事件分发为不同的回调
* 用户一般使用更高层的封装，例如TcpConnection


