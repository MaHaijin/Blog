---
date: 2015-01-20 21:46
status: public
author: Haijin Ma
title: '【原创】Java NIO写事件处理技巧'
categories: [并发编程]
tags: [NIO,Netty,memcached,Grizzly]
---

##问题背景
OP_WRITE事件是在Socket发送缓冲区中的可用字节数大于或等于其低水位标记SO_SNDLOWAT时发生。正常情况下，都是可写的，因此一般不注册写事件。所以一般代码如下：
```java
while (bb.hasRemaining()) {
     int len = socketChannel.write(bb);
     if (len < 0) {
      throw new EOFException();
    }
}
```
这样在大部分情况都没问题，但是高并发，并且在网络环境很差的情况下，发送缓冲区可能会满，导致无限循环，这样最终会导致CPU利用率100%。下面就看看一些基于NIO的框架，是如何处理这个问题的。

##Spymemcached的处理方式：
```java
  private void handleWrites(SelectionKey sk, MemcachedNode qa)
    throws IOException {
    // 填充写缓冲区
    qa.fillWriteBuffer(shouldOptimize);
    boolean canWriteMore = qa.getBytesRemainingToWrite() > 0;
    while (canWriteMore) {
      int wrote = qa.writeSome();
      qa.fillWriteBuffer(shouldOptimize);
      // 如果wrote等于零，表示没有写出数据，那么不再尝试写，等待下次线程外层循环注册write事件
      canWriteMore = wrote > 0 && qa.getBytesRemainingToWrite() > 0;
    }

public final int writeSome() throws IOException {
    int wrote = channel.write(wbuf);
    // 写入多少个字节，toWrite就减去对应的数量
    toWrite -= wrote;
    return wrote;
}

public final int getSelectionOps() {
    int rv = 0;
    if (getChannel().isConnected()) {
      if (hasReadOp()) {
        rv |= SelectionKey.OP_READ;
      }
      // 如果toWrite大于0，说明由于某种异常原因上次写入还未完成；hasWriteOp()用于判断写队列是否还有元素。这两种情况下，需要注册写事件。本文讨论的是toWrite>0的情况。
      if (toWrite > 0 || hasWriteOp()) {
        rv |= SelectionKey.OP_WRITE;
      }
    } else {
      rv = SelectionKey.OP_CONNECT;
    }
    return rv;
}

```
**说明：Spymemcached是单线程的，因此就是绝对不能阻塞，所以当发现不可写的时候，不能阻塞住线程，而是立即返回，等待下次主线程循环来注册事件。**

##Netty的处理方式：
```java
    protected void write0(AbstractNioChannel<?> channel) {
        boolean open = true;
        boolean addOpWrite = false;
        boolean removeOpWrite = false;
        boolean iothread = isIoThread(channel);

        long writtenBytes = 0;

        final SocketSendBufferPool sendBufferPool = this.sendBufferPool;
        final WritableByteChannel ch = channel.channel;
        final Queue<MessageEvent> writeBuffer = channel.writeBufferQueue;
        final int writeSpinCount = channel.getConfig().getWriteSpinCount();
        List<Throwable> causes = null;

        synchronized (channel.writeLock) {
            channel.inWriteNowLoop = true;
            for (;;) {
                MessageEvent evt = channel.currentWriteEvent;
                SendBuffer buf = null;
                ChannelFuture future = null;
                try {
                    if (evt == null) {
                        if ((channel.currentWriteEvent = evt = writeBuffer.poll()) == null) {
                            // 如果无数据可写，则需要删除可写事件的注册
                            removeOpWrite = true;
                            channel.writeSuspended = false;
                            break;
                        }
                        future = evt.getFuture();

                        channel.currentWriteBuffer = buf = sendBufferPool.acquire(evt.getMessage());
                    } else {
                        future = evt.getFuture();
                        buf = channel.currentWriteBuffer;
                    }

                    long localWrittenBytes = 0;
                    // 通过writeSpinCount来控制尝试写的次数，如果最终还是无法写入，就注册写事件
                    for (int i = writeSpinCount; i > 0; i --) {
                        // 写数据
                        localWrittenBytes = buf.transferTo(ch);
                        //  如果写入数据不等于零，表明写入成功，跳出循环
                        if (localWrittenBytes != 0) {
                            writtenBytes += localWrittenBytes;
                            break;
                        }
                        // 如果buf的数据都写完了，则跳出循环
                        if (buf.finished()) {
                            break;
                        }
                    }

                    if (buf.finished()) {
                        // Successful write - proceed to the next message.
                        buf.release();
                        channel.currentWriteEvent = null;
                        channel.currentWriteBuffer = null;
                        // Mark the event object for garbage collection.
                        //noinspection UnusedAssignment
                        evt = null;
                        buf = null;
                        future.setSuccess();
                    } else {
                        // Not written fully - perhaps the kernel buffer is full.
                        addOpWrite = true;
                        channel.writeSuspended = true;

                        if (writtenBytes > 0) {
                            // Notify progress listeners if necessary.
                            future.setProgress(
                                    localWrittenBytes,
                                    buf.writtenBytes(), buf.totalBytes());
                        }
                        break;
                    }
                }
            }
            channel.inWriteNowLoop = false;

            if (open) {
                if (addOpWrite) {
                    // 注册写事件
                    setOpWrite(channel);
                } else if (removeOpWrite) {
                   // 删除写事件
                    clearOpWrite(channel);
                }
            }
        }
    }
```
**说明：Netty是多线程的，因此其可以通过阻塞线程做一定的等待，等待通道可写。Netty等待是通过spinCount等待指定的循环次数。**

##Grizzly（诞生子Glass Fish项目）的处理方式：
```java
    public static long flushChannel(SocketChannel socketChannel, ByteBuffer bb, long writeTimeout)
            throws IOException {
        SelectionKey key = null;
        Selector writeSelector = null;
        int attempts = 0;
        int bytesProduced = 0;
        try {
            while (bb.hasRemaining()) {
                int len = socketChannel.write(bb);
                // 类似Netty的spinCount
                attempts++;
                if (len < 0) {
                    throw new EOFException();
                }
                bytesProduced += len;
                if (len == 0) {
                    if (writeSelector == null) {
                       // 获取一个新的selector
                        writeSelector = SelectorFactory.getSelector();
                        if (writeSelector == null) {
                            // Continue using the main one
                            continue;
                        }
                    }
                    // 在新selector上注册写事件，而不是在主selector上注册
                    key = socketChannel.register(writeSelector, key.OP_WRITE);
                    // 利用writeSelector.select()来阻塞当前线程，等待可写事件发生，总共等待可写事件的时长是3*writeTimeout
                    if (writeSelector.select(writeTimeout) == 0) {
                        if (attempts > 2)
                            throw new IOException("Client disconnected");
                    } else {
                        attempts--;
                    }
                } else {
                    attempts = 0;
                }
            }
        } 
        return bytesProduced;
    }
```
**说明：Grizzly是多线程的，因此其可以做合适的阻塞等待。其没有再主selector上注册写事件，而是在重新构造的selector上注册写事件，并且通过select()来阻塞一定的时间来等待可写。**
为什么要这么做呢？Grizzly的作者对此的回应如下：
1. 使用临时的Selector的目的是减少线程间的切换。当前的Selector一般用来处理OP_ACCEPT，和OP_READ的操作。使用临时的Selector可减轻主Selector的负担;而在注册的时候则需要进行线程切换，会引起不必要的系统调用。这种方式避免了线程之间的频繁切换，有利于系统的性能提高。
2. 虽然writeSelector.select(writeTimeout)做了阻塞操作，但是这种情况只是少数极端的环境下才会发生。> 大多数的客户端是不会频繁出现这种现象的，因此在同一时刻被阻塞的线程不会很多。
3. 利用这个阻塞操作来判断异常中断的客户连接。
4.  经过压力实验证明这种实现的性能是非常好的。
