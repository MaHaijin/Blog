---
date: 2015-01-08 21:46
status: public
author: Haijin Ma
title: 【原创】Linux学习系列-轮询函数
categories: [Linux]
tags: [epoll,统一事件源]
---

理解这三个轮询函数差异的关键在于理解其轮询的文件描述符（socket也是文件）的数据结构。

##select轮询函数

函数定义：

 ``` C
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exeptfds, struct timeval *timeout);

// fd_set操作宏
void FD_SET(int fd, fd_set *fdset);
void FD_CLR(int fd, fd_set *fdset);
void FD_ISSET(int fd, fd_set *fdset);
void FD_ZERO(fd_set *fdset);
```

- nfds参数指定被监听的文件描述符总数；
- readfds、writefds、exeptfds参数分别指向可读、可写和异常事件的accept到的对应的文件描述符集合；fd_set中用一个整型数组的每一个元素的每一位（bit）来表示每一个文件描述符是否有相应的事件发生（0表示有；1表示没有），那么理论上fd_set就可以表示【数组长度*32】个文件描述符，而数组长度由FD_SETSIZE 来设置，一般是1024。在select之前，每次先调用FD_SET()设置accept到的文件描述符标记位为1，表示需要监听这个文件描述符需要监听可读、可写或者异常事件；当有可读、可写和异常事件发生时，在select调用过程中，内核会修改相应的这三个文件描述符集合中的没有事件发生的文件描述符标记位为0；在select之后，应用程序就可以轮询accept到的所有文件描述符，并通过FD_ISSET来判断该文件描述符是否有事件发生（fd_set中对应的文件描述符标记位是1表示有事件发生）。
- timeout就是轮询时间片，表示多久轮询一次。

##这种函数设计实现，有三个缺陷：##
1. 监听的事件类型有限，只有3种；
2. readfds、writefds和exeptfds有两重角色，第一层是当做要监听的文件描述符集合来传给内核监听；第二层是当做发生事件的文件描述符结果集合。因此每次轮询处理完事件，需要重新设置需要监听的文件描述符；
3. 事件和文件描述符没有绑定，因此要处理事件需要轮询所有的已accept到的文件描述符。

##poll轮询函数
函数定义：
 ``` C
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd
{
    int fd;  //文件描述符
    short events;  //注册的事件
    short revents  //实际发生的事件，由内核填充
}
```

- fds是需要监听的文件描述符集合，它是一个pollfd结构体的数组。
- nfds是监听的文件描述符的数量，也就是fds数组的长度。

说明：与select相比这种函数设计清晰简洁，将文件描述符和事件通过pollfd结构体来进行了绑定，并且区分了注册的时间和实际发生的事件。但是其和select一样，最终轮询的结果都是所有已经注册的文件描述符的集合。

##epoll轮询函数
函数定义:
 ``` C
/* 创建内核事件表，返回的就是事件表的文件描述符 */
int epoll_create(int size); 
/* 往内核事件表注册、修改、删除指定fd上的事件 */
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); 
/* 轮询检测事件 */
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

- epoll和select、poll有明显不同，其提供了一套函数。它会在内核开辟一个事件表用于存储注册的事件。
- epoll_ctl中的epfd就是epoll_create的返回值，表示内核事件表的文件描述符；op就是操作，包括注册、删除、修改；fd表示要操作的文件描述符；event表示要注册到fd上的事件，需要特别之处epoll_event结构实现了事件和文件描述符的绑定。
- epoll_wait是轮询函数，如果检测到事件，就将所有就绪的事件从内核事件表中的拷贝到第二个参数指向的数组events中。这和select、poll有明显区别，这提高了应用程序轮询就绪事件的效率。
- epoll还有一个特殊的地方在于，其存在两种模式：LT（电平触发）和ET（边沿触发）。LT是默认工作模式，而当往内核事件表注册一个文件描述符的EPOLLET事件时，epoll将以ET模式来操作文件描述符。ET是一种高效模式。LT模式下，当epoll_wait检测到事件时，应用程序可以不用立即处理，下次epoll_wait时还会再次通告应用程序；ET模式下，应用程序必须立即处理事件，因此避免了epoll_wait重复触发事件的次数，因此较高效。

##统一事件源
我们这里讲的统一事件源中的事件源是指IO事件和信号。信号原理可以参见《Linux学习系列-信号》。通常我们通过轮询函数来处理IO事件，既然要统一，那么自然也要使用轮询函数来处理信号。典型的处理方案是：信号发生时，信号处理函数一般通过管道来通知程序主循环信号值，那么主循环中的轮询函数就可以通过轮询函数来监听管道上的IO事件即可。这样就实现了IO事件和信号的统一处理。