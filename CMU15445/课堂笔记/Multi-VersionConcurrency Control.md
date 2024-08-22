```toc
```
设计理念：写者不阻塞读者，读者也不阻塞写者
只读事务可以读取一致性快照，而无需加锁
事务通过时间戳，访问数据的快照

T1读A时，将通过时间戳去A的版本中找相应快照，当时间戳满足`being <= ts < end || end = -`且快照提交，获取该快照。否则将获取上一快照
T2写A后，需要维护上一快照的end，并创建新的快照：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241039538.png)

如下图：T2只能读取A0，因为A1由T1创建，但通过事务状态表可知，T1未提交
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241043062.png)
之后T2想写入数据，只能等待T1提交快照：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241044337.png)

可以注意到：MVCC并不是可串行化隔离
如果是可串行化隔离，T1/T2读取的A应该是对方修改后的A，显然它们现在读取的A是未修改前的A
而MVCC并不只是并发控制手段，它直接影响了数据库管理事务的方式
## Concurrency Control Protocol
因为只依赖MVCC无法做到Serializable，所以我们需要与2PL，OCC结合，实现Serializable
## Version Storage
在tuple后增加指针域，使其指向下一条tuple，这样就能创建一条版本链
索引将指向版本链的链头
三种存储版本链的方式：
1. 简单追加：直接在page中插入新的tuple，并维护版本链
假设一张表有10条数据，在这种情况下，可能存储了不止10条数据，因为存在tuple的历史版本
将新tuple插入链表头，还是链表尾？需要根据场景决定
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241058559.png)
2. 使用Time-Travel Table存储版本链：修改数据后，将旧数据追加到Time-Travel Table中，并维护版本链，再修改Main Table的数据
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241102461.png)
3. 使用Delta-Storage-Segment存储修改量
假设当前数据是222，将其修改为333，此时不需要将整个tuple保存到版本链中，只需要保存修改的attribute即可。也就是版本链存储的是指示语句，用来告知：你需要对tuple做什么操作，才能获取当前快照
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241148355.png)
## Garbage Collection
不断维护版本链，它将越来越大，数据越来越多。所以DBMS需要清理那些无用快照，以获取更多内存空间
哪些快照是无用的：
- 活跃事务不需要看见的版本
- 事务回滚时，也需要清理其产生的快照
GC有两种实现：
- 基于tuple
- 基于txn
**基于tuple**，又分为两种实现
一是使用后台清理程序，定时检查。Dirty Block：记录哪些page的tuple被更新过。清理程序将扫描脏页的快照，结合正在并发的事务时间戳，清理无用快照
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241538886.png)

二是协作清理：事务从最早的快照开始，往后找特定快照。如果发现自己不需要当前快照，且其他事务也不需要当前快照，事务将直接删除当前快照，再向后遍历
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241539299.png)

**基于txn的GC**：每个事务记录自己的修改产生了哪些快照，DBMS将基于事务时间戳，获取其快照信息，清理快照
## Index Management
在MVCC中，我们需要如何管理索引？
对于主键索引，我们需要将其指向版本链的链头（假设链头是最新快照）
更新主键索引时，我们先删除再插入（具体为P3中update算子实现）
对于辅助索引，其value值存储什么？
有两种思路：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241602126.png)
一是存储逻辑地址，即主键的key attrs或者RID，这种实现方式获取value后需要回表
二是存储物理地址，什么是物理地址？也就是和主键索引的value值相同，存储tuple的内存地址
这种实现方式存在什么问题？更新tuple时，不止需要维护主键索引，还需要维护辅助索引，而辅助索引可以有多个，这将是极大的维护成本
还存一个思路：辅助索引存储RID，使用额外空间维护RID与tuple物理地址之间的映射。此时只需要在维护完主键索引后，维护一次映射表即可
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241610517.png)
一般来说，索引不会保存版本信息，一个key对应一个value
但在MVCC中，一个key可能需要对应多个value，问题的本质是delete带来的
如下：T1读取A，随后T2更新A产生A2快照，接着删除A，此时表中已经没有A这一数据。但T1未commit，所以A的历史快照A1未被删除
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241625498.png)
此时T3插入A（这里可以理解为：和之前的A主键相同的记录），将产生什么样的快照？
不会产生A3，而是A1，因为表中已经没有了A这一数据，所以A的快照序号需要重新开始递增
可以看到，此时存储了A的索引需要指向两条tuple，故MVCC中的索引结构需要支持重复key
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241631076.png)

## Deletes
MVCC的索引结构必须支持非唯一key
一条查询语句可能返回多个tuple，我们需要根据其版本信息，获取正确的tuple
只有当被逻辑删除的tuple的所有快照对事务不可见时，DBMS才会真正地删除数据
此时我们可以规定，被逻辑删除的tuple无法创建新的快照，也就是上锁的意思
我们需要一个方式，来表示tuple的逻辑删除：
1. 使用额外的列，表示该tuple是否被删除
2. 使用空快照，表示墓碑