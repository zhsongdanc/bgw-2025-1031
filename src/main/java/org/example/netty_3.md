***\*1.关于ByteBuf的问题整理\****

***\*2.ByteBuf的分类\****

***\*3.ByteBuf分类的补充说明\****

***\*4.ByteBuf的主要内容分三大方面\****

***\*5.内存分配器ByteBufAllocator\****

***\*6.ByteBufAllocator的两大子类\****

***\*7.PoolArena分配内存的流程\****

***\*8.Netty的内存规格\****

***\*9.缓存数据结构\****

***\*10.命中缓存的分配流程\****

***\*11.Netty里有关内存分配的重要概念\****

***\*12.Page级别的内存分配\****

***\*13.SubPage级别的内存分配\****

***\*14.ByteBuf的回收\****

***\*15.读数据入口\****

***\*16.拆包原理\****

***\*17.解码器抽象的解码过程总结\****

***\*18.如何把对象变成字节流写到unsafe底层\****

***\*19.Netty的两大性能优化工具\****

***\*20.FastThreadLocal的总结\****

***\*21.Recycler的设计理念\****

***\*22.Recycler的总结\****

***\*
\****

***\*1.关于ByteBuf的问题整理\****

***\*问题一：Netty的内存类别有哪些？\****

答：ByteBuf可以按三个维度来进行分类：一个是堆内和堆外，一个是Unsafe和非Unsafe，一个是Pooled和非Pooled。堆内是基于byte字节数组内存进行分配，堆外是基于JDK的DirectByteBuffer内存进行分配。Unsafe是通过JDK的一个unsafe对象基于物理内存地址进行数据读写，非Unsafe是直接调用JDK的API进行数据读写。Pooled是预先分配的一整块内存，分配时直接用一定的算法从这整块内存里取出一块连续内存。UnPooled是每次分配内存都直接申请内存。

**
**

***\*问题二：如何减少多线程内存分配之间的竞争？如何确保多线程对于同一内存分配不产生冲突？\****

答：一个内存分配器里维护着一个PoolArena数组，所有的内存分配都在PoolArena上进行。通过一个PoolThreadCache对象将线程和PoolArena进行一一绑定(利用ThreadLocal原理)。默认一个线程对应一个PoolArena，这样就能做到多线程内存分配相互不受影响。

**
**

***\**\*问题三：不同大小的内存是如何进行分配的？\*\**\***

答：对于Page级别的内存分配与释放是直接通过完全二叉树的标记来寻找某一个连续内存的。对于Page级别以下的内存分配与释放，首先是找到一个Page，然后把此Page按照SubPage大小进行划分，最后通过位图的方式来进行内存分配与释放。不管是Page级别的内存还是SubPage级别的内存，当内存被释放掉时有可能会被加入到不同级别的一个缓存队列供下一次分配使用。

**
**

***\*2.ByteBuf的分类\****

***\*(1)Pooled和Unpooled\****

Pooled和Unpooled就是池化和非池化的分类。Pooled的内存分配每一次都是从预先分配好的一块内存里去取一段连续内存来封装成一个ByteBuf对象给应用程序。Unpooled的内存分配每一次都是直接调用系统API去向操作系统申请一块内存。所以两者最大区别是：一个是从预先分配好的内存里分配，一个是直接去分配。

**
**

***\*(2)Unsafe和非Unsafe\****

Unsafe和非Unsafe不需要我们操心，Netty会根据系统是否有unsafe对象自行选择。JDK里有个Unsafe对象，它可以直接拿到对象的内存地址，然后可以基于这个内存地址进行读写操作。Unsafe类型的ByteBuf可以拿到ByteBuf对象在JVM里的具体内存地址，然后直接通过JDK的Unsafe进行读写。非Unsafe类型的ByteBuf则是不会依赖到JDK里的Unsafe对象。

**
**

***\*(3)Heap和Direct\****

Heap和Direct就是堆内和堆外的分类。Heap分配出来的内存会自动受到GC的管理，不需要手动释放。Direct分配出来的内存则不受JVM控制，不参与GC垃圾回收的过程，需要手动释放以免内存泄露。UnpooledHeapByteBuf底层会依赖于一个字节数组byte[]进行所有内存相关的操作，UnpooledDirectByteBuf底层则依赖于一个JDK堆外内存对象DirectByteBuffer进行所有内存相关的操作。

**
**

***\*3.ByteBuf分类的补充说明\****

***\*(1)堆内存HeapByteBuf\****

优点是内存的分配和回收速度快，可以被JVM自动回收。缺点是如果进行Socket的IO读写，需要额外做一次内存复制。也就是将堆内存对应的缓冲区复制到内核Channel中，性能下降。

**
**

***\*(2)直接内存DIrectByteBuf\****

在堆外进行内存分配，相比于堆内存，它的分配和回收速度还会慢一些。但是如果进行Scoket的IO读写，由于少了一次内存复制，所以速度比堆内存快。所以最佳实践应该是：IO通信线程的读写缓冲区使用DirectByteBuf，后端业务消息的编解码模块使用HeapByteBuf。

**
**

***\*(3)Pooled和Unpooled\****

Pooled的ByteBuf可以重用ByteBuf对象，它自己维护了一个内存池，可以循环利用已创建的ByteBuf，从而提升了内存的使用效率，降低了由于高负载导致的频繁GC。尽管推荐使用基于Pooled的ByteBuf，但是内存池的管理和维护更加复杂，使用起来需要更加谨慎。

**
**

***\*(4)池化和非池化的HeapByteBuf\****

UnpooledHeapByteBuf是基于堆内存进行内存分配的字节缓冲区，每次IO读写都会创建一个新的UnpooledHeapByteBuf。频繁进行大块内存的分配和回收会对性能造成影响，但相比于堆外内存的申请和释放，成本还是低些。



UnpooledHeapByteBuf的实现原理比PooledHeapByteBuf简单，不容易出现内存管理方面的问题，满足性能下推荐UnpooledHeapByteBuf。

**
**

***\*4.ByteBuf的主要内容分三大方面\****

一.内存与内存分配器的抽象

二.不同规格大小和不同类别的内存的分配策略

三.内存的回收过程

**
**

***\*5.内存分配器ByteBufAllocator\****

***\*(1)ByteBufAllocator的核心功能\****

所有类型的ByteBuf最终都是通过Netty里的内存分配器分配出来的，Netty里的内存分配器都有一个最顶层的抽象ByteBufAllocator，用于负责分配所有类型的内存。



ByteBufAllocator的核心功能如下：

一.buffer()分配一块内存或者说分配一个字节缓冲区，由子类具体实现决定是Heap还是Direct。

二.ioBuffer()分配一块DirectByteBuffer的内存。

三.heapBuffer()在堆上进行内存分配。

四.directBuffer()在堆外进行内存分配。

**
**

***\*(2)AbstractByteBufAllocator\****

AbstractByteBufAllocator实现了ByteBufAllocator的大部分功能，并最终暴露出两个基本的API，也就是抽象方法newDirectBuffer()和newHeapBuffer()。这两个抽象方法会由PooledByteBufAllocator和UnpooledByteBufAllocator来实现。

**
**

***\*6.ByteBufAllocator的两大子类\****

***\*(1)UnpooledByteBufAllocator介绍\****

对于UnpooledHeadByteBuf的创建，会直接new一个字节数组，并且读写指针初始化为0。对于UnpooledDirectByteBuf的创建，会直接new一个DirectByteBuffer对象。注意：其中unsafe是Netty自行判断的，如果系统支持获取unsafe对象就使用unsafe对象。



对比UnpooledUnsafeHeadByteBuf和UnpooledHeadByteBuf的getByte()方法可知，unsafe和非unsafe的区别如下：unsafe最终会通过对象的内存地址 + 偏移量的方式去拿到对应的数据，非unsafe最终会通过数组 + 下标或者JDK底层的ByteBuffer的API去拿到对应的数据。一般情况下，通过unsafe对象去取数据要比非unsafe要快一些，因为unsafe对象是直接通过内存地址操作的。

**
**

***\*(2)PooledByteBufAllocator介绍\****

PooledByteBufAllocator的newHeapBuffer()方法和newDirectBuffer()方法，都会首先通过threadCache获取一个PoolThreadCache对象，然后再从该对象获取一个heapArena对象或directArena对象，最后通过heapArena对象或directArena对象的allocate()方法去分配内存。具体步骤如下：

**
**

***\**\*步骤一：拿到线程局部缓存PoolThreadCache。\*\**\***因为newHeapBuffer()和newDirectBuffer()可能会被多线程同时调用，所以threadCache.get()拿到的是当前线程的cache，一个PoolThreadLocalCache对象。PoolThreadLocalCache继承自FastThreadLocal，FastThreadLocal可以当作JDK的ThreadLocal，只不过比ThreadLocal更快。每个线程都有唯一的PoolThreadCache，PoolThreadCache里维护两大内存：一个是堆内存heapArena，一个是堆外内存directArena。

**
**

***\**\*步骤二：在线程局部缓存的Arena上进行内存分配。\*\**\***Arena可以翻译成竞技场的意思。创建PooledByteBufAllocator内存分配器时，会创建两种类型的PoolArena数组：heapArenas和directArenas。这两个数组的大小默认都是两倍CPU核数，因为这样就和创建的NIO线程数一样了。这样每个线程都可以有一个独立的PoolArena。PoolThreadLocalCache的initialValue()方法中，会从PoolArena数组中获取一个PoolArena与当前线程进行绑定。对于PoolArena数组里的每个PoolArena，在分配内存时是不用加锁的。

**
**

***\*(3)PooledByteBufAllocator的结构\****

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/DXGTicJyJ8QCyEBV5swDYspiaCwmeqLdqWPP4mibdXxtzUnRjIjynWmgicM2Sgb8WIeicLIoicjCpgvb1wrlibOqgWyfQ/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

***\*(4)PooledByteBufAllocator如何创建一个ByteBuf总结\****

每个线程调用PoolThreadLocalCache的get()方法时，都会拿到一个PoolThreadCache对象。然后通过PoolThreadCache对象可以拿到该线程对应的一个PoolArena对象。



这个PoolThreadCache对象的作用就是通过FastThreadLocal的方式，把PooledByteBufAllocator内存分配器的PoolArena数组中的一个PoolArena对象放入它的成员变量里。



比如第1个线程就会拿到内存分配器的heapArenas数组中的第1个PoolArena对象，第n个线程就会拿到内存分配器的heapArenas数组中的第n个PoolArena对象，从而将一个线程和一个PoolArena进行绑定了。



PoolThreadCache除了可以直接在这个PoolArena内存上进行分配外，还可以在它维护的一些ByteBuffer或者byte[]缓存列表上进行分配。比如我们通过PooledByteBufAllocator内存分配器创建了一个1024字节的ByteBuf，该ByteBuf被用完并释放后，可能还需要在其他地方继续分配1024字节大小的内存。这时其实不需要重新在PoolArena上进行内存分配，而可以直接从PoolThreadCache里维护的ByteBuffer或byte[]缓存列表里拿出来返回即可。



PooledByteBufAllocator内存分配器里维护了三个值：tinyCacheSize、smallCacheSize、normalCacheSize，tinyCacheSize表明tiny类型的ByteBuf最多可以缓存512个，smallCacheSize表明small类型的ByteBuf最多可以缓存256个，normalCacheSize表明normal类型的ByteBuf最多可以缓存64个。



在创建PoolThreadCache对象时，会把这3个值传递进去。然后用于创建：

tinySubPageHeapCaches、

smallSubPageHeapCaches、

normalHeapCaches、

tinySubPageDirectCaches、

smallSubPageDirectCaches、

normalDirectCaches。

**
**

***\*7.PoolArena分配内存的流程\****

PooledByteBufAllocator内存分配器在使用其方法newHeapBuffer()和newDirectBuffer()分配内存时，会分别执行代码heapArena.allocate()和directArena.allocate()，其实就是调用PoolArena的allocate()方法。在PoolArena的allocate()方法里，会通过其抽象方法newByteBuf()创建一个PooledByteBuf对象，而具体的newByteBuf()方法会由PoolArena的子类DirectArena和HeapArena来实现。



PoolArena的allocate()方法分配内存的大体逻辑如下：首先通过由PoolArena子类实现的newByteBuf()方法获取一个PooledByteBuf对象，接着通过PoolArena的allocate()方法在线程私有的PoolThreadCache上分配内存，这个分配过程其实就是拿一块内存，然后初始化PooledByteBuf对象里的内存地址值。



PoolArena.allocate()方法分配内存的逻辑如下：

**
**

***\**\*步骤一：\*\**\***首先PoolArena.newByteBuf()方法会从RECYCLER对象池中，尝试获取一个PooledByteBuf对象并进行复用，若获取不到就创建一个PooledByteBuf。以DirectArena的newByteBuf()方法为例，它会通过RECYCLER.get()拿到一个PooledByteBuf。RECYCLER是一个带有回收特性的对象池，RECYCLER.get()的含义是：若对象池里有一个PooledByteBuf就拿出一个，没有就创建一个。拿到一个PooledByteBuf之后，由于可能是从回收站里拿出来的，所以要调用buf.reuse()进行复用，然后才是返回。

**
**

***\**\*步骤二：\*\**\***接着PoolArena.allocate()方法会在PoolThreadCache缓存上尝试进行内存分配。如果有一个ByteBuf对象之前已使用过并且被释放掉了，而这次需要分配的内存是差不多规格大小的一个ByteBuf，那么就可以直接在该规格大小对应的一个缓存列表里获取这个ByteBuf缓存，然后进行分配。

**
**

***\**\*步骤三：\*\**\***如果没有命中PoolThreadCache的缓存，那么就进行实际的内存分配。

**
**

***\*8.Netty的内存规格\****

***\*(1)4种内存规格\****

一.tiny：表示从0到512字节之间的内存大小

二.small：表示从512字节到8K范围的内存大小

三.normal：表示从8K到16M范围的内存大小

四.huge：表示大于16M的内存大小

**
**

***\*(2)内存申请单位\****

Netty里所有的内存申请都是以Chunk为单位向操作系统申请的，后续所有的内存分配都是在这个Chunk里进行对应的操作。比如要分配1M的内存，那么首先要申请一个16M的Chunk，然后在这个16M的Chunk里取出一段1M的连续内存放入到Netty的ByteBuf里。注意：一个Chunk的大小为16M，一个Page的大小为8K，一个SubPage的大小是0～8K，一个Chunk可以分成2048个Page。

**
**

***\*9.缓存数据结构\****

***\*(1)MemoryRegionCache的组成\****

Netty中与缓存相关的数据结构叫MemoryRegionCache，这是内存相关的一个缓存。MemoryRegionCache由三部分组成：queue、sizeClass、size。

**
**

***\*一.queue\****

queue是一个队列，里面的每个元素都是MemoryRegionCache内部类Entry的一个实体，每一个Entry实体里都有一个chunk和一个handle。Netty里所有的内存都是以Chunk为单位进行分配的，而每一个handle都指向唯一一段连续的内存。所以一个chunk + 一个指向连续内存的handle，就能确定这块Entry的内存大小和内存位置，然后所有这些Entry组合起来就变成一个缓存的链。

**
**

***\*二.sizeClass\****

sizeClass是Netty里的内存规格，其中有三种类型的内存规则。一种是tiny(0~512B)，一种是small(512B~8K)，一种是normal(8K~16M)。由于huge是直接使用非缓存的内存分配，所以不在该sizeClass范围内。

**
**

***\*三.size\****

一个MemoryRegionCache所缓存的一个ByteBuf的大小是固定的。如果MemoryRegionCache里缓存了1K的ByteBuf，那么queue里所有的元素都是1K的ByteBuf。也就是说，同一个MemoryRegionCache它的queue里的所有元素都是固定大小的。这些固定大小分别有：tiny类型规则的是16B的整数倍直到498B，small类型规则的有512B、1K、2K、4K，normal类型规定的有8K、16K、32K。所以对于32K以上是不缓存的。

**
**

***\*(2)MemoryRegionCache的类型\****

Netty里所有规格的MemoryRegionCache如下图示，下面的每个节点就相当于一个MemoryRegionCache的数据结构。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/DXGTicJyJ8QCyEBV5swDYspiaCwmeqLdqWFUNbABUcvlZYeUH9LokqNwcy1RzPnbAuXTNoBVibk54yRk70dUqwevw/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

其中tiny类型的内存规格有32种，也就是32个节点，分别是16B、32B、48B、......、496B。每个节点都是一个MemoryRegionCache，每个MemoryRegionCache里都有一个queue。假设要分配一个16B的ByteBuf：首先会定位到small类型的内存规格里的第二个节点，然后从该节点维护的queue里取出一个Entry元素。通过该Entry元素可以拿到它属于哪一个chunk以及哪一个handle，从而进行内存划分。



small类型的内存规格有4种，也就是4个节点，分别是512B、1K、2K、4K。每个节点都是一个MemoryRegionCache，每个MemoryRegionCache里都有一个queue。假设要分配一个1K的ByteBuf：首先会定位到small类型的内存规格里的第二个节点，然后从该节点维护的queue里取出一个Entry元素。这样就可以基于这个Entry元素分配出1K内存的ByteBuf，不需要再去Chunk上找一段临时内存了。



normal类型的内存规格有3种，也就是3个节点，分别是8K、16K、32K，关于Normal大小的ByteBuf的内存分配也是同样道理。

**
**

***\*(3)MemoryRegionCache的源码\****

每个线程都会有一个PoolThreadCache对象，每个PoolThreadCache对象都会有tiny、small、normal三种规格的缓存。每种规格又分heap和direct，所以每个PoolThreadCache有6种缓存。PoolThreadCache正是使用了6个MemoryRegionCache数组来维护这6种缓存。比如：



数组tinySubPageHeapCaches拥有32个MemoryRegionCache元素，下标为n的元素用于缓存大小为n * 16B的ByteBuf。



数组smallSubPageHeapCaches拥有4个MemoryRegionCache元素，下标为n的元素用于缓存大小为2^n * 512B的ByteBuf。



数组normalHeapCaches拥有3个MemoryRegionCache元素，下标为n的元素用于缓存大小为2^n * 8K的ByteBuf。



数组tinySubPageHeapCaches里的每个MemoryRegionCache元素，最多可以缓存tinyCacheSize个即512个ByteBuf。



数组smallSubPageHeapCaches里的每个MemoryRegionCache元素，最多可以缓存smallCacheSize个即256个ByteBuf。



数组normalHeapCaches里的每个MemoryRegionCache元素，最多可以缓存normalCacheSize个即64个ByteBuf。

**
**

***\*10.命中缓存的分配流程\****

***\*(1)内存分配的入口\****

内存分配的入口是PooledByteBufAllocator内存分配器的newHeapBuffer()方法或newDirectBuffer()方法，其中这两个方法又会执行heapArena.allocate()方法或者directArena.allocate()方法，所以内存分配的入口其实就是PoolArena的allocate()方法。

**
**

***\*(2)首先进行分段规格化\****

normalizeCapacity()方法会根据reqCapacity进行分段规格化，目的是为了让内存在分配完后、后续在release时可以直接放入缓存里而无须进行释放。



当reqCapacity是tiny类型的内存规格时它是以16B进行自增，会把它当成16B的n倍。当reqCapacity是small类型的内存规格时它是以2的倍数进行自增，会把它变成512B的2^n倍。当reqCapacity是normal类型的内存规格时它是以2的倍数进行自增，会把它变成8K的2^n倍。

**
**

***\*(3)然后进行缓存分配\****

在进行缓存分配时会有3种规格：一是cache.allocateTiny()方法，二是cache.allocateSmall()方法，三是cache.allocateNormal()方法。这三种类型的原理差不多，下面以cache.allocateTiny()方法为例介绍命中缓存后的内存分配流程。

**
**

***\**\*步骤一：首先找到size对应的MemoryRegionCache。\*\**\***也就是说需要在一个PoolThreadCache里找到一个节点，这个节点是缓存数组中的一个MemoryRegionCache元素。



PoolThreadCache.cacheForTiny()方法的目的就是根据规格化后的需要分配的size去找到对应的MemoryRegionCache节点。该方法会首先将需要分配的size除以16，得出tiny缓存数组的索引，然后通过数组下标的方式去拿到对应的MemoryRegionCache节点。

**
**

***\**\*步骤二：然后从queue中弹出一个Entry给ByteBuf初始化。\*\**\***每一个Entry都代表了某一个Chunk下的一段连续内存。初始化ByteBuf时会把这段内存设置给ByteBuf，这样ByteBuf底层就可以依赖这些内存进行数据读写。首先通过queue.poll()弹出一个Entry元素，然后执行initBuf()方法进行初始化。初始化的关键在于给PooledByteBuf的成员变量赋值，比如chunk表示在哪一块内存进行分配、handle表示在这块chunk的哪一段内存进行分配，因为一个ByteBuf对象通过一个chunk和一个handle就能确定一块内存。

**
**

***\**\*步骤三：最后将弹出的Entry放入对象池里进行复用。\*\**\***Entry被弹出之后其实就不会再被用到了，而Entry本身也是一个对象。在PooledByteBuf对象初始化完成后，该Entry对象就不再使用了，不再使用的对象有可能会被GC垃圾回收掉。



而Netty为了让对象尽可能复用，会对Entry对象进行entry.recycle()处理，也就是把Entry对象放入到RECYCLE对象池中。后续当ByteBuf对象需要进行回收的时候，就可以直接从RECYCLE对象池中取出该Entry元素。然后把该Entry元素里对应的chunk和handle指向已被回收的ByteBuf对象来实现复用。Netty会尽可能做到对象的复用，它会通过一个RECYCLE对象池的方式去减少GC，从而减少对象的重复创建和销毁。

**
**

***\*11.Netty里有关内存分配的重要概念\****

***\*(1)PoolArena\****

***\*一.PoolArena的作用\****

当一个线程使用PooledByteBufAllocator内存分配器创建一个PooledByteBuf时，首先会通过ThreadLocal拿到属于该线程的一个PoolThreadCache对象，然后通过PoolArena的newByteBuf()方法创建出一个PooledByteBuf对象，接着调用PoolArena的allocate()方法为这个ByteBuf对象基于PoolThreadCache去分配内存。



PoolThreadCache有两大成员变量：一类是不同内存规格大小的MemoryRegionCache，另一类是PoolArena。PoolThreadCache中的PoolArena分为heapArena和directArena，通过PoolArena可以在PoolChunk里划分一块连续的内存分配给ByteBuf对象。和MemoryRegionCache不一样的是，PoolArena会直接开辟一块内存，而MemoryRegionCache是直接缓存一块内存。

**
**

***\*二.PoolArena的数据结构\****

PoolArena中有一个双向链表，双向链表中的每一个节点都是一个PoolChunkLisk。PoolChunkLisk中也有一个双向链表，双向链表中的每一个节点都是一个PoolChunk。Netty向操作系统申请内存的最小单位就是PoolChunk，也就是16M。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/DXGTicJyJ8QCyEBV5swDYspiaCwmeqLdqW8j54IwvR6STwCp7hDDpXA578W4eXQlqxGyq4awo5ZGdX5XCjkFvIuA/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

***\*(2)PoolChunk\****

为什么PoolArena要通过双向链表的方式把PoolChunkList连接起来，且PoolChunkList也通过双向链表的方式把PoolChunk连接起来？那是因为Netty会实时计算每一个PoolChunk的使用率情况，比如16M分配了8M则使用率为50%。然后把同样使用率范围的PoolChunk放到同一个PoolChunkList中。这样在为ByteBuf寻找一个PoolChunk分配内存时，就可以通过一定的算法找到某个PoolChunkList，然后在该PoolChunkList中选择一个PoolChunk即可。

**
**

***\*(3)Page和SubPage\****

由于一个PoolChunk的大小是16M，每次分配内存时不可能直接去分配16M的内存，所以Netty又会把一个PoolChunk划分为大小一样的多个Page。Netty会把一个PoolChunk以8K为标准划分成一个个的Page(2048个Page)，这样分配内存时只需要以Page为单位进行分配即可。



比如要分配16K的内存，那么只需要在一个PoolChunk里找到连续的两个Page即可。但如果要分配2K的内存，那么每次去找一个8K的Page来分配又会浪费6K的内存。所以Netty会继续把一个Page划分成多个SubPage，有的SubPage大小是按2K来划分的，有的SubPage大小是按1K来划分的。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/DXGTicJyJ8QCyEBV5swDYspiaCwmeqLdqWGMnkf1Xo1zrqoS14kGPJyia3J6EgjZgMkQKlYzYmXpqhyk77bv4NBCw/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

PoolArena中有两个PoolSubpage数组，其中tinySubpagePools有32个元素，分别代表16B、32B、48B、...、480、496B的SubPage。其中smallSubpagePools有4个元素，分别代表512B、1K、2K、4K的SubPage。



PoolSubpage中的chunk属性表示该SubPage从属于哪个PoolChunk，PoolSubpage中的elemSize属性表示该SubPage是以多大的数值进行划分的，PoolSubpage中的bitmap属性会用来记录该SubPage的内存分配情况，一个Page里的PoolSubpage会连成双向链表。

**
**

***\*(4)Netty内存分配总结\****

首先从线程对应的PoolThreadCache里获取一个PoolArena，然后从PoolArena的一个ChunkList中取出一个Chunk进行内存分配。接着在这个Chunk上进行内存分配时，会判断需要分配的内存大小是否大于一个Page的大小。如果需要分配的内存超过一个Page的大小，那么就以Page为单位进行内存分配。如果需要分配的内存远小于一个Page的大小，那么就会找一个Page并把该Page切分成多个SubPage然后再从中选择。

**
**

***\*12.Page级别的内存分配\****

***\*(1)Page级别的内存分配的入口\****

下面这3行代码可以用来跟踪进行Page级别内存分配时的调用栈：

- 
- 
- 

```
PooledByteBufAllocator allocator = PooledByteBufAllocator.DEFAULT;ByteBuf byteBuf = allocator.heapBuffer(8192);//分配8192B内存byteBuf.release();
```

PooledByteBufAllocator的heapBuffer()方法通过其newHeapBuffer()方法执行代码heapArena.allocate()时，首先会调用PoolArena的allocate()方法，然后会调用PoolArena的allocateNormal()方法，接着会调用PoolChunk的allocate()方法，并最终调用到PoolChunk的allocateRun()方法进行Page级别的内存分配。进行Page级别的内存分配时只会分配多个Page而不会分配一个Page的一部分。



注意：真正要分配的内存其实就是byte[]或者ByteBuffer，所以实际的分配就是得到一个数值handle进行定位以方便后续的写。

**
**

***\*(2)Page级别的内存分配的流程\****

***\*一.尝试在现有的PoolChunk上分配\****

一个PoolArena里会有一个由多个PoolChunkList连接起来的双向链表，每个PoolChunkList代表了某种内存使用率的PoolChunk列表。PoolArena的allocateNormal()方法首先会尝试从某使用率的PoolChunkList中获取一个PoolChunk来分配，如果没法从某使用率的PoolChunkList中获取一个PoolChunk进行分配，那么就新创建一个PoolChunk。

**
**

***\*二.创建一个PoolChunk并进行内存分配\****

通过PoolArena的newChunk()方法去创建一个PoolChunk，然后调用该PoolChunk的allocate()方法进行内存分配。也就是调用PoolChunk的allocateRun()方法去确定PoolChunk里的一块连续内存，这块连续内存会由一个long型的变量handle来指向。

**
**

***\*三.初始化PooledByteBuf对象\****

在拿到一个PoolChunk的一块连续内存后(即allocateRun()方法返回的handle标记)，需要执行PoolChunk的initBuf()方法去把handle标记设置到PooledByteBuf对象上。

**
**

***\*四.将新建的PoolChunk添加PoolChunkList\****

也就是将新建的PoolChunk添加到PoolArena的qInit这个PoolChunkList中。

**
**

***\*(3)尝试在现有的PoolChunk上分配\****

在PoolChunkList的allocate()方法中：首先会从PoolChunkList的head节点开始往下遍历，然后对每一个PoolChunk都尝试调用PoolChunk.allocate()方法进行分配。如果PoolChunk.allocate()方法返回的handle大于0，接下来会调用PoolChunk.initBuf()方法对PooledByteBuf进行初始化，并且如果当前PoolChunk的使用率大于所属PoolChunkList的最大使用率，那么需要将当前PoolChunk从所在的PoolChunkList中移除并加入到下一个PoolChunkList中。

**
**

***\*(4)创建一个PoolChunk进行内存分配\****

***\*一.创建PoolChunk时的入参和构造方法\****

PoolArena的allocateNormal()方法在分配Page级别的内存时，会调用newChunk(pageSize, maxOrder, pageShifts, chunkSize)，其中pageSize = 8K、maxOrder = 11、pageShifts = 13、chunkSize = 16M。newChunk()方法会由PoolArena的内部类兼子类DirectArena和HeapArena实现。



在HeapArena和DirectArena的newChunk()方法中，会new一个PoolChunk。而在PoolChunk的构造方法中，除了设置newChunk()方法传入的参数外，还会初始化两个数组memoryMap和depthMap来表示不同规格的连续内存使用分配情况。其中的memoryMap是一个有4096个元素的字节数组，depthMap在初始化时和memoryMap完全一样。

**
**

***\*二.PoolChunk的memoryMap属性的数据结构\****

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/DXGTicJyJ8QCyEBV5swDYspiaCwmeqLdqWkHFiaQ0QzysWbowyAFaLydz9YzGq87iciaGSTJ59zK3dibUxkc3oYq9Ffg/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

首先PoolChunk是以Page的方式去组织内存的，然后memoryMap属性中的每个结点都表示一块连续内存是否已被分配，以及depthMap属性中的每个结点都表示其所在树的深度或层高。memoryMap数组中下标为n的元素对应于一棵高度为12的连续内存的平衡二叉树的第n个结点。比如0-4M这个连续内存结点是整棵连续内存的平衡二叉树中的第4个结点，其所在树的深度是2。注意：0-16M对应在memoryMap数组中的索引是1，memoryMap数组中索引为0的元素是空的。

**
**

***\*三.在新创建的PoolChunk上分配内存\****

PoolArena的allocateNormal()方法在创建完一个PoolChunk后，就要从这个PoolChunk上分配一块内存，于是会调用PoolChunk的allocate()方法。由于进行的是Page级别的内存分配，所以最终会调用PoolChunk的allocateRun()方法。



PoolChunk的allocateRun()方法首先根据需要分配的内存大小算出将被分配的连续内存结点在平衡二叉树中的深度，然后再根据PoolChunk的allocateNode()方法取得将被分配的连续内存结点在memoryMap数组中的下标id。



也就是说，执行allocateNode(d)会获得：在平衡二叉树中的 + 深度为d的那一层中 + 还没有被使用过的 + 一个连续内存结点的索引。获得这个索引之后，便会设置memoryMap数组在这个索引位置的元素的值为unusable = 12表示不可用，以及逐层往上标记结点不可用。其中，allocateNode()方法会通过异或运算实现 + 1。

**
**

***\*(5)初始化PooledByteBuf对象\****

在PoolChunk的initBuf()方法中，首先会根据handle计算出memoryMapIdx和bitMapIdx。由于这里进行的是Page级别的内存分配，所以bitMapIdx为0，于是接下来会调用buf.init()方法进行PooledByteBuf的初始化。



在PooledByteBuf的构造函数中：首先会设置PoolChunk，因为要拿到一块可写的内存首先需要拿到一个PoolChunk。然后设置handle，表示指向的是PoolChunk中的哪一块连续内存，也就是平衡二叉树中的第几个结点，或者是memoryMap中的第几个元素。接着设置memory，表示分配PoolChunk时是通过Heap还是Direct方式进行申请的。以及设置offset为0，因为这里进行的是Page级别的内存分配，所以没有偏移量。最后设置length，表示实际这次内存分配到底分配了多少内存。

**
**

***\*13.SubPage级别的内存分配\****

***\*(1)SubPage级别的内存分配的入口\****

下面这3行代码可以用来跟踪进行SubPage级别内存分配时的调用栈：

- 
- 
- 

```
PooledByteBufAllocator allocator = PooledByteBufAllocator.DEFAULT;ByteBuf byteBuf = allocator.directBuffer(16);//分配16B内存byteBuf.release();
```

PooledByteBufAllocator的directBuffer()方法通过其newDirectBuffer()方法执行代码directArena.allocate()时，首先会调用PoolArena的allocate()方法，然后会调用PoolArena的allocateNormal()方法，接着会调用PoolChunk的allocate()方法，并最终调用到PoolChunk的allocateSubpage()方法进行SubPage级别的内存分配。进行SubPage级别的内存分配时不会分配多个Page而只会分配一个Page的一部分。



在PoolArena的allocate()方法中，首先会通过tinyIdx(16)拿到tableIdx，此时tableIdx = 16 >>> 4 = 1。然后从PoolArena的tinySubpagePools数组中取出下标为tableIdx的一个PoolSubpage元素赋值给table。tinySubpagePools默认和MemoryRegionCache的缓存一样，也有32个元素：16B、32B、48B、...、480、496B。其中第n个元素表示大小为(n - 1) * 16B的专属SubPage。接着通过PoolArena.allocateNormal()方法调用到PoolChunk.allocateSubpage()方法。

**
**

***\*(2)SubPage级别的内存分配的流程\****

PoolChunk.allocateSubpage()方法的主要操作：

一.定位一个SubPage对象

二.初始化SubPage对象

三.调用SubPage的allocate()方法进行分配

**
**

***\*(3)定位一个SubPage对象\****

SubPage是基于一个Page进行划分的，不管是从一个现有的SubPage对象中分配，还是在没有SubPage对象时创建一个SubPage，第一个步骤都是定位一个SubPage对象。



在PoolChunk的allocateSubpage()方法中：首先会通过PoolArena的findSubpagePoolHead()方法去找到在PoolArena的tinySubpagePools中用于存放16B的PoolSubpage结点。然后在连续内存的平衡二叉树的第11层上分配一个Page结点，即通过allocateNode(d)找到一个Page在PoolChunk中的下标id。接着通过subpageIdx(id)确定PoolChunk中第subpageIdx个Page会以SubPage方式存在，从而定位到在PoolChunk的subpages数组中下标为subpageIdx对应的PoolSubpage元素可用来进行SubPage级别的内存分配。



注意：在PoolChunk的subpages数组中，如果某个下标对应的PoolSubpage元素为空，则说明这个下标对应的PoolChunk中的某个Page已经用来进行了Page级别的内存分配或者还没被分配。

**
**

***\*(4)初始化SubPage对象\****

如果PoolChunk的subpages数组中下标为subpageIdx的PoolSubpage元素为空，那么就会创建一个PoolSubpage对象并对其进行初始化。初始化的过程就是去一个PoolChunk里寻找一个Page，然后按照SubPage大小将该Page进行划分。当完成PoolSubpage对象的初始化之后，就可以通过它的allocate()方法来进行内存分配了。具体来说就是把内存信息设置到PooledByteBuf对象中。



PoolSubpage的构造方法调用init()方法的处理：

一.设置elemSize用于表示一个SubPage的大小，比如规格化后所申请的内存大小：16B、32B等。

二.设置bitmap用于标识把一个Page划分成多个SubPage后哪一个SubPage已被分配，0表示未分配，1表示已分配。

三.通过addToPool()方法把当前PoolSubpage对象添加到PoolArena的tinySubpagePools数组中可以分配某种规格大小内存的PoolSubpage链表里。

**
**

***\*(5)调用SubPage的allocate()方法进行分配\****

首先从位图bitmap里寻找一个未被使用的SubPage。如果可用的SubPage的数量为0，则直接把该PoolSubpage对象从PoolArena的tinySubpagePools数组的某种规格的结点中移除。接着将代表未使用SubPage的bitmapIdx转换成handle，也就是拼接成64位 + bitmapIdx变成高32位 + memoryMapIdx变成低32位，所得的handle便表示一个PoolChunk里第几个Page的第几个SubPage，从而可以拿到连续内存给PooledByteBuf进行初始化。

**
**

***\*14.ByteBuf的回收\****

***\*(1)池化的内存如何释放\****

比如byteBuf.release()会调用到PooledByteBuf的deallocate()方法。该方法首先会清空PooledByteBuf对象的handle、chunk、memory变量值。然后调用PoolArena的free()方法去释放对应PoolChunk在handle处的内存，也就是将连续内存的区段添加到缓存 + 标记连续内存的区段为未使用。接着调用PooledByteBuf的recycle()方法去复用PooledByteBuf对象。

**
**

***\*(2)将连续内存的区段加到缓存\****

进行内存分配时，第一个步骤就是从缓存里寻找是否有对应大小的连续内存区段，如果有就直接取出来进行分配。如果释放内存时，将连续内存的区段添加到缓存成功了，那么下次进行内存分配时，对于相同大小的PooledByteBuf就可以从缓存中直接取出来进行使用了。如果释放内存时，将连续内存的区段添加到缓存不成功，比如缓存队列已经满了就会不成功，那么就标记该PooledByteBuf对应的连续内存区段为未使用。



在PoolArena的free()方法中，首先会调用PoolThreadCache的add()方法将释放的连续内存区段添加到缓存，然后调用PoolArena的freeChunk()方法标记连续内存的区段为未使用。

**
**

***\*(3)标记连续内存的区段为未使用\****

标记方式会根据Page级别和SubPage级别进行标记，其中Page级别是根据二叉树来进行标记，SubPage级别是通过位图进行标记。

**
**

***\*(4)将ByteBuf对象添加到对象池\****

一开始时，对象池是没有PooledByteBuf对象的，当PooledByteBuf对象被释放时不会被立即销毁，而是会加入到对象池里。



这样当Netty每次去拿一个PooledByteBuf对象时，就可以先从对象池里获取，取出对象之后就可以进行内存分配以及初始化了。



考虑到PooledByteBuf对象会经常被申请和释放，如果QPS非常高，可能会产生很多PooledByteBuf对象，而且频繁创建和释放PooledByteBuf对象也会比较耗费资源和降低性能。



所以Netty便使用了对象池来减少GC：当申请PooledByteBuf对象时，就可以尽可能从对象池里去取。当释放PooledByteBuf对象时，则可以将对象添加到对象池，从而实现对象复用。

**
**

***\*15.读数据入口\****

当客户端Channel的Reactor线程NioEventLoop检测到有读事件时，会执行NioByteUnsafe的read()方法。该方法会调用doReadBytes()方法将TCP缓冲区的数据读到由ByteBufAllocator分配的一个ByteBuf对象中，然后通过pipeline.fireChannelRead()方法带上这个ByteBuf对象向下传播ChannelRead事件。



在传播的过程中，首先会来到pipeline的head结点的channelRead()方法。该方法会继续带着那个ByteBuf对象向下传播ChannelRead事件，比如会来到ByteToMessageDecoder结点的channelRead()方法。



注意：服务端Channel的unsafe变量是一个NioMessageUnsafe对象，客户端Channel的unsafe变量是一个NioByteUnsafe对象。NioByteUnsafe主要会进行如下处理：



一.通过客户端Channel的ChannelConfig获取内存分配器ByteBufAllocator，然后用内存分配器来分配一个ByteBuf对象

二.将客户端Channel中的TCP缓冲区的数据读取到ByteBuf对象

三.读取完数据后便调用DefaultChannelPipeline的fireChannelReadComplete()方法，从HeadContext结点开始在整个ChannelPipeline中传播ChannelRead事件

**
**

***\*16.拆包原理\****

***\*(1)不用Netty的拆包原理\****

不断地从TCP缓冲区中读取数据，每次读完都判断是否为一个完整的数据包。如果当前读取的数据不足以拼接成一个完整的数据包，就保留数据，继续从TCP缓冲区中读。如果当前读取的数据加上已读取的数据足够拼成完整的数据包，就将拼好的数据包往业务传递，而多余的数据则保留。

**
**

***\*(2)Netty的拆包原理\****

Netty拆包基类内部会有一个字节容器，每次读取到数据就添加到字节容器中。然后尝试对累加的字节数据进行拆包，拆成一个完整的业务数据包，这个拆包基类叫ByteToMessageDecoder。

**
**

***\*17.解码器抽象的解码过程总结\****

解码过程是通过一个叫ByteToMessageDecoder的抽象解码器来实现的，ByteToMessageDecoder实现的解码过程分为如下四步。



步骤一：累加字节流，也就是把当前读到的字节流累加到一个字节容器里。

步骤二：调用子类的decode()方法进行解析，ByteToMessageDecoder的decode()方法是一个抽象方法，不同种类的解码器会有自己的decode()方法逻辑。该decode()方法被调用时会传入两个关键参数：一个是ByteBuf对象表示当前累加的字节流，一个是List列表用来存放被成功解码的业务数据包。

步骤三：清理字节容器，为了防止发送端发送数据过快，ByteToMessageDecoder会在读取完一次数据并完成业务拆包后，清理字节容器。

步骤四：传播已解码的业务数据包，如果List列表里有解析出来的业务数据包，那么就通过pipeline的事件传播机制往下进行传播。

**
**

***\*18.如何把对象变成字节流写到unsafe底层\****

当调用ctx.channel().writeAndFlush(user)将自定义的User对象沿着整个Pipeline进行传播时：



首先会调用tail结点的write()方法开始往前传播，传播到一个继承自MessageToByteEncoder的结点。该结点会实现MessageToByteEncoder的encode()方法来把自定义的User对象转换成一个ByteBuf对象。转换的过程首先会由MessageToByteEncoder分配一个ByteBuf对象，然后再调用其子类实现的抽象方法encode()将User对象填充到ByteBuf对象中。填充完之后继续调用write()方法把该ByteBuf对象往前进行传播，默认下最终会传播到head结点。



其中head结点的write()方法会通过底层的unsafe进行如下处理：把当前的ByteBuf对象添加到unsafe维护的一个写缓冲区里，同时计算写缓冲区大小是否超过64KB。如果写缓冲区大小超过了64KB，则设置当前Channel不可写。完成write()方法的传播后，head结点的unsafe对象维护的写缓冲区便对应着一个ByteBuf队列，它是一个单向链表。



然后会调用tail结点的flush()方法开始往前传播，默认下最终会传播到head结点。head结点在接收到flush事件时会通过底层的unsafe进行如下处理：首先进行指针调整，然后通过循环遍历从写缓冲区里把ByteBuf对象取出来。每拿出一个ByteBuf对象都会把它转化为JDK底层可以接受的ByteBuffer对象，最终通过JDK的Channel把该ByteBuffer对象写出去。每写完一个ByteBuffer对象都会把写缓冲区里的当前ByteBuf所在的Entry结点进行删除，并且判断如果当前写缓冲区里的大小已经小于32KB就通过自旋 + CAS重新设置Channel为可写。

**
**

***\*19.Netty的两大性能优化工具\****

***\*(1)FastThreadLocal\****

FastThreadLocal的作用与ThreadLocal相当，但比ThreadLocal更快。ThreadLocal的作用是多线程访问同一变量时能够通过线程本地化的方式避免多线程竞争、实现线程隔离。Netty的FastThreadLocal重新实现了JDK的ThreadLocal的功能，且访问速度更快，但前提是使用FastThreadLocalThread线程。

**
**

***\*(2)Recycler\****

Recycler实现了一个轻量级的对象池机制。对象池的作用是一方面可以实现快速创建对象，另一方面可以避免反复创建对象、减少YGC压力。Netty使用Recycler的方式来获取ByteBuf对象的原因是：ByteBuf对象的创建在Netty里是非常频繁的且又比较占空间。但是如果对象比较小，使用对象池也不是那么划算。

**
**

***\*20.FastThreadLocal的总结\****

***\*(1)FastThreadLocal不一定比ThreadLocal快\****

只有FastThreadLocalThread线程使用FastThreadLocal才会更快，如果普通线程使用FastThreadLocal其实和普通线程使用ThreadLocal是一样的。

**
**

***\*(2)FastThreadLocal并不会浪费很大的空间\****

虽然FastThreadLocal采用了空间换时间的思路，但其设计之初就认为不会存在特别多的FastThreadLocal对象，而且在数据中没有使用的元素只是存放了同一个缺省对象的引用(UNSET)，所以并不会占用太多内存空间。

**
**

***\*(3)FastThreadLocal如何实现高效查找\****

***\*一.FastThreadLocal\****

在定位数据时可以直接根据数组下标index进行定位，时间复杂度为O(1)，所以在数据较多时也不会存在Hash冲突。在进行数组扩容时只需要把数组容量扩容2倍，然后再把原数据拷贝到新数组。

**
**

***\*二.ThreadLocal\****

在定位数据时是根据哈希算法进行定位的，在数据较多时容易发生Hash冲突。发生Hash冲突时采用线性探测法解决Hash冲突要不断向下寻找，效率较低。在进行数组扩容时由于采用了哈希算法，所以在数组扩容后需要再做一轮rehash。

**
**

***\*(4)FastThreadLocal具有更高的安全性\****

FastThreadLocal不仅提供了remove()方法可以主动清除对象，而且在线程池场景中(也就是SingleThreadEventExecutor和DefaultThreadFactory)，Netty还封装了FastThreadLocalRunnable。



FastThreadLocalRunnable最后会执行FastThreadLocal的removeAll()方法将Set集合中所有的FastThreadLocal对象都清理掉。



ThreadLocal使用不当可能会造成内存泄露，只能等待线程销毁。或者只能通过主动检测的方式防止内存泄露，从而增加了额外的开销。

**
**

***\*21.Recycler的设计理念\****

***\*(1)对象池和内存池都是为了提高并发处理能力\****

Java中频繁创建和销毁对象的开销是很大的，所以通常会将一些对象缓存起来。当需要某个对象时，优先从对象池中获取对象实例。通过重用对象不仅避免了频繁创建和销毁对象带来的性能损耗，而且对于JVM GC也是友好的，这就是对象池的作用。

**
**

***\*(2)Recycler是Netty提供实现的轻量级对象池\****

通过Recycler，在创建对象时，就不需要每次都通过new的方式去创建了。如果Recycler里面有已经用过的对象，则可以直接把这个对象取出来进行二次利用。在不需要该对象时，只需要把它放到Recycler里以备下次需要时取出。

**
**

***\*22.Recycler的总结\****

***\*(1)获取对象和回收对象的思路总结\****

对象池有两个重要的组成部分：Stack和WeakOrderQueue。从Recycler获取对象时，优先从Stack中查找可用对象。如果Stack中没有可用对象，会尝试从WeakOrderQueue迁移一些对象到Stack中。Recycler回收对象时，分为同线程对象回收和异线程对象回收这两种情况。同线程回收直接向Stack添加对象，异线程回收会向WeakOrderQueue中的最后一个Link添加对象。同线程回收和异线程回收都会控制回收速率，默认每8个对象会回收一个，其他的全部丢弃。

**
**

***\*(2)获取对象的具体步骤总结\****

如何从一个对象池里获取对象：

***\**\*步骤一：\*\**\***首先通过FastThreadLocal方式拿到当前线程的Stack。

***\**\*步骤二：\*\**\***如果这个Stack里的elements数组有对象，则直接弹出。如果这个Stack里的elements数组没有对象，则从当前Stack关联的其他线程的WeakOrderQueue里的Link结点的elements数组中转移对象，到当前Stack里的elements数组里。

***\**\*步骤三：\*\**\***如果转移成功，那么当前Stack里的elements数组就有对象了，这时就可以直接弹出。如果转移失败，那么接下来就直接创建一个对象然后和当前Stack进行关联。

***\**\*步骤四：\*\**\***关联之后，后续如果是当前线程自己进行对象回收，则将该对象直接存放到当前线程的Stack里。如果是其他线程进行对象回收，则将该对象存放到其他线程与当前线程的Stack关联的WeakOrderQueue里。

**
**

***\*(3)对象池的设计核心\****

为什么要分同线程和异线程进行处理，并设计一套比较复杂的数据结构？因为对象池的使用场景一般是高并发的环境，希望通过对象池来减少对象的频繁创建带来的性能损耗。所以在高并发的环境下，从对象池中获取对象和回收对象就只能通过以空间来换时间的思路进行处理，而ThreadLocal恰好是通过以空间换时间的思路来实现的，因此引入了FastThreadLocal来管理对象池里的对象。



但是如果仅仅使用FastThreadLocal管理同线程创建和回收的对象，那么并不能充分体现对象池的作用。所以通过FastThreadLocal获取的Stack对象，应该不仅可以管理同线程的对象，也可以管理异线程的对象。为此，Recycler便分同线程和异线程进行处理并设计了一套比较复杂的数据结构。

Netty · 目录

上一篇面试准备之Netty要点总结二

个人观点，仅供参考

阅读 248
