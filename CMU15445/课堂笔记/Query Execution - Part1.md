```toc
```
## Process Models
DBMS的处理模型定义了系统执行查询计划的过程
我们需要根据不同的工作负载选择不同的模型

### Iterator Model火山模型
整个过程xiang岩浆一样，数据从底部一点点地汇聚，最终在顶端喷发，但是我们能控制喷发的量。比如我们只需要输出100条数据，我们只需要限制上层算子，当其输出到100条数据时，就停止运算。所以对于数据结果的控制，我们不需要限制底层算子

- 每个算子都向上层提供了Next函数，每次调用Next函数时，算子都会向上层返回tuple或者null以表示运算的完成
- 每个算子都会不断地调用子节点的Next函数，获取tuple并处理它们
火山模型也叫流式模型

以下是每个算子Next接口的伪代码实现
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407152027259.png)

可以看到：连接算子需要不断调用左孩子的Next，拽出tuple后用来构建hash表。这个过程并不会向上产出数据，也就是说数据的流动被阻塞了。hash表建立完成后，将调用右孩子的Next，拽出数据后，probe hash表，此时数据才会向上流动
- 几乎所有的DBMS都实现了火山模型
- Join, Subqueries, Oreder By算子都会阻塞数据的流动
- 容易控制模型的输出
但问题是：函数调用次数过多，虽然和IO次数比，函数调用时间可以忽略不记，但是相等其他模型，不断调用Next函数导致了极大的开销

### Materialization Model物化模型
在不了解数据库内核，作为使用者时，直观感受到的模型就是物化模型
下面是物化模型的伪代码
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407152037574.png)
可以看到：每个算子不再返回tuple，而是将所有tuple存储在数组中，返回数组
数据在物化模型中的流动是经常阻塞的，上层算子必须等待下层算子获取所有tuple后，才能得到数据
比如筛选算子，它将遍历全表（如果数据无序），获取所有`value > 100`的tuple，再交给上层，此时连接算子才能进行Join

- 适合OLTP场景，因为TP场景中，点查询多，数据量较小（AP场景下，每个函数吐出的数据将是非常惊人的）
- 函数调用次数极少

### Vertorization Model向量模型
火山模型和物化模型的函数调用次数是两个极端，向量模型则是中间派
与火山模型相同，每个算子都有Next接口，但是Next不再吐一条tuple，而是吐多条（a batch of）tuple。每次吐出的tuple的数量可变
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407152052493.png)

- 相对于火山模型，向量模型对AP友好，因为函数调用的次数少了。相对于物化模型，每次函数调用产生的数据量也少了
- 能够充分利用支持向量指令的CPU性能

**Processing Direction**
查询的执行过程分为自底向上和自顶向下
自顶向下：数据的传输都是通过函数返回值，
自底向上：可以对数据流动进行更细的控制

## Access Methods
DMBS如何访问表中的数据？也就是最底层的算子是如何工作的?
- 顺序扫描
- 索引扫描
- 多索引/位图扫描

### 顺序扫描
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407152101666.png)
遍历table的每一页再遍历页中的tuple，判断tuple是否符合筛选条件
这是一种直接，暴力的访问方式

基于全表扫描的优化：
- 预读取
- Buffer Pool Bypass（不使用缓存池）
- 并行扫描
- Heap Clustering
- Zone Maps
- 延迟物化

**Zone Maps**：记录每个page的额外信息，如最大/最小/平均值。在全表扫描前先读取Zone Map，判断是否有扫描相应page的必要
但是Zone Map极大地增加了DBMS的复杂性，一是在什么地方存储Zone Map，二是每次修改数据后都需要维护，维护的成本，三是额外的空间开销

延迟物化：根据筛选条件，只获取相应的列，向上吐出列相应的行ID，需要获取具体数据时再回表查询。合适列存数据库
### 索引扫描
通过索引筛选想要的tuple，如何选择索引？我们需要选择一个能筛掉大部分tuple的索引，直接扫描剩下的tuple，这样能提高筛选的效率
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407152121639.png)

### 多索引扫描
先用一个索引筛选出结果，再用其他索引筛选出结果，最后将这些结果取并集
如果还有筛选条件，就直接扫描并集
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407152121742.png)
## Modification Queries
修改数据库的算子还需要负责检查约束和更新索引
**update/delete**
子算子吐出行ID给修改算子，由修改算子回表进行修改操作，但是必须记录/追踪之前已经修改过的算子
比如：更新tuple时，update算子根据索引从前往后遍历整张表，从表中取出tuple更新后再插回tuple，此时索引已经更新。但是从前往后遍历时，仍然可能遍历到已经update过的tuple，再次进行更新。所以update算子必须追踪修改过的算子，防止多次update
**insert**
子算子提供物化好的tuple，insert算子负责将tuple进行插入
## Expression Evaluation
DMBS用表达式树表示where子句，where也叫谓词表达式
表达式树可能包含以下符号：
- 比较运算符
- 逻辑运算符
- 算术运算符
- 常量
- Tuple Attribute Referrences
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160833188.png)

以下表达式树存在效率问题，存在可以优化的地方
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160838688.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160837879.png)
每取出tuple的attribute后，就需要进行+运算，实际上+运算的结果是一个常数。高效的数据库
可以在查询语句后加上`select * from table where 1=1`来测试查询效率

总结：
- 相同的查询语句，可以被不同的方式执行
- DBMS应该尽可能地使用index访问数据
- 树形表达式很灵活，但是有时会很慢

