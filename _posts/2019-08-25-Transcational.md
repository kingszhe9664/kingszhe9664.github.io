---
layout: post
title: 事务闲谈
categories: Java
description: 事务闲谈
keywords: Java
---

我们今天来讨论下事务（Transaction）：谈论这个前提 同样也是最重要的就是这个所谓的ACID原则
原子性(A : Atomicity) 一致性(C : consistency ) 隔离性(I : isolation) 持久性(D : durability )

**其中由于原子性 隔离性 导致 事务的代价高于一般的操作。**


首先我们来看下MYSQL中事务处理过程（顺带提一下MySQL的事务处理功能在MYSIAM存储引擎中是不支持的，在InnoDB存储引擎中是支持的）：

我的理解是：MYSQL中的事务是 begin开始 到commit 或者rollback结束的一系列的DML语句的集合。

我们今天要说的是数据库的redo log文件（实务操作以后的数据）和 undo log文件 （事务操作之前的数据）

Undo Log的原理很简单，为了满足事务的原子性，在操作任何数据之前，首先将数据备份到一个地方（这个存储数据备份的地方称为Undo Log）

如果出现任何问题 就会进行 ROLLBACK语句，系统就可以利用Undo Log中的备份将数据恢复到事务开始之前的状态。

Redo Log原理 和Undo Log相反，Redo Log记录的是新数据的备份。在事务提交前，只要将Redo Log持久化即可， 不需要将数据持久化。当系统崩溃时，虽然数据没有持久化，但是Redo Log已经持久化。系统可以根据Redo Log的内容，将所有数据恢复到最新的状态。

简单理解的话：其实可以： undo当作是 保存数据修改之前旧数据的镜像  redo当作是 保存数据修改之后的数据

数据库引擎会给 redo undo日志开辟内存缓存（log buffer）。磁盘上的日志文件称为(log file)，是顺序追加的(空间连续的)，性能比较高， 注：磁盘的顺序写性能比内存的写性能差不了多少。

此处有一点值得提出来的是： redo log的持久化 是在事务之后的!!!!!! 可能大家会有觉得奇怪。怎么没有写道log file 就提交事务了。

因为这样一是 减少了IO的开销，数据库引擎会在某一个时间把log buffer中的数据写入到log file 中去。

其二是；就算这个时候没有持久化 ，数据库宕机  因为存在log buffer的 关系。数据日志还在。根据日志就可以恢复数据。并不用担心数据的丢失。 这种持久化日志的策略叫做Write Ahead Log（WAL），即预写日志。

好多数据库都用到了此模式 MySQL,habase,zookeeper 非数据库也用到了此模式,在每一次更新节点的操作时，都会去做WAL。

当然事务涉及到的还有很多 比如spring中的事务管理：有事务隔离级别。传播行为等。





### 简单聊了下数据库的事务之后， 我们再来说目前被比较热门的分布式事务。

随着各种大型网站的各种高并发访问、海量数据处理等场景越来越多，为了提高网站的高可用性。现在普遍采用的就是分布式的架构。

由此也就引出了所谓的分布式事务：其实就是把原本对一个数据库的一系列业务操作扩展到对多个数据库上。这种时候再涉及整个系统的架构的时候。 再考虑事务问题就会理所当然的会想要用一个第三者来同意记录管理事务。
也就引出我们下面要提到的2PC（二阶段提交）。
    首先需要了解一个X/OPEN  DTP（Distributed Transaction Processing Reference Model）的 模型（分布式事务参考模型 ）。 首先解释下：X/OPEN  是一个组织机构 定义出的一套分布式事务标准

其实我们现在使用的 J2EE遵循X/OPEN 规范 设计实现了JTA：

JTA Transaction(事务) 分两种 首先说明  他其实并不是一种具体的实现，而是一套接口。

JTA提供了以下三个接口：
> 1.javax.transaction.UserTransaction，是面向开发人员的接口,能够编程地控制事务处理。
> 
> 2.javax.transaction.TransactionManager，允许应用程序服务器来控制代表正在管理的应用程序的事务。
> 
> 3.javax.transaction.xa.XAResource，面向提供商的实现接口，是一个基于X/Open CAE Specification的行业标准XA接口的Java映射。提供商在提供访问自己资源的驱动时，必须实现这样的接口。

Local Transaction 和 Global Transaction

 •涉及到一个连接的Commit，称为Local Transaction

 •涉及到多个连接的Commit，称为Global Transaction

Local Transaction  直接使用JDBC的事务实现 当然要使用Global Transaction 就比较复杂了。这里不深入介绍了。有兴趣的可以去了解下。

现在回到我们的2PC:这个
