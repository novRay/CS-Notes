# #11: Query Execution I

## Introduction

本节将介绍DBMS query的执行和处理。

![](<../.gitbook/assets/image (2) (1) (1).png>)

DBMS会把SQL语句转换成一个查询计划(query plan)。计划中的算子会被组织成一棵树（如上图所示），数据从叶结点流向根结点，根结点的输出即为query的最终结果。

本节大纲：

* Processing Models
* Access Methods
* Modification Queries
* Expression Evaluation

## Processing Models

DBMS的**processing model** 定义了系统如何执行一个query plan。不同的工作量有不同的trade-off。主要有三种模型：

* Iterator Model
* Materialization Model
* Vectorized / Batch Model

### Iterator Model

每个算子执行一个`Next()`方法。

* 每次调用算子返回一个tuple或者一个`null`marker表明没有更多的tuple。
* 算子循环调用子结点的`Next()`，取得tuple并处理。

这种模型也叫做火山模型(**Volcano Model**)或者**Pipeline Model。**

这是一种自下而上的模型。底层算子处理完后向上“吐出“数据，直到根结点。像火山喷发一样。

![](<../.gitbook/assets/image (5) (1) (1) (1) (1) (1) (1) (1).png>)

![Volcano Model的调用](<../.gitbook/assets/image (6) (1) (1) (1) (1).png>)

这种模型几乎运用在每个DBMS中。它允许tuple的pipelining（流处理）。

注意有一些算子会block直到子算子吐出所有的数据，如Join, Subqueries, Order By。（上图2中buildHashTable）

此模型的缺点在于过于频繁的函数调用。

### Materizalization Model

每个算子处理完所有的输入后一次性吐出所有的输出。

* 算子把输出结果“物化(materizalize)”成一个单一结果（如数组）。
* DBMS可以把一些hint(e.g., **LIMIT**) push down，来避免扫描过多的tuple
* 可以发送materialized row或者一个single column

输出既可以是整个tuple(NSM)，或者列的子集(DSM)

这是一中自顶向下的模型

![Materialization Model](<../.gitbook/assets/image (2) (1) (1) (1) (1).png>)

这种模型更适合OLTP：query一次只接触小规模的tuple。

* 降低执行/调度过度开销
* 更少的函数调用

但不适合OLAP：query有大规模的中间结果(intermediate results)

### Vectorization Model

经典hyrid approach，综合了前两者：算子调用`Next()`方法，但是吐出一批(a batch of)数据

* 算子的循环内部一次处理多个tuple
* batch的大小可以基于硬件或query properties改变。

![Vectorization Model](<../.gitbook/assets/image (7) (1) (1) (1) (1) (1).png>)

这种模型是OLAP的理想模型：

* 极大地减少了调用次数。
* 允许算子使用 vectorized instructions (SIMD) 来批量处理 tuples

目前使用这种方法的DBMS：vectorwise, snowflake, SQL Server, DB2, Oracle, Amazon Redshift等

### Plan Processing Direction

1. Top-to-Bottom
   * 从根节点起从子结点拉取(pull)数据
   * tuples依靠函数调用传递
2. Bottom-to-Top
   * 从叶结点起将数据推送(push)给parent节点
   * 允许对流水线中caches/registers更严格的控制

## Access Methods

access method是指DBMS获取存储在表中数据的方式。它不在关系代数中定义。

三种基本的方法：

* Sequential Scan
* Index Scan
* Multi-Index / "Bitmap" Scan

### Sequential Scan

最简单粗暴的办法，直接双循环检查每个tuple是否满足条件：

```python
for page in table.pages:
    for t in page.tuples:
        if evalPred(t):
            // Do Something 
```

DBMS会维护一个内部**cursor**来追踪上一个被检查的page/slot

优化办法：

* Prefetching
* Buffer Pool Bypass
* Parallelization
* **Heap Clustering**
* **Zone Maps**
* **Late Materizalization**

#### Zone Maps

预先计算好一张页内数据的统计信息attribute values。DBMS先检查zone map来决定是否进一步获取数据。

![Zone Maps](<../.gitbook/assets/image (8) (1) (1) (1) (1) (1) (1) (1).png>)

如上图，由于MAX为400，而条件是val > 600，因此不需要扫描该页。

缺点：

* 需要专门一块空间存放所有页的统计信息（zone map不能存页里）
* 数据修改开销大

**Late Materizalization**

延迟拼接tuple，只向上传满足条件的tuple的offset，而不是完整的tuple。适合列存。

![Late Materizalization](<../.gitbook/assets/image (10) (1) (1) (1) (1) (1) (1).png>)

### Index Scan

DBMS挑选query所需tuple的索引。使用哪个索引取决于：

* 索引包含了什么attributes
* query引用了哪些attributes
* attribute的值域
* 谓词的组成
* 索引是unique还是non-unique

在13节会进一步介绍上述内容。

![](<../.gitbook/assets/image (3) (1) (1) (1) (1) (1).png>)

针对不同的情景，选取合适的索引来过滤掉更多的数据。

如上图，在情景1中，选用dept；在情景2中，选用age

#### Multi-Index Scan

如果有更多的索引可供使用：

* 计算符合每个索引条件的record id set
* 基于谓词（union/intersect），将这些set组合
* 取出记录，完成剩下的谓词处理。

在Postgres中，这种方法叫**BitMap Scan**

![](<../.gitbook/assets/image (9) (1) (1) (1) (1) (1) (1) (1) (1).png>)

## Modification Queries

之前提到的query都是读取数据。对于需要修改数据库的query，它们的算子（**INSERT, UPDATE, DELETE**）要负责检查索引的限制条件，并更新索引。

**UPDATE/DELETE**:

* 子算子传递目标tuple的Record ID
* 必须跟踪之前见过的tuple，否则会导致Halloween Problem（见下文）

**INSERT**：

* Choice #1:在算子内部物化tuple
* Choice #2:算子插入任何从子算子传进来的tuple

#### Halloween Problem

![](<../.gitbook/assets/image (4) (1) (1) (1) (1).png>)

更新操作改变了一个tuple的物理位置，导致scan算子多次访问了同一个tuple

出现在clustered tables或index scans

最早由IBM System R的研究员在1976年的万圣节那天发现。

## Expression Evaluation

DBMS用一棵**expression tree**表示**WHERE**语句：

![](<../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1).png>)

DBMS遍历整棵树来实现对数据的过滤。虽然灵活，但整个过程速度很慢。

更好的办法是直接计算表达式，采用JIT Compilation，即时编译成机器码执行。
