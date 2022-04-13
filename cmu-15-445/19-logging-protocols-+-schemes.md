# #19: Logging Protocols + Schemes

数据库在运行过程中可能遇到各种故障（例如断电），因此我们需要一种恢复算法，来保证数据库的一致性、事务的原子性和面对意外的持久性。

恢复算法有两部分：

* 事务处理过程中出现的故障的恢复措施（本节内容）
* 故障恢复后保证数据库原子性、一致性和持久性的措施

本节大纲：

* Failure Classification
* Buffer Pool Policies
* Shadow Paging
* Write-Ahead Log
* Logging Schemes
* Checkpoints

## Failure Classification

先介绍一下存储类型：

* Volatile Storage
  * 易失性存储，数据在断电后或程序结束后不保留
  * DRAM, SRAM
* Non-volatile Storage
  * 非易失性存储，数据在断电后或程序结束后保留
  * HDD, SDD
* Stable Storage
  * 一种**不存在**的非易失性存储，能在任何故障下存货。

故障分类：

#### Transaction Failures

* **Logical Errors：**由于内部error条件，事务无法完成（如数据一致性约束）
* **Internal State Errors：**由于error条件，DBMS必须终止活跃的事务（如死锁）

#### System Failures

* **Software Failure**：OS或DBMS实现的问题（如未捕获的divide-by-zero exception
* **Hardware Failure：**
  * 计算机崩溃（插头被拔了）
  * Fail-stop Assumption：假设非易失性存储内容不会因系统崩溃而损坏

#### Storage Media Failure

* Non-Repairable Hardware Failure：
  * 磁盘故障，毁掉了全部或者部分非易失性存储内容（如磁头坏了）
  * 假设破坏时可被检测的（例如，磁盘控制器用checksum）

没有DBMS能恢复这个！数据库必须从存档的版本中恢复。

#### Obervation

数据库主要存在非易失性设备，但它的速度要比易失性设备慢多了。易失性内存访问速度更快：

* 拷贝目标记录到内存
* 在内存中执行写操作
* 将脏记录写回磁盘

DBMS需要保证：

* 一旦DBMS说事务提交，那么所有改动必须是持久的。
* 如果事务被abort，任何部分更改不能存在。

## Buffer Pool Policies

故障恢复主要有：

* Undo：去除未完成或中止事务的效果
* Redo：恢复已提交事务的效果

DBMS如何支持这些功能，取决于如何管理buffer pool

![](<../.gitbook/assets/image (23).png>)

如上图所示，T1读A，将包含ABC数据的页读入内存，然后修改了A；T2修改了B。现在T2要提交，有两点设计考量：

* 要不要强制把T2的改动页刷进磁盘？
* 要不要把还没提交的T1的改动也一起刷进磁盘？

若之后T1要回滚，应该做什么？

#### Steal Policy

DBMS是否允许未提交事务覆盖易失性设备中最近提交的值？

**STEAL**：允许；**NO-STEAL**：不允许

#### Force Policy

DBMS是否需要在事务可以提交前，将所有更新反映在非易失性设备上？

**FORCE**：需要；**NO-FORCR**：不需要

基于以上两种policy，可以实现四种组合。**NO-STEAL+FORCE**的例子如下：

![](<../.gitbook/assets/image (11).png>)

NO-STEAL：T1对A的改动不能刷进磁盘，因此复制一张页并把A置为原值，只刷入B。

FORCE：T2提交时，改动必须刷入磁盘。

若之后需要回滚T1，只需要简单地把内存里那张副本页删除即可，不需要真的“回滚”。

NO-STEAL+FORCE是最容易实现的：

* 不需要undo改动，因为中止的事务没有写入磁盘。
* 不需要redo改动，因为提交的事务肯定已经在提交时刷进磁盘了。（假设是原子的硬件写）

之前的例子不支持write set超过内存可用大小空间。

## Shadow Paging

维护两个单独的数据库副本：

* **Master**：只保存提交了的事务的改动
* **Shadow**：临时数据库，保存未提交事务的改动

事务只对shadow copy更新。当事务提交，原子地将shadow切换成新的master

buffer pool policy：NO-STEAL+FORCE

Example：

事务开始前，只有一份master表，DB Root指向master

![](<../.gitbook/assets/image (24).png>)

事务开始，复制一份shadow

![](<../.gitbook/assets/image (37) (1).png>)

在shadow上作改动。T1提交，将Root指针指向shadow，把内存中原来的master给删掉。

![](<../.gitbook/assets/image (1) (1) (1).png>)

#### **Shadow Paging - Undo/Redo**

**Undo**：删除shadow pages。只保留master和DB Root pointer

**Redo**：不需要

#### **Shadow Paging - Disadvantages**

* 复制整张表开销非常昂贵，优化措施：
  * 用类似B+树的页表结构。
  * 不需要复制整棵树，只复制导向被更新叶结点的路径
* 提交时开销高：
  * 需要刷脏页，页表和root
  * 数据碎片化
  * 需要垃圾回收
  * 只支持一次一个写事务或一批事务

2010年之前。SQLite使用一种改进的shadow paging：要改动页之前，先将页原来的样子复制到journal file。当故障需要恢复，将journal file读入内存，将磁盘中的页覆盖，实现undo。

![](<../.gitbook/assets/image (36).png>)

Shadow paging需要DBMS对随机不连续的页执行写操作。我们需要一种方法让DBMS实现顺序写。

## Write-Ahead Log

维护一个日志文件，包含事务对数据库的修改。

* 日志存在stable storage上
* 日志包含足够的信息，来执行必要的undo和redo

DBMS必须在刷数据前把日志文件先刷入磁盘。

buffer pool policy:**STEAL+NO-FORCE**

### **WAL Protocal**

DBMS把事务的日志记录暂存(stage)在易失性设备（buffer pool）。

脏页相关的日志必须在页被刷进磁盘前，先刷进磁盘。

当日志写入stable storage，事务才视为提交成功。



事务开始，写一条**\<BEGIN>**记录

事务结束：

* 写一条**\<COMMIT>**记录
* 保证所有的日志记录刷盘，再向应用程序返回acknowledgement。

每条日志包含的信息：

* Transaction Id
* Object Id
* Before Value(UNDO)
* After Value(REDO)

Example：

![](<../.gitbook/assets/image (32).png>)

如果突然故障，内存清空。根据磁盘上的日志就可以恢复。

![](<../.gitbook/assets/image (20).png>)

### WAL - Implementation

#### Group Commit

一个事务要提交时先卡住，让其他事务运行，然后打包多个日志一起刷盘，可以均摊开销。但问题是用户可能会互相等待。

Example：

执行一些操作后，某个日志页满了，先将它刷盘

![](<../.gitbook/assets/image (27).png>)

等待一段时间后，开始打包提交。

![](<../.gitbook/assets/image (12).png>)

![](<../.gitbook/assets/image (16).png>)

#### Buffer Pool Policies

几乎所有的DBMS用的是NO-FORCE+STEAL，拥有最好的运行时性能。但恢复性能稍慢。

![](<../.gitbook/assets/image (3).png>)

## Logging Schemes

#### **Physical Logging**

* 记录具体改动地址
* Example：`git diff`

#### **Logical Logging**

* 记录high-level操作
* 不局限于单一页
* Example：UPDATE, DELETE, INSERT

需要写入日志的数据量更少，但并发事务会使它难以实现：

* 难以确定数据库哪部分在崩溃前被修改了
* 花更多时间恢复，因为必须重新执行每个事务

#### Physiological Logging

经典hybrid approach。日志记录目标页，但不指定页的组成（不指定offset）：

* 用slot number查找指定tuple
* 日志写入磁盘后，允许DBMS重新组织（reorganize）页

这种是最流行的方法。

![](<../.gitbook/assets/image (34).png>)

## Checkpoints

WAL会一直增长，如果崩溃后DBMS要重放（replay）整个日志，会花费非常长的时间。

DBMS会周期性地设一个检查点（checkpoint），将所有buffer（包括所有日志文件和修改过的block）都刷进盘。写一条**\<CHECKPOINT>**记录

![](<../.gitbook/assets/image (18).png>)

以上图为例。T1在检查点前已经提交，因此忽视它。

T2和T3在检查点后提交，所以：

* redo T2，因为T2在检查点后，crash前提交
* undo T3，因为T3在crash前未提交。

#### Challenges

DBMS必须暂停所有事务来设一个checkpoint，以保证得到一个一致性的快照。

扫描整个日志来查找未提交的事务很费时间。

不清楚多久应该设一个checkpoint

#### Frequency

太频繁设checkpoint导致运行时性能下降。（花太多时间刷buffer）

但间隔太长也不好：（checkpoint太大太慢；恢复时间更长）

## Conclusion

Write-Ahead Logging几乎是最好的用于处理易失性存储丢失的方案。

它使用incremental updates(STEAL + NO-FORCE)

恢复时：undo未提交 + redo已提交
