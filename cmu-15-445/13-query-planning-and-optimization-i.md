# #13: Query Planning & Optimization I

SQL是声明式的(declarative)。不同的query plan的性能差别可能会非常大。

本节和下节将探讨query plan的优化。优化办法主要有如下两种：

* Heuristics / Rules
  * 重写query，去掉愚蠢的、低效的东西
  * 需要检查catalog，但不需要检查数据
* Cost-based Search
  * 用一种模型来评估query执行的开销
  * 评估多个等价的query plan，选取代价最小的

![Architecture Overview](<../.gitbook/assets/image (18) (1) (1).png>)

### Logical vs. Physical Plans

优化器将逻辑代数表达式映射到优化后等价的物理代数表达式。

物理算子定义了执行策略：取决于处理数据的物理格式（i.e., sorting, compression）；逻辑和物理上的对应关系不总是1:1的。

#### Query Optimization is NP-Hard

这是构建DBMS里最困难的问题。

人们现在开始着眼于使用ML来提高优化器的准确性和效率。



本节大纲：

* Relational Algebra Equivalences
* Logical Query Optimization
* Nested Queries
* Expression Rewriting

## Relational Algebra Equivalences

如果两个关系代数表达式生成了相同tuple集合，那么就说它们是等价的(equivalent)。

基于此的优化不需要cost model。叫做**query rewriting**

### **Predicate Pushdown**

谓词下方，提前缩小数据行范围

![](<../.gitbook/assets/image (10) (1) (1).png>)

### Projection Pushdown

投影下放，提前缩小数据列范围，减少中间结果。对列存不重要。

![](<../.gitbook/assets/image (1) (1) (1) (1) (1).png>)

## Logical Query Optimization

使用模式匹配规则，把一个逻辑计划transform成一个等价的计划。

目的在于增加枚举过程中得到最优计划的可能性。

由于没有cost model，无法比较哪个更好。只是说把计划直接变成preferred的样子。

* **Split Conjunctive Predicates**: 把谓词分解成最简形式，能被优化器更灵活处理。（比如把几个AND拆开成多个谓词）
* **Predicate Pushdown**: 谓词下放
* **Replace Cartesian Product**: 把笛卡尔积替换成inner join
* **Projection Pushdown**: 投影下放，去除冗余attributes，降低物化成本

## Nested Queries

DBMS把where语句中的嵌套子查询看成是一个函数：接收参数，返回一个或一组值。

两个方案：

* rewrite to de-correlate and/or flatten them
* 分解嵌套查询，在临时表中存放结果

#### Rewrite：

![](<../.gitbook/assets/image (5) (1) (1) (1) (1).png>)

#### Decompose:

对于复杂的query，DBMS把它分解成多个block，一次只集中处理一个block。

子查询会被写入临时表中。当整个query结束，临时表会被丢弃。

![](<../.gitbook/assets/image (17) (1) (1).png>)

![](<../.gitbook/assets/image (9) (1) (1) (1) (1).png>)

## Expression Rewriting

用pattern-matching rule engine重写表达式。

有很多例子，例如修改不可能表达式，删除多余join，合并谓词等。

