---
slug: java-zero-copy
title: JAVA Zero Copy
authors: [ledai]
tags: [JAVA Zero Copy]
date: 2020-03-04
---

# JAVA Zero Copy
<!-- truncate -->



##  传统I/O 流程

![Reactor模型](http://images2.pianshen.com/413/26/266343e285f9265a0afc0793230cdf9d.png)

流程: 用户程序 application 通过调用操作系统提供的 read method java 通过native  切换内核态 向硬件驱动获取数据 通过DMA (直接内存访问机制)  无需CPU 介入  然后内核空间将数据copy到 用户空间  然后用户执行自定义处理 然后application 调用操作系统的write 方法 将数据copy到内核空间  然后操作系统调用硬件驱动写入数据再将结果然后给用户空间



![Reactor模型](http://images1.pianshen.com/122/62/6269fe16792ca97021471d28a0b7e4b2.png)

操作系统提供了一种方式访问数据 sendfile 这种方式无需将内核态数据copy到 用户态 

### NIO中内存映射方式I/O

NIO中的`FileChannel.map()`方法其实就是采用了操作系统中的内存映射方式，**内存地址映射其实是操作系统将内存地址和磁盘文件做一个映射，读写这块内存，相当于直接对磁盘文件进行读写，但是实际上的读还是要经过OS读取到内存`PageCache`中，写过程也需要OS自动进行脏页置换到磁盘中。**



可以看到用户调用sendfile()之后，直接在内核空间进行数据的传输了。但是在内核空间中还是进行了一次将内核中数据拷贝到socket buffer中的操作，相当于第二幅图中的2直接指向Socket。但是相对于传统的方式，只减少了一次数据拷贝，和上下文切换，没有做到真正的零拷贝。所以Linux2.4实现了下面的方式来实现真正的零拷贝，java NIO中的零拷贝也是指使用操作系统提供的下面这种方式。

![Reactor模型](http://images1.pianshen.com/205/b6/b69623ea885638e4fac2c71ef708ccb5.png)



数据从磁盘复制到 内核空间，而后进行CPU copy，CPU copy不是复制数据，而是复制文件描述符（起始位置和长度）到socket buffer。直接将读取到内核的磁盘数据放置到protocol engine(协议解析引擎)中发送出去。所以最终只进行了两次数据的拷贝。

java 的零拷贝多在网络应用程序中使用。关键的api是java.nio.channel.FileChannel的transferTo()，transferFrom()方法。我们可以用这两个方法来把bytes直接从调用它的channel传输到另一个writable byte channel，中间不会使数据经过应用程序，也就是用户空间，以便提高数据转移的效率。



## Native 堆 与 head 堆的问题

https://www.zhihu.com/question/57374068

1.DirectByteBuffer 身是（Java）堆内的，它背后真正承载数据的buffer是在（Java）堆外——native memory中的。这是 malloc() 分配出来的内存，是用户态的。

 所以利用堆外内存仍需要将数据从用户空间 C head copy 到内核空间 所以如果使用DirectByteBuffer 是一次copy

如果使用HeapByteBuffer 是两次copy 首先要将jvm heap堆的数据 copy到 C heap 然后在 copy到 内核空间进行与外部设备i/o 交互

2.为何 在jvm heap 堆上内存需要copy 到 C heap ?

引用RednaxelaFX 的原文回答

`作者：RednaxelaFX
链接：https://www.zhihu.com/question/57374068/answer/152691891
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。


```
这里其实是在迁就OpenJDK里的HotSpot VM的一点实现细节。

HotSpot VM里的GC除了CMS之外都是要移动对象的，是所谓“compacting GC”。

如果要把一个Java里的 byte[] 对象的引用传给native代码，让native代码直接访问数组的内容的话，就必须要保证native代码在访问的时候这个 byte[] 对象不能被移动，也就是要被“pin”（钉）住。

可惜HotSpot VM出于一些取舍而决定不实现单个对象层面的object pinning，要pin的话就得暂时禁用GC——也就等于把整个Java堆都给pin住。HotSpot VM对JNI的Critical系API就是这样实现的。这用起来就不那么顺手。

所以 Oracle/Sun JDK / OpenJDK 的这个地方就用了点绕弯的做法。它假设把 HeapByteBuffer 背后的 byte[] 里的内容拷贝一次是一个时间开销可以接受的操作，同时假设真正的I/O可能是一个很慢的操作。

于是它就先把 HeapByteBuffer 背后的 byte[] 的内容拷贝到一个 DirectByteBuffer 背后的native memory去，这个拷贝会涉及 sun.misc.Unsafe.copyMemory() 的调用，背后是类似 memcpy() 的实现。这个操作本质上是会在整个拷贝过程中暂时不允许发生GC的，虽然实现方式跟JNI的Critical系API不太一样。（具体来说是 Unsafe.copyMemory() 是HotSpot VM的一个intrinsic方法，中间没有safepoint所以GC无法发生）。

然后数据被拷贝到native memory之后就好办了，就去做真正的I/O，把 DirectByteBuffer 背后的native memory地址传给真正做I/O的函数。这边就不需要再去访问Java对象去读写要做I/O的数据了。
```

大概的意思就是 如果让native 代码访问数组内容的话 必须要证这个数据的内存地址位置是固定的 但是由于 gc的缘故 不得不迁就 jvm 做一次额外的数据copy

