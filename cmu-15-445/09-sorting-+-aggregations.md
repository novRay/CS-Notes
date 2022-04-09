# #09: Sorting + Aggregations

本节大纲：

* External Merge Sort
* Aggregations

## Sorting

### Why Do We Need To Sort?

* 关系模型/SQL是无序的
* Query需要我们对tuples进行某种排序（**ORDER BY**）
* Query不指定要排序，但我们还需排序来实现其他事情：
  * 去重（**DISTINCT**）
  * 批量加载排好序的tuple到B+树中，速度更快
  * 聚合（**GROUP BY**）
  * ...

### Sorting Algorithms

如果数据能fit进内存，可以使用标准的排序算法，如quicksort。

但如果数据不能fit进内存，就需要一种考虑读写磁盘开销的技巧。

#### External Merge Sort

分治算法：把数据分割成分立的**runs**，单独对他们排序，然后将它们组合成一个更长的有序runs

* Phase #1 - Sorting：对能fit进内存的数据块(chunk)排序，然后写回磁盘
* Phase #1 - Merging：将有序的runs组成成大的chunks

一个run是a list of key/value对，其中key是用于比较的attribute，value有两种选择：

* Tuple（早物化）
* Record ID（晚物化）

![](<../.gitbook/assets/image (10) (1) (1) (1) (1).png>)

#### 2-Way External Merge Sort

* ”2“是每趟用于合并成一个新的run的run的数量
* 数据被切分进N页
* DBMS有B个buffer pool来输入输出数据。

_Pass #0_

* 读取B页进内存
* 对每页排序，写回磁盘

_Pass #1,2,3,..._

* 递归地合并两个run成一个两倍长度的run
* 用到三个buffer pages（2个放输入，1个放输出）

![2-way external merge sort](<../.gitbook/assets/image (14) (1) (1) (1).png>)

流程及复杂度分析：

![](<../.gitbook/assets/image (9) (1) (1) (1) (1) (1).png>)

#### Double Buffering Optimization

当系统处理当前run时，预拉取下一个run，存入第二个buffer。

可以减少等待I/O请求的时间，连续地利用磁盘

![](<../.gitbook/assets/image (7) (1) (1).png>)

#### General External Merge Sort

_Pass #0_

* 读取B页进内存
* 生成 \[N / B] 有序的run，每个run的大小为B

_Pass #1,2,3,..._

* 合并B-1个run（i.e., K-way merge）

![](<../.gitbook/assets/image (4) (1) (1) (1).png>)

#### Using B+Tree For Sorting

如果被排序的表已经在sort attribute(s)上建立B+树了，那么我们可以利用它来加速排序。

按所需的顺序遍历树的页结点即可。

Cases to consider:

* Clustered B+Tree
* Unclustered B+Tree

_Case #1 - Clustered B+Tree_

从left-most leaf page遍历即可。这种办法总是比外排序好，因为没有计算开销，磁盘访问也是顺序的

![Clustered B+Tree](<../.gitbook/assets/image (2) (1) (1).png>)

_Case #2 - Unclustered B+Tree_

需要追随指针来找目标数据所在页，一个bad idea，每条数据都有可能引发I/O开销

![Unclustered B+Tree](<../.gitbook/assets/image (5) (1) (1) (1) (1).png>)

## Aggregations

把某个attribute上的多条数据坍缩成一个单独的标量值

两种实现选择：

* Sorting
* Hashing

### Sorting Aggregation

![Sorting Aggregation](<../.gitbook/assets/image (15) (1) (1) (1) (1).png>)

如果我们并不需要数据是有序的呢？比如**GROUP BY**，**DISTINCT**

Hashing在这种情景下是一个更好的替换方案。

* 只去重，不需要排序
* computationally cheaper

### Hashing Aggregation

DBMS扫描表时，生成一个临时哈希表。对于每条记录，检查哈希表中是否已经存在一个entry了：

* **DISTINCT**: 丢弃重复记录
* **GROUP BY**: 执行聚合计算&#x20;

如果能fit进内存，那很简单。如果DBMS必须把数据溢出到磁盘呢？

#### External Hashing Aggregation

_Phase #1 - Partition_

* 基于hash key，把tuples放进对应的buckets（用哈希函数h1）
* 当buckets满，写回磁盘

![](<../.gitbook/assets/image (12) (1) (1) (1).png>)

Phase #2 - ReHash

* 对于每个partition，读入内存，用哈希函数h2建立in-memory的哈希表，计算聚合

ReHash的目的在于区分阶段1中的哈希碰撞的值，并让数据fit in memory

![](<../.gitbook/assets/image (11) (1) (1) (1).png>)

在 ReHash阶段中，存着(GroupKey -> RunningVal)的键值对，当我们需要向哈希表中插入新的 tuple 时：

* 如果我们发现相应的 GroupKey 已经在内存中，只需要更新 RunningVal 就可以
* 否则，插入新的 GroupKey 到 RunningVal 的键值对

![](<../.gitbook/assets/image (8) (1) (1) (1).png>)

