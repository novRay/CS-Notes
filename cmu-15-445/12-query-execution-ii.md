# #12: Query Execution II

本章将介绍**parallel query execution**

## Background

### Why Care About Parallel Execution?

提高性能：增加吞吐量，降低延迟

提高responsiveness和availability

可能减少total cost of ownership（硬件采购、软件许可证、部署人力）

DBMS一般支持两种并行：inter-query parallelism和intra-query parallelism

### Parallel vs. Distributed

* Parallel DBMSs
  * 资源是physically相邻的
  * 资源间通过高速网络通信
  * 通信成本低且可靠
* Distributed DBMSs
  * 资源间距离可能非常远
  * 资源间通过较低速网络通信
  * 通信成本和问题不可忽略



本节大纲：

* Process Models
* Execution Parallelism
* I/O Parallelism

## Process Models

DBMS的process models定义了系统如何架构，以支持来自多用户应用的并发请求。

一个**worker**是DBMS中的一个组件，负责代表客户端执行任务并返回结果。

### Approach #1: Process per DBMS Worker

每个worker是一个分立的OS进程

* 依赖OS调度器
* 使用shared-memory存储全局数据结构
* 一个进程crash掉不影响整个系统
* Examples: IBM DB2, Postgres, Oracle

![Process per DBMS Worker](<../.gitbook/assets/image (17).png>)

### Approach #2: Process Pool

一个worker可以用进程池中任何空闲进程。

* 仍然依赖OS调度器和shared memory
* 对CPU缓存局部性不好
* Exampel: IBM DB2, Postgres(2015)

![Process Pool](<../.gitbook/assets/image (15).png>)

### Approach #3: Thread Per Worker

单进程，多worker线程

* DBMS自己管理调度
* 可以用也可以不用dispatcher线程
* 线程crash可能kill整个系统
* Example: IBM DB2, MSSQL, MySQL, Oracle(2014)

![Thread Per Worker](<../.gitbook/assets/image (10).png>)

多线程架构的优点：

* 更少的context switch开销
* 不用管理shared memory

thread per worker模型不意味着DBMS支持intra-query parallelism

### Scheduling

对于每个query plan，DBMS决定何处、何时、如何执行它：

* 它需要用多少个任务？
* 它需要用多少个CPU cores？
* 任务应该执行在哪个CPU core上？
* 任务输出应该放在哪？

_The DBMS **always** knows more than the OS._

## Execution Parallelism

Inter-Query: 不同的query并发执行。增加吞吐量，降低延迟。

Intra-Quey: 并行执行一个query。可以降低long-running query的延迟。

### Inter-Query Parallelism

允许多个query同时执行，提高整体性能。

如果query只读，那么几乎不需要query间的协作。

如果多个query需要同时更新数据库，那么就很难保证正确性了。会在15节深入研究这个问题。

### Intra-Query Parallelism

通过并行执行操作，提高单个query的性能。

可以把一组算子看成一个**生产者/消费者**范式。

每个算子都有并行的版本。既可以是多线程访问中心数据结构，也可以分区来divide work up。

#### Approach #1: Intra-Operator (Horizontal)

将算子分解成独立的**fragments**，对不同子集的数据执行同样的函数。

DBMS会插入一个**exchange**算子，来合并/分割来自children/parent算子的结果。exhange算子有三种：**Gather, Distribute, Repartition**

![Intra-Operator](<../.gitbook/assets/image (14).png>)

#### Approach #2: Inter-Operator (Vertical)

* 将 operators 串成 pipeline，将数据传输到下一阶段，一般无需等待前一步操作执行完毕
* worker同时执行算子的不同fragment。

也叫**pipeline parallelism**

![Inter-Operator](<../.gitbook/assets/image (16).png>)

此方法常用于流处理系统

#### Approach #3: Bushy Parallelism

经典hybrid方法

![Bushy Parallelism](<../.gitbook/assets/image (9).png>)

#### Observation

如果磁盘是主要的瓶颈，使用额外的进程/线程并行执行query并不会对提高性能有什么帮助。

相反，如果每个worker在磁盘的不同segments上工作，可能会导致性能更差。

## I/O Parallelism

将DBMS分裂到多个存储设备上：

* 多个磁盘，一个数据库
* 一个数据库，一个磁盘
* 一种关系，一个磁盘
* 把关系分裂到多个磁盘
* ...

### Multi-Disk Parallelism

通过配置OS/硬件，来把DBMS文件存储到多个存储设备上，如RAID Configuration。

这对于DBMS来说是透明的。



![](<../.gitbook/assets/image (19).png>)![](<../.gitbook/assets/image (18).png>)

### Database Partitioning

有些DBMS允许你指定单个数据库存储的磁盘位置

* buffer pool manager将一张页映射到一个磁盘位置

如果DBMS把每个数据库存储在分立的目录下，这在文件系统层面来说也是很容易实现的。

* 若事务能够更新多个数据库，DBMS的恢复日志文件可能仍然是shared。

### Partitioning

将单个logical table分裂成disjoint physical segments，分立存储。

理想情况下，partitioning对于应用来说应该是透明的：应用只会访问logical table，不用担心这张表在物理上是如何存储的。

#### Vertical Partitioning

垂直切分，必须存储tuple信息才能重建原始记录。

```sql
CREATE TABLE foo (
  attr1 INT,
  attr2 INT,
  attr3 INT,
  attr4 TEXT  
);
```

![](<../.gitbook/assets/image (1).png>)

#### Horizontal Partitioning

把表基于某种partitioning key分割成disjoint segements，比如Hash Partitioning, Range Partitioning, Predicate Partitioning

![](<../.gitbook/assets/image (13).png>)

## Conclusion

并行执行很重要，这也是为什么几乎每个主要的DBMS都会支持它。

然而，想要正确实现还是挺难的：

* 协作(coordination)带来的过度开销
* 调度
* 并发问题
* 资源竞争(contention)
