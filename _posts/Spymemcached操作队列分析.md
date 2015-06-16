---
date: 2015-01-20 21:50
status: public
author: Haijin Ma
title: 【原创】Spymemcached操作队列分析
categories: [缓存实现]
tags: [memcached]
---

Spymemcached是一个Memcached（一个高性能内存对象缓存系统）的开源客户端程序。其他MC客户端还有xmemcached、Memcached Client for Java等。我们使用MC客户端程序与MC服务端通信，实现set、get缓存的操作，MC支持的协议类型包括：ASCII 文本协议和二进制协议。

Spymemcached使用Java NIO来实现，并且是单线程的，也就是说在Spymemcached中只有一个主循环线程来处理各种IO事件（Connection、Read、Write）。对于业务线程来说，调用Spymemcached提供的API即可实现对服务端缓存的增、删、查、改操作，很是便利。处理业务线程操作的具体过程如下：
1. Spymemcached将每一个业务线程的操作都封装为一个Operation对象，然后放入内部的inputQueue队列中。
2. 在放入inputQueue的时候，调用selector.wakup()方法唤醒内部IO线程。
3. IO线程被唤醒后，会将inputQueue中所有的Operation拷贝到writeQueue中去，并且会立马对writeQueue中的Operation做IO写操作，而且会注册读事件，如果必要（还有可写的Operation或者写buf中还有数据）还会注册写事件。
4. 一旦MC服务端有数据返回，就会触发读事件，然后客户端就会返回缓存对象给业务线程。


下图描述了上述过程：

![](http://7xj5jf.com1.z0.glb.clouddn.com/Spymemcached.png)



