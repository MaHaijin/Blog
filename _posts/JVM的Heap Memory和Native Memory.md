---
date: 2015-02-27 10:34
status: public
author: Haijin Ma
title: '【原创】JVM的Heap Memory和Native Memory'
categories: [JVM]
tags: [Native Memory,Java Heap]
---

JVM管理的内存可以总体划分为两部分：Heap Memory和Native Memory。前者我们比较熟悉，是供Java应用程序使用的；后者也称为C-Heap，是供JVM自身进程使用的。Heap Memory及其内部各组成的大小可以通过JVM的一系列命令行参数来控制，在此不赘述。Native Memory没有相应的参数来控制大小，其大小依赖于操作系统进程的最大值（对于32位系统就是3~4G，各种系统的实现并不一样），以及生成的Java字节码大小、创建的线程数量、维持java对象的状态信息大小（用于GC）以及一些第三方的包，比如JDBC驱动使用的native内存。

##Native Memory里存些什么？
1. 管理java heap的状态数据（用于GC）;
2. JNI调用，也就是Native Stack;
3. JIT（即使编译器）编译时使用Native Memory，并且JIT的输入（Java字节码）和输出（可执行代码）也都是保存在Native Memory；
4. NIO direct buffer。对于IBM JVM和Hotspot，都可以通过-XX:MaxDirectMemorySize来设置nio直接缓冲区的最大值。默认是64M。超过这个时，会按照32M自动增大。
5. 对于IBM的JVM某些版本实现，类加载器和类信息都是保存在Native Memory中的。

##DirectBuffer的好处
DirectBuffer访问更快，避免了从HeapBuffer还需要从java堆拷贝到本地堆，操作系统直接访问的是DirectBuffer。DirectBuffer对象的数据实际是保存在native heap中，但是引用保存在HeapBuffer中。
另外，DirectBuffer的引用是直接分配在堆得Old区的，因此其回收时机是在FullGC时。因此，需要避免频繁的分配DirectBuffer，这样很容易导致Native Memory溢出。

![](http://7xj5jf.com1.z0.glb.clouddn.com/DirectBuffer.png)

##为什么会内存溢出？
简单理解java process memory = java heap + native memory。因此内存溢出时，首先要区分是堆内存溢出还是本地内存溢出。Native Memory本质上就是因为耗尽了进程地址空间。对于HotSpot JVM来书，不断的分配直接内存，会导致如下错误信息：Allocated 1953546760 bytes of native memory before running out

参考资料：
http://www.ibm.com/developerworks/library/j-nativememory-linux/index.html
http://www.techpaste.com/2012/07/steps-debugdiagnose-memory-memory-leaks-jvm/
https://sourcevirtues.wordpress.com/2013/01/14/java-heap-space-and-native-heap-problems/
http://www.theotherian.com/2013/08/understanding-javas-native-heap-or-c-heap.html
http://www.ibm.com/developerworks/library/l-kernel-memory-access/
http://www.ibm.com/developerworks/library/j-zerocopy/
http://en.wikipedia.org/wiki/Direct_memory_access