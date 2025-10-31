# Redis相关题目

## redis.md

**一.Redis的数据结构**

**1.Redis的数据结构**

**2.Redis的SDS**

**3.Redis的链表**

**4.Redis的字典**

**5.Redis的跳跃表**

**6.Redis的整数集合**

**7.Redis的压缩列表**

**8.Redis的对象**

**9.Redis的单线程为什么这么快**

**二.Redis的数据库原理**

**1.Redis数据库的构成**

**2.Redis过期键的删除策略**

**3.Redis的RDB持久化**

**4.Redis的AOF持久化**

**5.Redis的AOF重写机制**

**6.Redis基于子进程实现持久化的使用建议**

**7.Redis服务器的文件事件**

**8.Redis服务器的文件事件处理器**

**9.Redis对文件事件的处理流程**

**10.Redis服务器对命令请求的处理**

**三.Redis的复制原理**

**1.Redis使用sync命令实现的复制功能**

**2.Redis使用psync命令实现的复制功能**

**3.Redis主从服务器之间的心跳检测**

**4.从服务器如何实现复制主服务器的(复制的实现)**

**5.Redis的复制拓扑**

**6.Redis主从复制数据延迟的处理**

**7.Redis主从复制的优缺点**

**四.Redis的哨兵原理**

**1.Redis Sentinel和高可用**

**2.Redis如何保存更多的数据**

**3.一个普通Redis服务器的初始化过程**

**4.一个Sentinel服务器的初始化过程**

**5.Sentinel如何向主从服务器获取信息和发送信息**

**6.Sentinel如何检测主客观下线并实现故障转移**

**7.Sentinel客户端的基本实现原理**

**8.Sentinel的基本实现原理(哨兵机制的基本流程)**

**9.关于Sentinel的一些问题**

**五.Redis的集群原理**

**1.Redis Cluster集群的简介**

**2.Redis Cluster集群搭建的步骤**

**3.Redis Cluster集群执行命令的实现原理**

**4.Redis Cluster集群节点通信的实现原理**

**5.Redis Cluster集群复制与故障转移的实现原理**

**6.通过Smart客户端支持Redis Cluster集群**

**7.Redis Cluster集群的补充说明**

**8.Redis Cluster集群的倾斜问题**

**9.Redis Cluster集群的核心问题**

## redis_2.md

注意：该文件内容与jvm.md重复，都是关于G1垃圾回收器的内容。

**一.G1的分区机制和停顿预测模型**

**1.G1垃圾回收器的简介**

**2.HeapRegion是G1内存管理的基本单位**

**3.ParNew +CMS与G1的内存模型对比**

**4.G1如何设置HeapRegion(HR)的大小**

**5.应该如何设置G1的新生代内存大小**

**6.G1新生代扩展流程(新生代分区扩展流程)**

**7.停顿预测模型和衰减算法**

**二.G1的对象分配效率和垃圾回收效率**

