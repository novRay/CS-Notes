# #10: Joins Algorithms

## Why Do We Need To Join?

在关系型数据库中，我们常常需要对表进行正规化(normalization)来避免冗余的信息。这意味着我们需要用join算子重建没有信息损失的原始tuple。

如下图所示，Artist和Album之间的多对多关系拆成Artist, Album, ArtistAlbum三张表，使得数据存储的冗余减少。查询时，我们就需要用Join语句来重建关系，获得完整数据。

![](<../.gitbook/assets/image (7) (1).png>)

## Join Algorithm

我们主要关注**inner equijoin**算法，即将key相同的表join到一起，这些算法可以被调成来支持其他算法。

通常情况下，我们让较小的表称为左表(left table)，或者外表(outer table)

## Join Operators

Join算子的考察维度：

* 输出：在query plan树上，算子吐出怎样的数据给它的parent算子？
* 开销分析：如何测定某种算法比另一种更好？

### Operator Output

* **Data**: Early Materialization模型。将inner tuple和outer tuple的attribute组成新的tuple。之后的操作都不需要再从原表中获取数据。

![Early Materialization](<../.gitbook/assets/image (2).png>)

* **Record Ids**: Late Materialization模型。只记录匹配上的tuple的record id，后续操作根据id再从原表中获取数据。对列存友好，因为DBMS不需要复制多余的数据。

![Late Materialization](<../.gitbook/assets/image (9) (1) (1) (1) (1).png>)

### Cost Analysis

![Cost Analysis](<../.gitbook/assets/image (8) (1) (1).png>)

## Nested LoopJoin

简单暴力双循环：

![Stupid Nested Loop Join](<../.gitbook/assets/image (15) (1) (1) (1).png>)

为什么说它stupid？R中的每一条数据，都要全表S作比较。

$$Cost=M + (m ×N )$$

### Block Nested Loop Join

对stupid方法的改进，每次取R表的一个block，来与S表比较。

![Block Nested Loop Join](<../.gitbook/assets/image (16) (1) (1) (1).png>)

$$Cost=M + ( M×N )$$

如果我们有B个buffer可用，则可以用B-2个buffer来扫描外表，1个buffer扫描内表，还有1个buffer放输出：

![](<../.gitbook/assets/image (3) (1) (1) (1).png>)

$$Cost: M + ( [M / (B-2)] ∙N)$$

如果外表可以整个放进内存，即$$B > M + 2$$，则Cost可降为$$M + N$$

### Index Nested Loop Join

为什么基础的嵌套循环很差？因为对于外表的tuple，总是要对内表全表扫描来检查。

因此我们想到用采用建立**索引**，来避免对内表的顺序扫描。

![Index Nested Loop Join](<../.gitbook/assets/image (18) (1) (1) (1).png>)

假设索引探测的时间开销为常数$$C$$，则总代价为：

$$Cost=M+(m×C)$$

### Key Takeaways

* 选取较小的表作为外表
* 尽可能多地buffer外表进内存
* 对内表作循环（或者用索引）

## Sort-Merge Join

_Phase #1: Sort_

* 基于join key对两表排序
* 可以用上节课提到的外归并排序算法。

_Phase #2: Merge_

* 用cursor遍历两表，吐出匹配的tuple
* 可能需要回溯(backtrack)，取决于join的类型

#### 代价分析：

$$Sort Cost (R): 2M ×(1 + ⌈logB-1 ⌈M / B⌉⌉)$$

$$Sort Cost (S): 2N ×(1 + ⌈logB-1 ⌈N / B⌉⌉)$$

$$Merge Cost: (M + N)$$

$$Total Cost: Sort + Merge$$

最坏情况：两表所有join key都只有一个值。$$Cost: (M ×N) + (sort cost)$$

**什么时候sort-merge join有用？**

* 两表中的一个或两个都已经按join key排好序了，如聚簇索引
* query输出必须按join key排序

## Hash Join

### Basic Algorithm

_Phase #1: Build_

* 扫描外表，对join attribute用哈希函数$$h_1$$生成哈希表

_Phase #2: Probe_

* 扫描内标，对每个tuple用$$h_1$$得到哈希值，在哈希表上对应位置找到匹配的项

![Basic Hash Join Algorithm](<../.gitbook/assets/image (6) (1) (1).png>)

#### Hash Table Contents

* **Key**: Join attribute(s)
* **Value**: 根据实现可变：
  * **Full Tuple**: 避免重取外表内容，但会占用更多空间
  * **Tuple Identifier**: 可以是基表的，也可以是子算子生成的中间表的。适用于列存。也适用于join选择性低的情况。

#### Probe Phase Optimization

Build Phase创建Bloom filter，过滤掉不太可能出现在哈希表中的key

* 探测哈希表前，线程会先检查filter。速度更快，因为filter能fit进CPU cache
* 也叫**sideways information passing**

![](<../.gitbook/assets/image (19) (1) (1).png>)

**Bloom Filters**

一种概率型数据结构（bitmap），测试某个元素是否是某个集合的一员。可能出现false positives，但不可能出现false negatives。

主要两种操作：

**Insert(x)**：用k个哈希函数，将filter中的一些bits置1

**Lookup(x)**: 对于每个哈希函数，检查是否对应位为1

Example：

如下图，插入字符串’RZA‘，将第6位和第4位置1。插入字符串'GZA'，将第3位和第1位置1

![](<../.gitbook/assets/image (11) (1) (1).png>)

查询字符串’Raekwon‘，哈希后检查第5位和第3位。由于第5位为0，得到false

![](<../.gitbook/assets/image (10) (1) (1) (1).png>)

查询字符串'ODB'，哈希后检查第6位和第3位，均为1。得到True（false positive）

![](<../.gitbook/assets/image (12) (1) (1).png>)

Basic Algorithm的问题：

我们可能没有足够大小的内存，来存放整个哈希表。

我们不希望buffer pool manager随机地把哈希表交换进出磁盘。

### Grace Hash Join / Hybrid Hash Join

用于哈希表放不进内存时：

* Build Phase: 用相同的哈希函数对两表都进行哈希partition
* Probe Phase: 比较两表对应partition内的tuple，进行join

![Grace Hash Join](<../.gitbook/assets/image (1) (1) (1) (1).png>)

如果某个partition还是放不进内存，则对它递归分区(**recursive partitioning**)，用另外一个哈希函数继续对它分区，使它能放进内存

![recursive partitioning 1](<../.gitbook/assets/image (13) (1) (1) (1).png>)

![recursive partitioning 2](<../.gitbook/assets/image (4) (1) (1).png>)

#### 代价分析：

Hash Join开销：

* 假设有足够buffer
* Cost：$$3(M+N)$$

Partition Phase:

* 读写两表
* $$2(M+N)$$ IOs

Probing Phase:

* 读两表
* $$M+N$$ IOs

#### Observation

如果DBMS知道外表的大小，则可以使用一个静态哈希表，降低开销。

如果不知道，则必须使用动态哈希表，或者允许页溢出。

## Summary

| Algorithm               | IO Cost                 | Example      |
| ----------------------- | ----------------------- | ------------ |
| Simple Nested Loop Join | $$M + (m×N)$$           | 1.3 hours    |
| Block Nested Loop Join  | $$M + (M×N)$$           | 50 seconds   |
| Index Nested Loop Join  | $$M + (M×C)$$           | Variable     |
| Sort-Merge Join         | $$M + N + (sort cost)$$ | 0.75 seconds |
| Hash Join               | $$3(M + N)$$            | 0.45 seconds |

## Conclusion

Hashing几乎总是sorting要好。但是如果数据不均匀，或者结果需要有序，那么sorting更好。

好的DBMS会用其中一个或两个

