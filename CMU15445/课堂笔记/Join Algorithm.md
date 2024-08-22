```toc
```

在关系型数据库中，我们根据数据的不同特征，分开存储不同数据，以避免数据的重复存储。而这些不同数据之间存在着关系，所以我们可以通过相同的关系将不同的数据Join起来，从而构建新的tuples，使数据变得完整

Join时，尽量将小表（page数量少的）作为左表（外表/驱动表）

根据Join的输出结果，我们可以将Join分为两类。一是在输出结果中保留非Join Attribution的attributions，这样就可以直接从输出结果获取数据，不需要进行回表。二是只在输出结果中保留行id，这样的数据结果较小，但是需要回表才能获取相关数据。这样的Join方式被称为延迟物化(late materialization)

连表算法有三种
- Nested Loop Join
- Sort-Merge Join
- Hash Join
## Nested Loop Join
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407151249829.png)
通常将R表称为外表（可能因为是在外循环），S表称为内表
Nested Loop Join需要在遍历外表的每条数据时，遍历内表的每条数据
这个就是原始的暴力算法，速度慢
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407151253633.png)
计算Join的IO次数，那么该算法的开销Cost: M + (m \* N)
因为Join的是两张大表，每次只能分别读取两张表的其中一个page到内存中
每次遍历内表时，需要从头到尾读取内表的所有page到缓存池中，下次遍历内表时，需要读取的page已经被淘汰，需要重新读取

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407151255401.png)
提出改进：不再遍历tuple而是遍历page，遍历R表的每个page时，遍历S表的每个page，在内存中进行tuple的匹配。这样的Cost: M + (M \* N)
这这个算法中，我们需要尽可能地以page数量少的表作为驱动表，因为这样的IO次数会更少

假设缓存池现在有B个frame可用，如何Join能使IO次数最少？
首先需要一个frame来存储输出结果，还需要一个frame来存储S表的block，剩下B-2个frame都用来存储R表的block，那么Cost: M + (M / (B - 2) * N)
其中M / (B-2)是上取整

那么如果缓存池有M + 2个frame可用，那么Cost：M + N

为什么Nested Loop Join这么慢？遍历R表的每个page时，只能通过顺序扫描S表来进行匹配
如何加快顺序扫描？对Join attribution建立索引。假设建立索引后，在S表查找数据的时间常数是C
那么，Cost：M + (m * C)

总之
- 我们需要选择page数量少的表作为驱动表
- 尽可能地在缓存池中存储驱动表
- 尽可能地使用索引扫描内表
## Sort-Merge Join
1. 根据Join Attribution对两张表进行排序
2. 使用双指针合并两张表
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407151626883.png)

用双指针遍历R表和S表
- R表的数据大，R指针往后走
- S表的数据大，S指针往后走
- 数据相同，则Join两条数据，两指针同时往后走
以上算法针对的是Join Attribution唯一的情况，如果Join Attribution不唯一，在数据相同时，只需要移动S表指针
此外，还存在着特殊情况，如果R表的Join Attribution重复，只有第一个Join Attribution会进行Join
比如R表的数据为200，200，S表的数据为200，200
根据以上算法，只有R表的第一个200会与S表进行Join，后续的200将跳过Join，所以在遍历S表时，需要记录遍历过的上一次数据，当R表的数据和S表的上一条数据相同时，需要往前移动S表指针
Merge Cost：M + N
最坏情况是：两表的数据完全相同，此时Sort-Merge将退化成Nested Loop Join

总之，Sort-Merge Join适用于
- 一张或者两张表都已经排好序的情况，如聚簇索引
- 输出结果必须按Join Attribution排序的情况
## Hash Join
思想：只有在Join Attribution相同的情况下，tuple才会进行Join。而相同的Join Attribution经过hash函数得到的hash值也是相同的，我们可以利用额外空间，建立R表的hash表，加快S表的查询速度

1. Build：扫描外表，根据Join Attribution的hash值建立hash表T
2. Probe：扫描内表，对内表的每个tuple的Join Attribution进行hash，用hash值到T表中查找，连接相同hash值的tuple
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407151653135.png)

但是存在一个问题：hash表可能很大，只能加载一部分到BufferPool中。在内表查询时，hash表需要在内存和磁盘中来回移动，因为S表的数据无序，所以page的驱逐是随机的。有没有什么算法能控制page的驱逐？

**Grace Hash Join**
- 对两张表使用相同的hash函数建立hash表，那么只要桶号相同，里面的数据一定是匹配的
- 将相同的bucket加载到内存，进行Join
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407151809108.png)
如果bucket还是过大，缓存池无法存下怎么办？可以采用递归的方法，将过大的bucket进行rehash

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407151712537.png)
