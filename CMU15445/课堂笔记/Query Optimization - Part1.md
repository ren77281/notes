```toc
```
SQL是声明性的，即没有告知DBMS要这么做，只告知DBMS我需要查询的结果
与SQL相对，直接写执行计划也是一种查询方式，但是执行计划十分具体，需要主动考虑其中的细节
不同的执行计划，性能表现也不同。所以对于SQL，我们需要使用优化器对SQL进行优化，以尽可能达到最优的执行表现
优化方式分为两种
- Heuristics/Rules：基于规则的启发式优化，重写查询语句，删除语句中不合理的地方。这种优化方式需要检查元数据（比如表的情况，数据的情况），但是不需要检查数据本身
- Cost-based Search：基于代价模型的搜索，在等效代价中找出最小代价。这种优化方式需要检查数据本身，只有知道了具体数据才能做代价的估算

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161316542.png)

1. SQL Rewriter：对于SQL文本进行预处理（较少）
2. Parser：将SQL文本转换成抽象语法树
3. Binder：将表名、列名、索引名转换成数据库内部认识的ID、标识符。转换的过程也会进行检查，由此判断用户的SQL是否非法
4. Tree Rewrite：将抽象语法树转换成逻辑计划，作为优化器优化的起点
5. 优化器优化完成后，将生成一个可执行的物理计划

Logical vs Physical Plans
逻辑表达式与物理表达式存在映射关系。比如Join算子，对应着具体的执行计划，如Hash Join，Sort-Merge Join
优化器将生成一张映射表，维护逻辑表达式与物理表达式的映射关系
物理表达式定义了具体的数据访问方式，比如全表扫描，通过索引访问

查询优化是一个NP-Hard问题，我们不能确定优化的结果是最优的

## Relational Algebra Equivalences
如果两个表达式输出的结果相同，我们就说这两个关系代数表达式等价
在不使用代价模型的情况下，DBMS可以识别出更好的查询计划，替换原来的查询计划
也叫query rewriting
比如以下表达式，我们可以进行谓词下推
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161340775.png)

此外，还存在着许多等价的关系表达式。如下：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161342891.png)

Inner Join满足交换律与结合率，n张表做Inner Join，不同的Join方式有4的n次方种，要选择合适的Join方式也是一个难点
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161344864.png)

在某些运算中，对于不在输出结果的数据，我们有两种处理方式
- 丢弃它们，产生更小的输出结果
- 保留它们
通常我们都会丢弃它们，只保留有效数据
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161358984.png)
## Logical Query Optimization
逻辑查询的优化：通过**模式匹配规则**，将其转换成等价的逻辑查询，以执行更高效的查询
但问题是：因为没有代价模型，所以无法进行查询之间的比较。为什么说转换后的查询更高效？因为你内置了模式匹配规则，程序默认这样转换能得到更高效的查询。满足了规则程序就会执行转换

逻辑查询优化分为四个步骤
- Split Conjunctive Predicates
- Predicate Pushdown
- Replace Cartesian Products with Joins
- Projection Pushdown
### Split Conjunctive Predicates
将谓词拆分成最简形式，以便优化器更容易地移动它们
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161407075.png)

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161408733.png)
### Predicate Pushdown
尽可能地将谓词移动到离表最近的地方
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161413254.png)
### Replace Cartesian Products with Joins
使用谓词与笛卡尔积合并，转换成内联
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161415476.png)
### Projection Pushdown
在吐出数据前，丢弃无用数据，减少物化成本
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161416196.png)
## Nested Sub-Query
对于子查询，我们有两种优化方式
- 重写：将子查询与父查询写成一条SQL
- 解耦：提前执行子查询
### Rewrite
这条SQL语句似乎存在语法问题，有点不理解
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161422150.png)
### Decompose
学习了火山模型后，我们知道嵌套查询是非常低效的，如下：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161424261.png)
外查询每拿到一条tuple，都需要执行一遍子查询。我们可以将其解耦，先得到子查询的结果，作为常数带入外查询
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161426385.png)

## Expression Rewriting

where 1=0永远为假，查询语句永不执行。通常可以基于已有的表，使用where 1=0来创建一张相同结构的表
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161440400.png)

Expression Rewriting通过对表达式（如谓词）进行重写，来获得一个更高效的表达式
和逻辑查询优化一样，重写也是基于预设的规则，如if/then/else子句，或者模式匹配规则
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161438737.png)

Join条件的重写，比如以下的SQL语句：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161445671.png)
等价与select * from A
谓词的重写，merge，如：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161447585.png)
## Cost Model
通过执行特殊的查询计划，生成对当前数据库状态的评估
在不同硬件设备上比较评估结果没有意义，只有在同一设备上比较不同查询计划的才有意义
代价一般包含以下方面：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161452264.png)

物理开销模型，一般只有在商用一体机上采用。因为这种方案与硬件强耦合，只有熟悉硬件性能才能很好地评估代价
面向磁盘DBMS的开销模型中，磁盘I/O为主要开销。如果DBMS实现了Buffer Pool，就能更好地估计开销，因为几乎所有的磁盘I/O都经过了Buffer Pool

pg的开销模型，分别计算了CPU开销与I/O开销，分别乘以一个“魔法数”，加起来得到一个总开销
默认情况下，内存空间都远小于磁盘空间。这个魔法数假设顺序I/O快于随机I/O 4倍，处理内存中的tuple比处理磁盘中的tuple快400倍
如果修改了魔法数，可能将导致负优化查询计划，因为pg的所有开发，测试，查询优化都和这个魔法数有关

IBM下，DB2的开销模型，考虑了更多情况，如：
- 分布式环境下，通信的带宽
- 内存资源，如缓存池，排序用的heap
- 并发环境下，用户的平均数量，隔离级别，可用锁的数量
- 数据库中，数据本身的特点
- 硬件环境，配置
- 存储设备特征

