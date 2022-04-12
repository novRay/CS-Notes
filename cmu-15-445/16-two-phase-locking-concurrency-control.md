# #16: Two-Phase Locking Concurrency Control

上节课我们知道，可以用swapping或者依赖图的方法验证schdule是否是conflict serializable的。任何声称自己支持serializable isolation的DBMS都使用这种办法。

我们需要一种办法，在不提前知道schdule的前提下，确保所有执行的shedule是正确的（也就是serializable的）。解决方案是：用**lock**保护数据库对象

![](<../.gitbook/assets/image (6) (1).png>)

本节大纲：

* Lock Types
* Two-Phase Locking
* Deadlock Detection + Prevention
* Hierarchical Locking

## Lock Types

### Locks vs. Latches

|            | Locks                                | Latches       |   |
| ---------- | ------------------------------------ | ------------- | - |
| Separate   | 事务                                   | 线程            |   |
| Protect... | 数据库内容                                | in-memory数据结构 |   |
| During...  | 整个事务                                 | 临界区           |   |
| Modes...   | Shared, Exclusive, Update, Intention | Read, Write   |   |
| Deadlock   | 检测&解决                                | 避免            |   |
| ...by...   | Waits-for, Timeout, Aborts           | 编码规范          |   |
| Kept in... | Lock Manager                         | 受保护的数据结构      |   |

本节探讨的“锁”指的是lock

### Basic Lock Types

**S-LOCK**：用于读的共享锁

**X-LOCK**：用于写的排他锁

![](<../.gitbook/assets/image (8) (1) (1) (1).png>)

### Executing With Locks

* 事务请求锁（或升级锁）
* Lock manager准许(grant)或阻塞(block)请求
* 事务释放锁(release)
* Lock manager更新内部锁表
  * 跟踪什么事务持有什么锁、什么事务等待获得锁

一个shedule example如下：

![](<../.gitbook/assets/image (7) (1) (1).png>)

但这实际上并没有保证schedule的正确性，因为T1释放锁A后T2又对A写入，导致T1前后两次读A读到了不一样的值。

![](<../.gitbook/assets/image (17) (1).png>)

我们需要一种并发控制协议，用于确定事务是否可以即时访问数据库中的对象。二阶段锁（2PL）就是其中之一。

协议不需要提前知道事务要执行什么query

## Two-Phase Locking

_Phase #1: Growing_

* 每个事务向lock manager请求锁
* lock manager grant/deny 锁请求

_Phase #2: Shrinking_

* 允许事务释放之前获得的锁。不能获得新锁。

![](<../.gitbook/assets/image (20) (1).png>)

![](<../.gitbook/assets/image (4).png>)

Example:

![](<../.gitbook/assets/image (18) (1).png>)

2PL本身已经足够保证conflict serializable。它生成的依赖图是无环的。

但它受限于级联终止（**cascading aborts**）

如下图所示。当T1 abort，T2也必须abort。否则意味着T2读到了T1写入、但是abort了的数据。也就是说T1的数据"泄露"给了T2

![](<../.gitbook/assets/image (24) (1).png>)

### 2PL Obervation

* 可能有serializable，但不符合2PL的schedule
  * 锁的存在限制了并发
* 还是可能出现“脏读”
  * Solution：**Strong Strict 2PL(aka Rigorous 2PL)**
* 可能导致死锁
  * Solution：检测（**Detection**）或预防（**Prevention**）

### **Strong Strict Two-Phase Locking**

更严格的2PL，当且仅当在commit或abort时，释放掉所有的锁。

![](<../.gitbook/assets/image (13) (1).png>)

严格的2PL保证了：如果一个值被一个事务写过，那么其他事务就不允许读或写这个值，知道当前事务结束。

好处在：

* 不会引发级联终止
* 只要恢复被改动的tuple的原值，就可以undo被abort掉的事务。

#### Examples：

事务T1：从A转账100到B

事务T2：计算AB账户余额总和

非2PL，T2读了还没加上100的B，导致出错。

![](<../.gitbook/assets/image (3) (1).png>)

2PL，结果正确。当T2想读B时，由于已经进入了shrinking阶段，请求被deny。直到T1把所有的锁释放，才获得了读B的锁。

![](<../.gitbook/assets/image (23) (1) (1).png>)

严格的2PL，T2对A的读锁请求要等到整个T1完成了才被grant。结果同样正确，但并发性稍差。

![](<../.gitbook/assets/image (15) (1) (1) (1) (1).png>)

把no cascading aborts加入universe of schedule：

![](<../.gitbook/assets/image (25) (1).png>)

## Deadlock

当事务间循环等待对方释放锁时，就会出现死锁。

处理死锁有两个办法：

* Approach #1: Deadlock Detection
* Approach #2: Deadlock Prevention

## Deadlock Detection

创建waits-for graph。周期性检查是否存在环，决定如何打破环。

![](<../.gitbook/assets/image (5) (1) (1).png>)

死锁处理方案：挑选一个victim事务，rollback

victim事务可以重启，也可以终止（common）。

需要对死锁检验的频率做一个trade-off

#### Victim Selection

考虑的变量：

* age（最小的时间戳）
* progress（最少/最多执行的query）
* 已经上锁的数量
* 已经回滚的数量

需要考虑事务已经被重启的次数，以防饥饿。

#### Rollback Length

选好victim后，DBMS可以决定应该整个回滚，还是部分回滚。

### Deadlock Prevention

当一个事务尝试获取一把被另一个事务持有的锁，DBMS杀死其中一个来防止死锁。

这种方案不需要waits-for graph或检测算法。



基于事务的时间戳，赋予每个事务priority。更老的时间戳=更高的优先权

* Wait-Die("Old Waits for Young")
  * 老事务要年轻事务的锁，老事务等年轻事务。
  * 年轻事务要老事务的锁，年轻事务自杀。
* Wound-Wait("Young Waits for Old")
  * 老事务要年轻事务的锁，年轻事务自杀，把锁给老事务。
  * 年轻事务要老事务的锁，年轻事务等老事务。

![](<../.gitbook/assets/image (12) (1).png>)

**为什么这些方案能保证没有死锁？**

因为等待锁只有单条边

**当一个事务重启，它的priority是？**

原来的时间戳。避免饥饿

#### Obersation

之前的例子中，数据库对象到锁的映射是一对一的。

如果一个事务想要更新one billion个tuple，就得获得one billion个lock。

获得lock要比latch的成本更昂贵

## Hierarchical Locking

当事务请求lock，DBMS可决定lock的粒度（granularity）：

* Attribute? Tuple? Page? Table?

理想情况下，DBMS应该获得事务所需的最少lock数

需要在parallelism和overhead间作trade-off：

* 更少的lock，更大的粒度 vs. 更多的锁，更小的粒度

![](<../.gitbook/assets/image (16) (1).png>)

### Intention Locks

意向锁允许给higher-level的结点上共享锁或者排他锁，不需要检查所有下级的结点。

如果结点被上了意向锁，说明某些事务正在给lower-level的结点上上显式锁。

* Intention-Shared(IS)
  * 表明某些下层结点上了共享锁
* Intention-Exclusive(IX)
  * 表明某些下层结点上了排他锁
* Shared+Intention-Exclusive(SIX)
  * 对整个下层结点上共享锁，但某些结点上了排他锁

![](<../.gitbook/assets/image (14) (1).png>)

Hierarchical locks在实践中很有用，让事务只需要更少的锁。

意向锁有助于提高并发性。

#### Lock Escalation

当有太多low-level的lock，会动态地请求粗粒度的lock。从而降低lock manager处理的请求。

#### Locking In Practice

事务通常不需要手动上锁。显式锁在对数据库作重大改动时有用。

![](<../.gitbook/assets/image (9) (1) (1).png>)

![](<../.gitbook/assets/image (11) (1) (1).png>)
