# #18: Multi-Version Concurrency Control

## Introduction

对于数据库当中的每一个逻辑对象，DBMS维护着它的多个物理版本。

* 当事务写一个对象，DBMS给它创建一个新版本
* 当事务读一个对象，DBMS读取事务开始时最新的版本

#### MVCC历史

首次提出是在1978年MIT的一篇PhD论文。首次实现是在80年代初，由DEC的Rdb/VMS和InterBase。作者是Jim Starkey，NuoDB的合伙创始人。DEC Rdb/VMS就是现在的Oracle Rdb，InterBase被开源成Firebird.

#### MVCC

基本思想：

> Writers do not block readers. 读者不阻塞写者。
>
> Readers do not block writers. 写者不阻塞读者

只读的事务可以读取数据库某一时刻一致性的**快照（snapshot）**，而不需要lock。（使用时间戳来决定visibility）

易于支持time-travel查询。

#### Example：

T1写A，创建一个A的新版本A1，标记Begin为1，并将A0版本的End标记为1。

![](<../.gitbook/assets/image (30).png>)

T2读A，由于T1还没提交，T2只能读取A0版本

![](<../.gitbook/assets/image (28).png>)

T2写A，由于存在一个还未提交的新版本A1，T2需要等待T1提交

![](<../.gitbook/assets/image (31).png>)

T1读A，读取新版本A1

![](<../.gitbook/assets/image (26).png>)

T1提交，T2创建新版本A2

![](<../.gitbook/assets/image (21).png>)

由上面的例子可以看出，MVCC本身并不能实现serializability，还需要结合2PL或OCC。

MVCC不仅仅是一种并发控制协议。它深刻影响了DBMS如何管理事务和数据库。

![](<../.gitbook/assets/image (17).png>)

## MVCC Design Decisions

设计决策：

* Concurrency Control Protocol
* Version Storage
* Garbage Collection
* Index Management
* Deletes

### Concurrency Control Protocol

* Approach #1: Timestamp Ordering
  * 时间戳顺序决定串行顺序
* Approach #2: Optimistic Concurrency Control
  * 上节提到的三阶段协议
  * 使用private workspace存放新版本
* Approach #3: Two-Phase Locking
  * 读写逻辑tuple前，给物理版本上lock

### Version Storage

给逻辑tuple创建”版本链“（**version chain**）。

* 允许DBMS在运行时找到一个对特定事务可见的版本。
* 索引永远指向链头。

存储范式：

#### Append-Only Storage

逻辑tuple的所有物理版本存在同一张表里。不同的tuple的版本是混在一块的。

更新时只要在表的空余位置附加一条新版本的tuple

![](<../.gitbook/assets/image (15).png>)

版本链的顺序有两种：

* **Oldest-to-Newest (O2N)**
  * 在链表末尾附加新版本
  * 必须遍历链表来查找
* Newest-to-Oldest (N2O)
  * 必须给每个新版本更新索引指针
  * 不需要遍历链表查找

#### Time-Travel Storage

单独用一张表存储过去的版本。

每次更新，把当前版本复制到time-travel表中，并更新指针。

![](<../.gitbook/assets/image (22).png>)

![](../.gitbook/assets/image.png)

#### Delta Storage

对time-travel storage的优化，只存变化信息。

![](<../.gitbook/assets/image (9).png>)

### Garbage Collection

DBMS每隔一段时间需要删除一些可回收的（reclaimable）物理版本。可回收的定义是：

* 任何活跃的事务都不能“看见”这个版本
* 这个版本是被一个已经被abort掉的事务创建的

两点额外的设计决策：

* 如何查询过期的版本？
* 如何决定什么时候回收是安全的？

#### Tuple-level

直接检查旧版本的tuple

有background vacuuming和cooperative cleaning两种：

**Background Vacuuming：**

用专门的线程周期性地扫描表，查找可回收版本。可用于任何storage

![](<../.gitbook/assets/image (35).png>)

优化：可以用一个dirty block，标记改动过的页。扫描时只需要扫描这些脏页。

![](<../.gitbook/assets/image (10).png>)

**Cooperative Cleaning：**

让某个工作线程在遍历版本链时识别可回收版本。只用于O2N

![](<../.gitbook/assets/image (13).png>)

#### Transaction-Level GC

每个事务记录自己的read/write set，执行过程中记录下老版本。DBMS决定何时将所有已完成事务产生的旧版本回收。需要多线程工作来提速。

如下图，事务更新A和更新B时记录下旧版本A2，B6

![](<../.gitbook/assets/image (14).png>)

DBMS回收掉TS<10的版本，则A2，B6被回收

![](<../.gitbook/assets/image (38).png>)

### Index Management

主键索引指向版本链头

* tuple更新时，系统创建新版本，则需要更新主键索引
* 如果事务更新tuple的主键，则需要先DELETE后INSERT

辅助索引就比较复杂了：

* Approach #1: Logical Pointers，逻辑指针，用间接层存储主键或tuple id，需要回表查。
* Approach #2: Physical Pointers，物理指针，直接指向版本链头的物理地址

使用物理指针，辅助索引都指向了链表头。如果更新版本，需要移动所有指针。

![](<../.gitbook/assets/image (25).png>)

使用逻辑指针，让辅助索引指针指向主键索引。

![](<../.gitbook/assets/image (33).png>)

指向中间表（tuple id），每次更新不需要更新所有辅助索引。空间换时间。

![](<../.gitbook/assets/image (29).png>)

#### MVCC Indexs

MVCC DBMS通常不存储索引的版本信息

* 一个例外是Index-organized tables（e.g., MySQL）

同一个key可能指向在不同快照中的不同的逻辑tuple

**MVCC重复键问题：**

如下图所示，线程2更新了A，插入新版本A2；又删除了A，删除版本A2，提交。

此时线程3要插入A，对线程3来说A已经不存在了（因为线程2的删除已提交），因此又插入了版本A1。但现在由于线程1的A1版本还未提交，导致出现了两个A1。

若此时线程1想要再次读A，应该读哪个版本？

![](<../.gitbook/assets/image (19).png>)

**MVCC Indexes**

每个索引的底层数据结构必须支持存储非唯一键。

用额外的逻辑约束插入主键/唯一索引（原子性的检查key是否存在，然后插入）

工作线程可能会得到多个满足条件的条目（如上例），必须跟踪指针找到一个合适的物理版本（见下文）。

### MVCC Deletes

DBMS只有当一个**逻辑上**被删除的tuple的所有版本不可见是，才能把它从**物理上**删除。

需要一种办法标识tuple已经在某时被逻辑上删除了：

#### Approach #1: Deleted Flag

* 维护一个flag，表示这个logical tuple已经被删除了
* 可以是tuple header，可以是一个单独的列

#### Approach #2: Tombstone Tuple

* 创建一个空的物理版本，表示一个logical tuple及其以前的版本被删除
* 在版本链用一个bit pattern标识墓碑，减少存储空间。

### MVCC Implementations

![](<../.gitbook/assets/image (8).png>)

## Conclusion

MVCC被各种DBMS广泛使用，甚至是不支持多语句事务的系统（e.g., NoSQL）
