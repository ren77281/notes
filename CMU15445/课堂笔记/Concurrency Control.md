```toc
```
并发控制与恢复横跨了数据库多个层级，这是两个十分重要的模块
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407230956001.png)
- 两个用户同时修改一张表，必然存在竞争问题（数据一致性）
- 操作不具有原子性：A给B转账时，A的剩余金额减少，此时数据库崩溃，再次启动时要怎么做？

以上两个问题分别为丢失更新与持久化问题，分别需要使用并发控制与恢复解决

为什么MySQL如此流行，因为它是第一个开源且支持事务并发的数据库
## 事务
事务：在数据库中执行一个或多个操作，以实现一个高级操作
事务是比SQL语句更高层级的动作，是数据库执行的最小单位，即具有原子性

事务交叉，任意地运行，可能导致暂时的数据不一致问题
我们需要一些规则去校验，判断事务的并发是否出现了问题
DBMS只知道最基本的读写操作，不知道业务逻辑，所以我们需要在上层添加校验规则
检验标准：ACID
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231042520.png)
原子性：要么执行，要么不执行
一致性：如转账前后，双方金额总和必须一致（逻辑上正确）
隔离性：当前事务进行的操作，对于其他事务不可见（好像整个数据库中，只有当前事务在运行）
持久性：事务的修改需要落盘（永久保存）
### Atomicty
事务的执行结果只有两种状态：
1. 在commit后，执行完所有结果
2. 被用户/自身abort，回滚已经执行的操作
实现原子性，最常用的方法：**undo log**，还有一种少见的方法：**shadow page**（页备份）
**undo log**
事务每执行一步，都留下记号（log），如果事务回滚，事务需要修改哪些数据（回到什么状态）
在内存和磁盘中都存在undo log
undo log涉及数据恢复，将在后续章节提到

日志的其他两个作用：
- 审计追踪：监控作用（最常用），用来检查数据库出现的问题
- 提高性能：暂时不执行用户的事务，而直接将其写入日志

**shadow page**
执行事务之前，将数据所在页进行备份。对新页做修改，回滚时，用旧页替换新页即可
### Consistency
数据库表现出的“外部世界”，应该是逻辑上正确的
一致性分为两种：Database Consistency与Txn Consistency
Databast Consistency由数据库负责：数据库模拟出的外部世界应该遵循完整性约束
未来的事务应该能看到过去事务的修改
而Txn Consistency则是由业务负责，也就是事务内部的SQL语句需要遵循完整性约束，转账时，减少的前和增加的前不一样，这就是业务层面的问题了
### Isolation
理想中的隔离：事务开始到结束的这段时间中，似乎整个数据库只有当前事务在运行。即看不到其他事务的提交，当前事务的修改其他事务也不可见，直到当前事务提交
为什么需要实现隔离性：使上层用户能够更好地实现业务，上层用户可以认为数据库只有自己在使用，无需考虑其他用户的影响，能够更专注与业务的开发

实现中的隔离，不可能同一时间只有一个事务在运行，必然存在多个事务同时运行的情况。DBMS让这些事务交替运行，使上层用户认为同一时间只有一个事务在运行

那么如何让事务以正确的顺序交替执行？有两种思路：悲观与乐观
悲观：假设事务并发会遇到很多问题，在一开始就不让问题发生
乐观：假设事务并发很少会遇到问题，直到问题发生，才去解决问题

例子：银行给用户发利息与用户的转账同时运行
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231204441.png)

理想的运行情况：两个事务串行地调度，最终两个用户的金额为2120，银行派发了120元利息
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231206572.png)

实际的运行情况：两个事务内的SQL交替运行，可能出现错误的运行结果
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231207811.png)

错误的交替运行：银行少派发了6元
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231207970.png)

那么我们需要实现什么样的规则，使DBMS能够分辨运行结果是否正确？
也就是判断并发调度是否和串行调度等效？

我们将调度结果与串行调度结果等效的调度计划，称为Serializable Schedule
如果每个事务保证了一致性，那么Serializable Schedule具有一致性
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231216774.png)

可串行化给DBMS带来了更多灵活性，因为DBMS有更多调度的选择

#### Conflicting Operations
两个操作冲突：
- 它们来自不同的事务
- 它们在操作同一数据，至少有一个是写

所以冲突有三种：读写，写读，写写冲突
#### 读写冲突
也叫不可重复读：T1先读取数据，T2随后修改，此时T1再次读取，就会读到T2修改后的数据
这样的冲突无法使当前事务认为：数据库中只有我一个事务在运行
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231310780.png)

#### 写读冲突
也叫读未提交、脏读：T1写入数据后，T2随后读取，T2基于该数据进行了修改（比如说在原来的基础上增加/减少）。如果T1回滚，那么T2的修改是无效的，因为T2读到了脏数据。由于这样的数据还未落盘，是一个中间状态，我们不能基于这样的数据进行任意操作
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231314204.png)

#### 写写冲突
假设两条数据A，B。T1写入数据A，T2随后写入A与B。T1再写B，那么最终的结果为T2写的A与T1写的B。等效串行化的运行结果应该是：T1或T2写的A与B

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231326212.png)

我们需要根据以上冲突，检查调度过程中是否正确。而不是利用这些冲突，生成正确的调度计划
DBMS不停地接收用户的事务，只能被动地检查这些事务地调度是否正确

可串行化分为两种：
- Conflict Serializability（大部分数据库试着支持这个 ）
- View Serializability（没有数据库能做到）

如果两条SQL语句没有冲突，我们可以交换这两条SQL语句的调度顺序，这不会对导致不一样的结果
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231343807.png)

尽可能地交换，使最终的事务以真正串行的调度方式等效
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231344462.png)
我们称这样的调度为Serial Schedule

我们不能交换冲突的SQL语句：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231345678.png)
所以以上调度不能称为串行调度
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231346883.png)

假设50个事务并发，我们要怎么基于以上思想设计算法让DBMS知道：当前调度是否是串行调度？
就算实现了这样的算法，计算过程也是一个巨大的开销，我们还需要其他思想来判断串行调度

根据依赖图，判断调度是否可串行化：
以事务id为节点，根据调度先后顺序，将冲突的SQL语句转换成依赖图中的有向边
如果依赖图存在环，那么该调度一定**不可串行化**
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231410074.png)
有向边是什么意思：表示了一个事务必须先执行，另一个事务才能执行。否则将发生数据不一致

以下调度就是一个串行调度，其调度结果等效于串行地调度T2，T1，T3
当然，串行调度完一个事务，将进行commit，才开始下一个事务的执行
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231415972.png)

最后一个例子：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231427702.png)

实际上，依赖图会误判某些串行调度为不可串行，如下图的调度：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231429804.png)
通过观察，我们发现无论调度怎么执行，只要T3最后commit，该调度就是一个串行调度
因为T3无脑写入了数据，写入之前也没有进行读取

但是基于view的串行判断较难实现，以下是不同调度的韦恩图
其中View Serializable表示正确地调度
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231432070.png)
### Durability
持久性：一旦事务commit，数据就必须永久的修改。一旦事务roll back，数据也必须回滚，不能残留。这样之后的事务才能基于正确的数据执行