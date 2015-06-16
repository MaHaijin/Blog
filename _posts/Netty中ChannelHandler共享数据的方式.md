---
date: 2015-05-28 22:50
status: public
author: Haijin Ma
title: 【原创】Netty中ChannelHandler共享数据的方式
categories: [并发编程]
tags: [Netty]
---
##（一）成员变量
```JAVA
public class DataServerHandler extends SimpleChannelHandler {
    // 成员变量
    private boolean loggedIn;
 
    @Override
    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) {
        ...
    }
}
```
- 如果只是想在当前连接内共享数据，那么需要针对不同的Channel创建不同的ChannelHandler实例，避免共享范围扩大至所有连接。

```JAVA
// 所有Channel共用一个ChannelHandler实例 
public class DataServerPipelineFactory implements ChannelPipelineFactory {
 
     private static final DataServerHandler SHARED = new DataServerHandler();
 
     public ChannelPipeline getPipeline() {
         return Channels.pipeline(SHARED);
     }
 }
// 每一个Channel创建一个ChannelHandler实例
bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
    public ChannelPipeline getPipeline() {
        ChannelPipeline pipeline = pipeline();      
        pipeline.addLast("handler", new TestHandler());
        return pipeline;
    }
});
```

##（二）ChannelHandlerContext
- ChannelHandlerContext是在Pipline注册ChannelHandler的时候和其绑定的，因此一个被多次注册（无论是否是同一个Pipline）的ChannelHandler会同时拥有多个ChannelHandlerContext。
- 对于ChannelHandlerContext一个常见的误解是以为其可以在同一个Pipline的各个Handler之间传递数据。如果真要这样做，应该使用MessageEvent或者ChannelLocal。

```JAVA
@Sharable
public class DataServerHandler extends SimpleChannelHandler {
 
    @Override
    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) {
        Channel ch = e.getChannel();
        Object o = e.getMessage();
        if (o instanceof LoginMessage) {
            authenticate((LoginMessage) o);
            // 设置共享数据
            ctx.setAttachment(true);
        }
    }
}
```
- @Sharable注解只是起到一个标示的作用，说明一个该ChannelHandler实例可以被多次注册到一个或者多个Pipline中，而且不会导致不同的Channel共享数据出现竞争条件。
- 基于ChannelHandlerContext同ChannelHandler的绑定时机和机制，很容易理解在本例中是怎么保证@Sharable的。

##（三）Channel
- 该方法类似ChannelLocal，只是共享数据是直接存储在Channel实例中的。

```JAVA
@Sharable
public class TestHandler extends SimpleChannelHandler {
    @Override
    public void messageReceived(ChannelHandlerContext ctx, MessageEvent event) throws Exception {
        // 通过任意一种方式获取到当前Channel，然后设置共享数据
        ctx.getChannel().setAttachment(true);
        event.getChannel().setAttachment(true);
    }
}
```

- 很明显，本例中的ChannelLocal也是符合@Sharable要求的。

##（四）ChannelLocal
- 如果想在当前连接的其他ChannelHandler或者外部Handler中共享数据，则可以使用ChannelLocal。其设计思想类似于JDK中的ThreadLocal，其内部是一个Channel做Key，共享数据做Value的结构。

```JAVA
public final class DataServerState {
 
    public static final ChannelLocal<Boolean> loggedIn = new ChannelLocal<>() {
        protected Boolean initialValue(Channel channel) {
            return false;
        }
    }
}
 
@Sharable
public class DataServerHandler extends SimpleChannelHandler {
 
    @Override
    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) {
        Channel ch = e.getChannel();
        Object o = e.getMessage();
        if (o instanceof LoginMessage) {
            authenticate((LoginMessage) o);
            // 设置共享数据
            DataServerState.loggedIn.set(ch, true);
        }
    }
}
```

- 很明显，本例中的ChannelLocal也是符合@Sharable要求的。