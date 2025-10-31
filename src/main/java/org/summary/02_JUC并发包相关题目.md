# JUC并发包相关题目

## juc.md

**1.Java集合包源码**

(1)ArrayList源码总结

(2)LinkedList源码总结

(3)ArrayList和LinkedList的区别

(4)栈的数据结构总结

(5)HashMap的源码之数组 + 链表 + 红黑树

(6)HashMap的底层原理总结

(7)迭代器应对多线程并发修改的Fail-Fast机制

**2.Thread源码分析**

(1)线程的运行状态

(2)如何减少线程上下文切换

(3)创建和启动一个线程的主要方法

(4)以daemon模式运行微服务的存活监控线程

(5)Thread线程初始化要点总结

(6)Thread线程的启动源码

(7)yield()方法可切换当前线程执行其他线程

(8)join()方法实现服务注册线程的阻塞式运行

**3.volatile关键字的原理**

**4.Java内存模型JMM**

**5.JMM如何处理并发中的原子性可见性有序性**

**6.volatile如何保证可见性**

**7.volatile的原理(Lock前缀指令 + 内存屏障)**

**8.双重检查单例模式的volatile优化**

**9.synchronized关键字的原理**

**10.wait()与notify()的底层原理**

**11.Atomic原子类中的CAS无锁化原理**

**12.LongAdder的分段CAS优化多线程自旋**

## juc_2.md

**1.JUC中的Lock接口**

**2.如何实现具有阻塞或唤醒功能的锁**

**3.ReentractLock如何获取锁**

**4.AQS的acquire()方法和release()方法总结**

**5.ReentractReadWriteLock的基本原理**

**6.ReentractReadWriteLock如何竞争写锁**

**7.ReentractReadWriteLock如何竞争读锁**

**8.ReentractReadWriteLock的公平锁和非公平锁**

**9.Condition的说明和源码**

**10.等待多线程完成的CountDownLatch**

**11.控制并发线程数的Semaphore**

**12.同步屏障CyclicBarrier**

**13.ConcurrentHashMap的原理**

**14.并发安全的数组列表CopyOnWriteArrayList**

**15.并发安全的链表队列ConcurrentLinkedQueue**

**16.JUC的各种阻塞队列介绍**

**17.LinkedBlockingQueue的具体实现原理**

**18.基于两个队列实现的集群同步机制**

**19.锁优化总结**

**20.线程池介绍**

**21.如何设计一个线程池**

**22.ThreadPoolExecutor线程池的执行流程**

**23.如何合理设置线程池参数 + 定制线程池**

**24.ThreadLocal的设计原理**

**25.Disruptor的生产者源码**

**26.Disruptor的消费者源码**

**27.Disruptor的WaitStrategy等待策略**

**28.Disruptor的高性能原因**

