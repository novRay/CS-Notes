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

![](<../.gitbook/assets/image (1).png>)

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

****
