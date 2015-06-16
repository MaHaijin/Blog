---
date: 2015-04-17 11:25
status: public
author: Haijin Ma
title: 【原创】基于Netty的“请求-响应”同步通信机制实现
categories: [并发编程]
tags: [Netty,Future]
---

## 需求描述
实现基于Netty的“请求-响应”同步通信机制。
## 设计思路
Netty提供了异步IO和同步IO的统一实现，但是我们的需求其实和IO的同步异步并无关系。我们的关键是要实现请求-响应这种典型的一问一答交互方式。要实现这个需求，需要解决两个问题：
1. 请求和响应的正确匹配。
       当服务端返回响应结果的时候，怎么和客户端的请求正确匹配起来呢？解决方式：通过客户端唯一的RequestId，服务端返回的响应中需要包含该RequestId，这样客户端就可以通过RequestId来正确匹配请求响应。

2. 请求线程和响应线程的通信。
       因为请求线程会在发出请求后，同步等待飞鸽服务端的返回。因此，就需要解决，Netty在接受到响应之后，怎么通知请求线程结果。

       解决方式是利用Java中的CountDownLatch类来实现同步Future。具体过程是：客户端发送请求后将<请求ID，Future>的键值对保存到一个全局的Map中，这时候用Future等待结果，挂住请求线程；当Netty收到服务端的响应后，响应线程根据请求ID寸全局Map中取出Future，然后设置响应结果到Future中。这个时候利用CountDownLatch的通知机制，通知请求线程。请求线程从Future中拿到响应结果，然后做业务处理。

       通过对以上两个问题的解决，又引出了一个新的问题：全局Map在并发较大，网络环境较差的情况下，有可能爆满。目前SDK通过一个定时任务，来定时清理该全局Map中超时的请求。

同步Future的实现如下：
 ```Java
public class SyncFuture<T> implements Future<T> {
    // 因为请求和响应是一一对应的，因此初始化CountDownLatch值为1。
    private CountDownLatch latch = new CountDownLatch(1);
    // 需要响应线程设置的响应结果
    private T response;
    // Futrue的请求时间，用于计算Future是否超时
    private long beginTime = System.currentTimeMillis();
    public SyncFuture() {
    }
    @Override
    public boolean cancel(boolean mayInterruptIfRunning) {
        return false;
    }
    @Override
    public boolean isCancelled() {
        return false;
    }
    @Override
    public boolean isDone() {
        if (response != null) {
            return true;
        }
        return false;
    }
    // 获取响应结果，直到有结果才返回。
    @Override
    public T get() throws InterruptedException {
        latch.await();
        return this.response;
    }
    // 获取响应结果，直到有结果或者超过指定时间就返回。
    @Override
    public T get(long timeout, TimeUnit unit) throws InterruptedException {
        if (latch.await(timeout, unit)) {
            return this.response;
        }
        return null;
    }
    // 用于设置响应结果，并且做countDown操作，通知请求线程
    public void setResponse(T response) {
        this.response = response;
        latch.countDown();
    }
    public long getBeginTime() {
        return beginTime;
    }
}
```