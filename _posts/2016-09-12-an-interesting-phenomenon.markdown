---
layout:     post
title:      "基于Reactor模式的网络库(四)"
subtitle:   " \"简析Eventloop Channel Epoller的合作\""
date:       2017-05-12 
author:     "xihoo"
tags:
    - C++
    - Linux网络编程
---

## 更新epoll的监听事件

如果想要监听一个事件，可以先建立这个事件的时间分发器Channel，然后
`Channel::update(this)->EventLoop::updateChannel(Channel*)->EPoller::updateChannel(Channel*)`
在EPoller中执行`epoll_ctl(epollfd_, operation, fd, &event)`注册监听的IO事件。  

在Channel中封装了函数`enableReading()` `enableWriting()` `disableWriting()` `disableAll()`
其中调用了`update()`函数。  

## 监听IO事件

EventLoop的`loop()`函数调用了EPoller中的`poll()`函数，`poll()`函数监听注册的文件描述符(IO事件)
一旦事件发生，便调用填充的事件分发器`activeChannels_`的`handleEvent()`函数，执行回调函数。  

