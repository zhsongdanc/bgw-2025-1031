# MySQL相关题目

## mysql.md

**一.InnoDB的内存结构和更新机制**

**1.SQL的执行流程**

**2.InnoDB的内存模型**

(1)缓冲池Buffer Pool的默认大小是128MB

(2)数据页是MySQL抽象出来的数据单位

(3)Buffer Pool中每个缓存页都有对应的描述数据

**3.Buffer Pool中的空闲缓存页与free链表**

(1)数据库启动时会按照设置的Buffer Pool大小向OS申请内存

(2)Buffer Pool有一个叫free链表的双向链表

(3)根据free链表的节点可得到一个空闲缓存页

(4)增删改查一条数据时InnoDB引擎会怎么处理

**4.Buffer Pool中的脏页和flush链表**

(1)通过free链表来管理所有空闲的数据页

(2)通过哈希表来管理在Buffer Pool缓存的数据页

(3)通过flush链表来管理更新后等待被刷盘的缓存页

**5.Buffer Pool通过LRU链表来淘汰缓存页**

(1)简单LRU链表的工作原理

(2)简单LRU链表可能存在的问题

(3)基于冷热数据分离思想设计LRU链表

**6.Buffer Pool的缓存页以及几个链表总结**

(1)当加载一个数据页到一个缓存页时

(2)当修改一个缓存页时

(3)当查询一个缓存页时

**7.LRU链表的冷数据区域的缓存页何时刷盘**

时机一：定时把LRU尾部的部分缓存页刷入磁盘

时机二：把flush链表中的一些缓存页定时刷入磁盘

时机三：实在没有空闲缓存页时

**8.增大Buffer Pool来提升MySQL的并发能力**

**二.InnoDB的存储模型**

**1.InnoDB的存储模型以及对应的读写机制**

**2.提交事务时会将redo日志写入磁盘中**

(1)当innodb_flush_log_at_trx_commit = 0时

(2)当innodb_flush_log_at_trx_commit = 1时

(3)当innodb_flush_log_at_trx_commit = 2时

**3.MySQL的redo log和binlog对比**

**4.提交事务时同时也会写入binlog**

**5.在redo日志中写入commit标记的意义**

**6.后台IO线程随机将内存更新后的脏数据刷回磁盘**

**7.InnoDB的执行流程总结**

**8.redo日志和redo log机制的作用**

**9.redo日志会写入日志文件里的Redo Log Blcok**

**10.Redo Log Buffer和Redo Log文件**

**11.redo日志从Redo Log Buffer中刷盘时机**

**12.undo log回滚日志原理**

**13.系统和数据库能抗多少QPS**

**14.性能压测指标和命令**

**15.简单总结增删改SQL语句的实现原理**

**三.并发事务原理**

**四.索引原理和索引优化**

## mysql_2.md

**一.InnoDB的内存结构**

**1.Buffer Pool的概念**

**2.Buffer Pool的Page管理机制**

**3.Change Buffer的概念和作用**

**4.Log Buffer的概念和作用**

**二.InnoDB的存储结构**

**1.InnoDB磁盘结构**

**2.表空间(Tablespaces)**

**3.数据字典(Data Dictionary)**

**4.双写缓冲区(Double Write Buffer Files)**

**5.重做日志(redo log)**

**6.撤销日志(undo log)**

**7.二进制日志(binlog)**

**8.表空间文件结构**

**三.索引优化**

**1.Explain的字段说明**

**2.索引优化原则总结**

**3.慢查询优化思路**

**四.事务原理**

**1.ACID之原子性**

**2.ACID之持久性**

**3.ACID之隔离性**

**4.ACID之一致性**

**5.ACID的关系**

**6.MVCC事务控制**

**五.锁机制原理和索引原理**

## mysql_3.md

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

