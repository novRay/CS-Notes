# #20: Crash Recovery Algorithms

上节课提到，恢复算法有两部分：

* 事务处理过程中出现的故障的恢复措施
* 故障恢复后保证数据库原子性、一致性和持久性的措施

本节将介绍第二部分。

## ARIES

**A**lgorithms for **R**ecovery and **I**solation **E**xploiting **S**emantics，1990年代IBM为DB2研发的故障恢复和隔离算法。Main ideas如下：

* **Write-Ahead Logging**:
  * 任何改动在写回磁盘前，会先写入记录并入盘
  * 必须使用STEAL + NO-FORCE的buffer pool policies
* **Repeating History During Redo**:
  * 重启时，按照日志内容重放操作，将数据库恢复到与故障前完全一致
* **Logging Changes During Undo**:
  * 将undo操作记入日志，保证重复的故障不会导致重复的undo

本节大纲：

* Log Sequence Numbers
* Normal Commit & Abort Operations
* Fuzzy Checkpointing
* Recovery Algorithm

## Log Sequence Numbers

我们需要给上节课讲的日志记录添加额外的信息：每条记录包括了一个全局唯一的**log sequence number(LSN)**，系统中各种部分都需要用到LSN：

| Name         | Where  | Definition          |
| ------------ | ------ | ------------------- |
| flushedLSN   | Memory | 磁盘中最后一条LSN          |
| pageLSN      | page X | 对page X的最新更新        |
| recLSN       | page X | 自上次刷盘以来对page X最老的更新 |
| lastLSN      | Ti     | 事务Ti的最新记录           |
| MasterRecord | Disk   | 最新检查点的LSN           |

每张data page包含一个pageLSN（最新更新记录）

系统跟踪flushedLSN（最新刷盘记录）

page X写入磁盘前，必须满足$$pageLSN_x ≤ flushedLSN$$

![](<../.gitbook/assets/image (5).png>)

在下图中，pageLSN是019，而flushedLSN是016，因此不能把buffer pool中的页刷盘。必须先把日志刷盘。

![](<../.gitbook/assets/image (38).png>)

所有日志记录都有LSN。每当事务修改页中数据时，更新pageLSN；每当DBMS将WAL写入磁盘，更新flushedLSN

## Normal Commit & Abort Operations

### Normal Execution

事务调用一系列读和写后，进行commit或abort

本节课做出如下假设：

* 所有日志记录放得进一张页
* 磁盘写是原子的
* tuple只有一个版本，遵循strict 2PL
* **STEAL + NO-FORCE** with WAL

### Transaction Commit

事务提交时，写一条**COMMIT**日志记录。

到COMMIT为止的所有事务记录刷入磁盘

* 刷日志是顺序的(sequential)，同步的(synchronous)
* 每张日志页有多条记录

当提交成功，写一条**TXN-END**日志记录

* 不需要立刻刷入磁盘

事务准备提交：

![](<../.gitbook/assets/image (8).png>)

012\~015刷盘，更新flushedLSN；一段时间后记TXN-END，这时可以把内存里flushedLSN之前的记录裁剪掉。

![](<../.gitbook/assets/image (15).png>)

### Transaction Abort

abort是ARISE undo操作的一种特殊情况：回滚单个事务

需要给日志记录添加额外的field，以支持事务回滚。

* **prevLSN**：当前事务内上一条LSN
* 维护类似一个双向链表，方便walk through事务的记录

![](<../.gitbook/assets/image (28).png>)

如上图所示。当T4需要abort，只需跟着prevLSN往上回滚（AfterValue -> BeforeValue）直到nil即可。但我们需要记录具体undo了什么（见下文），来避免回滚时再次故障，导致重复执行一些操作。确定回滚完毕后，再写条TXN-END记录。

### Compensation Log Records

**CLR**描述了undo过程中执行的操作。

它包括所有update日志记录的field，外加一个undoNext指针（下一个要被undo的LSN）

![](<../.gitbook/assets/image (1) (1).png>)

### Abort Algorithm

首先，写一条ABORT记录

然后，倒叙回放事务的更新记录

* 写一项CLR到日志
* 恢复旧值

最后，写一条TXN-END记录

注意：**CLR**永远不会被undo

## Fuzzy Checkpointing

### Non-Fuzzy Checkpoints

非模糊检查点：DBMS会暂停所有事务，以保证一个一致性的快照。

* 暂停任何新事物开始
* 等待所有执行中的活跃事务完成
* 把脏页刷进磁盘

这种检查点对于恢复很容易，但对运行时性能很糟糕。

### Slightly Better Checkpoints

暂停写事务，阻止获得表和索引页的write latch。不需要等待所有事务结束。

必须在checkpoint开始时记录两个内部状态：

* 活跃事务表 **Active Transaction Table(ATT)**
  * txnId, status, lastLSN
  * status: **R**unning/**C**ommitting/Candidate for **U**ndo
* 脏页表 **Dirty Page Table(DPT)**
  * recLSN：第一个导致页变脏的记录LSN

![](<../.gitbook/assets/image (37).png>)

如上图所示。在第一个检查点，T2还在活跃(R)，P22是脏页。在第二个检查点，T2和T3在活跃(T2:C, T3:R)，P11和P33是脏页。

这还不够理想，因为DBMS必须要暂停事务。

### Fuzzy Checkpoints

DBMS允许在设检查点时让活跃事务继续运行。

需要新的日志记录来跟踪检查点边界：

* **CHECKPOINT-BEGIN**：表示检查点开始
* **CHECKPOINT-END**：包含**ATT + DPT**

只有当检查点成功完成，CHECKPOINT-BEGIN的LSN才会写到数据库的MasterRecord项中。

任何在检查点启动之后开始的事务不会写到CHECKPOINT-END的ATT当中。

![](../.gitbook/assets/image.png)

## Recovery Algorithm

经过以上的准备工作，现在我们可以梳理ARIES恢复算法的整个过程了。

ARIES恢复算法的三阶段：

* Phase #1 - Analysis：分析。从最新的MasterRecord读取WAL，找出buffer pool中的脏页，以及崩溃时活跃的事务。
* Phase #2 –Redo：在日志的某个合适的点开始，重复所有操作。（即便事务将会abort）
* Phase #3 –Undo：复原崩溃前未提交事务的所有操作。

![](<../.gitbook/assets/image (22).png>)

### Analysis Phase

从日志的最新一个成功的检查点开始顺序扫描，如果找到**TXN-END**，将对应事务从**ATT**删除。

对于其他的记录：

* 事务放入**ATT**，标记状态**UNDO**
* 若遇到事务提交，改变状态为**COMMIT**
* 对于**UPDATE**记录：如果页P不在**DPT**，则放入**DPT**，设置它的**recLSN=LSN**

分析结束时，ATT中的事务在崩溃时仍然活跃；DPT中的脏页还未写入磁盘。

![分析示范。分析结束时，活跃事务T97未提交，脏页P33和P20未刷盘。](<../.gitbook/assets/image (13).png>)

### Redo Phase

Redo的目的是重复历史，以重建崩溃时的状态。（重放所有更新，包括中止事务，重做**CLRs**）。有一些办法可以让DBMS避免不必要的读写，但本节课不做介绍。

从**DPT**中最小的**recLSN**开始顺序扫描。重做每条更新日志或**CLR**，除非：

* 被影响的页不在**DPT**中，或
* 被影响的页在**DPT**中，但记录的LSN比页的**recLSN**小（只重做最后一次更新）

重做时，需要：

* 重新执行日志操作
* 设**pageLSN**为日志记录的**LSN**
* 不再新增日志，也不强制刷盘！

Redo阶段结束，为所有状态为**C**的事务写一条**TXN-END**日志记录，并将其从**ATT**中删除。

### Undo Phase

撤销崩溃时还在活跃的事务的所有操作（分析阶段结束后，**ATT**中状态为**U**的事务）。

利用**lastLSN**，倒序执行。

每一次撤销，写一条**CLR**。



完整的执行范例见课程[slides](https://15445.courses.cs.cmu.edu/fall2021/slides/20-recovery.pdf)的118-136页。

### Additional Crash Issues

> 如果DBMS在分析阶段崩溃怎么办？

无所谓，再次恢复就行。

> 如果DBMS在Redo阶段崩溃怎么办？

也无所谓，再次重做就行。

> 如何在Redo阶段提高性能？

假设不会再次崩溃的情况下，可以将所有改动在后台**异步**地刷入磁盘。

> 如何在Undo阶段提高性能?

* Lazily rollback：新事物访问数据页时才回滚
* 重写应用程序，避免长运行事务

## Conlusion

ARIES的main ideas回顾：

* WAL with **STEAL + NO-FORCE**
* Fuzzy Checkpoints（脏页ID的快照）
* 从最早的脏页开始redo一切
* undo从未提交的事务
* undo时写**CLRs**，避免重启时再次故障。

Log Sequence Numbers：

* LSNs标明日志记录；每个事务内通过prevLSN构建双向链表。
* pageLSN允许数据页和日志记录作比较。



至此，单节点DBMS的构建介绍完毕。
