# #17: Timestamp Ordering Concurrency Control

上节课的2PL是一种**悲观**的并发控制协议。本节将介绍一种乐观的协议——**Timestamp Ordering(T/O)**：使用时间戳决定事务可串行化顺序。

如果$$TS(T_i)<TS(T_j)$$，那么DBMS必须保证执行的schedule等价于$$T_i$$先执行、$$T_j$$后执行的串行顺序。

每个事务递增地被赋予一个唯一、固定的时间戳。有许多实现策略：

* 系统时钟
* 逻辑计数器
* 混合

本章大纲：

* Basic Timestamp Ordering (T/O) Protocol
* Optimistic Concurrency Control
* Isolation Levels

## Basic T/O

事务的读写不需要lock，每条数据X携带两个标记:

* W-TS(X)：最后一次写X发生的时间戳
* R-TS(X)：最后一次读X发生的时间戳

每次操作都会检查时间戳，如果当前事物尝试访问一个“来自未来”的数据，则终止或重启。

#### Basic T/O - Reads

若TS(Ti) < W-TS(X)，abort Ti并重启,赋予其新的时间戳

否则：

* 允许Ti读X
* 更新R-TS(X)=max(R-TS(X), TS(Ti))
* copy一份X，保证Ti可重复读

#### Basic T/O - Writes

如果TS(Ti) < R-TS(X)或TS(Ti) < W-TS(X)，abort并重启Ti

否则:

* 允许Ti写X，更新W-TS(X)
* copy一份X，保证可重复读

#### Example 1

![](<../.gitbook/assets/image (15) (1) (1) (1).png>)

1. T1读B，1 ≥ W-TS(B)=0，可读，R-TS(B)改为1
2. T2读B，2 ≥ W-TS(B)=1，可读，R-TS(B)改为2
3. T2写B，2 ≥ W-TS(B)=0且2 ≥ R-TS(B)=2，可写，W-TS(B)改为2
4. T1读A，1 ≥ W-TS(A)=0，可读，R-TS(A)改为1
5. T2读A，2 ≥ W-TS(A)=0，可读，R-TS(A)改为2
6. T1读A，1 ≥ W-TS(A)=0，可读，R-TS(A)仍为2
7. T2写A，2 ≥ W-TS(A)=0，可写，W-TS(B)改为2

#### Example 2

![](<../.gitbook/assets/image (21) (1).png>)

1. T1读A，1 ≥ W-TS(A)=0，可读，R-TS(A)改为1
2. T2写A，2 ≥ W-TS(A)=0，且2 ≥ R-TS(A)=1，可写，W-TS(A)=2
3. T1写A，1 < W-TS(A)=2，不可写。T1 abort

但仔细观察我们可以发现，如果不让T1写，也不更新W-TS(A)，那么T1和T2都可以正常提交，结果与T1、T2串行等价。

引入一个**Thomas Write Rule**：

如果TS(Ti) < W-TS(X)：忽略当前写操作，允许事务继续执行，不必终止。

这种办法违背了Ti的时间戳顺序，但提高了一定的并发性。

#### Basic T/O Summary

如果不使用Thoma Write Rule，那么生成的schedule是conflict serializable的

* 无死锁，因为没有事务等待
* 可能因为短事务总是冲突，导致长事务的饥饿
* 可能导致schedule不可恢复

#### Recoverable Schedules

如果一个schedule能保证：对于每个事务而言，如果它读取了其他事务的改动，那么那些其他事务必须比自己先提交，那么这个schedule就是**可恢复的**。否则，DBMS无法保证事务读取的数据，会在崩溃恢复后恢复。

下图就是一个不可恢复的例子。T2读了T1写入的A，然而T1在T2提交后abort，T2依赖T1写入的数据A并没有真实写入

![](<../.gitbook/assets/image (8) (1) (1).png>)

#### Performance Issues

basci T/O有一些性能问题：

* 复制数据、更新时间戳带来的高额开销
* 长事务饥饿

#### Observation

如果事务间冲突罕见，且绝大多数事务都比较短，那么不用lock的确能降低很多开销。

一种更好的办法是优化非冲突的case

## Optimistic Concurrency Control

DBMS给每个事务创建一个工作区。

* 读操作读取原表数据，复制到工作区
* 修改操作作用到工作区

当事务提交，DBMS比较工作区的write set，查看是否存在冲突。

#### OCC Phases

* Read Phase：
  * 读写数据。所有访问的数据都复制进工作区，保证可重复读。
* Validation Phase：
  * 事务COMMIT时，检查是否与其他事务冲突（RW,WW），保证只存在older->younger的单项边。
  * 有backward和forward两种。
* Write Phase：
  * 将工作区改动写回数据库，使其对其他事务可见。
  * 需要上write latch，保证一次只能由一个事务在写。

Example:

![](<../.gitbook/assets/image (5) (1).png>)

1. T1开始READ阶段
2. T1读A，向工作区写入（A,123,0）
3. T2开始READ阶段
4. T2读A，向工作区写入（A,123,0）
5. T2开始VALIDATE，得到时间戳1.
6. T2与T1工作区无冲突，T2 WRITE（不需要操作）
7. T2 COMMIT
8. T1写A，将原先的记录修改为（A,456,∞）（因为尚不知道自己的时间戳，所以是∞）
9. T1 VALIDATE，得到时间戳2.
10. T1 WRITE, COMMIT

#### Backward Validation

检查已提交的事务，如果有冲突，自杀

![](<../.gitbook/assets/image (6).png>)

#### Forward Validation

检查“未来”的事务

![](<../.gitbook/assets/image (2).png>)

事务会在validation阶段开始时被赋予一个时间戳。

有如下三种情形：

#### Step 1：

Ti整个三阶段都在Tj开始前完成

![](<../.gitbook/assets/image (23) (1).png>)

#### Step 2：

Ti在Tj的Write阶段开始前完成，并且$$WriteSet(T_j)∩ReadSet(T_j)=Ø$$

下图的情形中，T1先validate，比T2早。而未来的T2读取的还是原表A的值（T1工作区还未写入），因此T1必须abort

![](<../.gitbook/assets/image (11) (1).png>)

而如果T2先validate，则T1是未来的事务。T2正确读取了T1发生前原表中的数据。两者都可以安全的提交。

![](<../.gitbook/assets/image (22) (1) (1).png>)

#### Step 3：

Ti的Read阶段的完成，早于TjRead阶段的完成。

并且：

$$WriteSet(T_i) ∩ ReadSet(T_j) = Ø$$

$$WriteSet(T_i) ∩ WriteSet(T_j) = Ø$$

下图中，当T1 validate时，T1的write和T2的write set和read set都没有交集，可以安全写入数据库，提交。

当T2读A，读到的是T1写入的A，符合逻辑顺序，因此最后也可以安全提交。

![](<../.gitbook/assets/image (7) (1).png>)

#### OCC - Obersavation

当冲突数量少时，OCC效果很好。

* 最理想情况是所有事务时只读的
* 事务访问的数据集没有交集。

**怎样的情况冲突发生的概率低？**

数据库很大，workload不倾斜。

#### **OCC - Performance Issues**

* 复制数据的高额开销
* Validate/Write阶段的瓶颈
* 如果abort，比2PL更浪费。因为OCC的abort只会发生在事务执行之后。

### The Phantom Problem

之前提到的事务只涉及读和更新已经存在的数据。如果我们的数据支持插入、删除操作，就可能会导致“幻读”的问题：

![](<../.gitbook/assets/image (27) (1).png>)

发生的原因：事务T1只锁住了已经存在的记录。

解决方案：

#### Approach #1: Re-Execute Scans

DBMS维护一个scan set，记录WHERE语句筛出的记录。事务COMMIT时，再扫描一遍，看scan set是否相同。

#### Approach #2: Predicate Locking

给SELECT的WHERE上共享锁，给UPDATE, INSERT, DELETE的WHERE上排他锁。

这个方法只在Hyper系统中实现过。

![](<../.gitbook/assets/image (7).png>)

#### Approach #3: Index Locking

以上文中的例子为例，如果status attribute上有索引，则锁住所有包含status='lit'的索引页。

如果没有索引，则：

* 给table的每个页上锁，防止其他记录的status被改为lit
* 给table本身上锁，防止添加或删除status=‘lit'

## Isolation Levels

serializability很有用，让程序员可以不用关注并发问题。但是强制实现它会导致太低的并发度和性能，因此我们希望一种更低层次的一致性，以此提高scalability。

更弱的隔离级别会把事务一定程度上暴露给其他事务，提高并发性，代价是会出现一些问题：

* 脏读 Dirty Reads
* 不可重复度 Unrepeatable Reads
* 幻读 Phantom Reads

隔离级别由强到弱：\


![](<../.gitbook/assets/image (26) (1).png>)

* SERIALIZABLE：首先获得所有锁；加上索引锁；加上严格的2PL
* REPEATABLE READS：同上，但没有索引锁
* READ COMMITTED：同上，但共享锁会第一时间立即释放
* READ UNCOMMITTED：同上，但允许脏读（没有共享锁）

常见DBMS支持的隔离级别：

![](<../.gitbook/assets/image (15) (1) (1).png>)
