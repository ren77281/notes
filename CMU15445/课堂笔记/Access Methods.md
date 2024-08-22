```toc
```
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406050817938.png)
这是整个数据库系统的架构，课程自底向上讲解了
- Disk Manager：磁盘管理模块，数据在磁盘中如何存储？（静态）
- Buffer Pool Manager：内存池管理模块，数据如何在磁盘与内存之间流动？（动态）
而Access Methods讲的是：执行器如何从内存池中读写数据？读写方法是什么样的？

读写方法涉及到的数据结构有两种：
- Hash Tables
- Trees
## Hash Tables
Hash Tables在数据库中，主要用来存储：
- Internal Meta-data：元数据
- Core Data Storage：核心数据，如Redis使用哈希表存储所有数据
- Temporary Data Structures：临时数据结构，如Join表后产生的中间表
- Table Indexes：表上的索引

设计哈希表面临的两个问题：
- Data Organization：具体的数据结构是什么样的？
- Concurrency：多线程环境下的并发问题（普通哈希表不支持并发，我们需要对其加以限制）

可以将哈希表理解成允许key值重复的map
关于哈希表的具体数据结构，有两个问题：
1. 通过哈希函数将key值映射为哈希值（哈希函数存在速度与冲突的矛盾）
2. 如何处理冲突？
### Hash Function
由于哈希值不会对外暴露，所以DBMS的哈希表没有必要使用加密哈希函数，我们只希望哈希函数越快越好。现代DBMS使用的哈希函数有：
- [MurmurHash](https://github.com/aappleby/smhasher)
- [Google CityHash](https://github.com/google/cityhash)
- [Google FarmHash](https://github.com/google/farmhash)
- [CLHash](https://github.com/lemire/clhash)
### Hash Scheme
解决哈希冲突的方法有两种：开散列与闭散列
对于开散列，常用的哈希结构有：线性探测哈希，Robin Hood Hash，Cuckoo Hash
为了减小哈希值冲突的概率，哈希表大小通常为元素数量的两倍

**小结**：开散列通常为静态哈希，要求使用者能够预判哈希表的元素数量，提前建立足够大小的哈希表，否则将发生大量的哈希冲突。通常用于Table Join场景
#### Chained Hashing
链式哈希，哈希表由多个哈希桶组成
Chained  Hashing将哈希桶设计为链表节点，通过不断追加链表节点以解决冲突
但是最坏的情况下，哈希表将退化成链表，时间复杂度退化成$O(n)$
#### Extendible Hash
在Chained Hashing的基础上，加入动态扩容的思想。同时每个哈希桶能存储多个value
首先我们需要有两个概念：
- 目录数组：下标对应哈希值，指向一个/多个哈希桶
- 哈希桶：用来存储一个/多个value，桶溢出时将往后追加桶
Extendible Hash一般不会追加桶，而是扩展目录数组，同时扩展哈希桶的数量
对于哈希值，有两个概念：
**Global Depth**：针对目录数组
**Local Depth**：针对目录数组对应的哈希桶（hash buckets）
Depth表示哈希值的前Depth位，而Global Depth为Local Depth的最大值

Extendible Hash的目录数组长度为2的(Global Depth)次方，比如global depth为2，目录数组有4个元素，分别指向了哈希值前2位为：00，01，10，11的哈希桶
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406050926846.png)
每个哈希桶能存储n个value，如果溢出，将导致目录数组的分裂，分裂过程如下：
`Global Depth += 1`，溢出哈希桶的`Local Depth +=1`
目录数组延长为原来的两倍，分别指向哈希值前3位为：000，001，010，011，100，101，110，111的哈希桶
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406050929280.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406050929551.png)
溢出桶内的value将以global depth = 3的规则进行重新映射，再存储新的value
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406050931739.png)

查询时，根据Global Depth在目录数组中选择相应的哈希桶，遍历所有节点找到value

当目录数组无法分裂时（Global Depth为哈希值的bit长度），将进行哈希桶的追加
## Tree Index
hash table最常用的数据结构有：B+ Tree，Skip Lists，Radix Tree
这里只讲解B+ Tree
B+树是一种多路平衡树，与B树的区别是：将所有实际数据存储在叶子节点中，同时叶子节点之间使用双向指针连接，这样的结构支持了顺序访问
相比于哈希表，B+树有一个天然的优点。哈希表需要考虑表的切分：将表切成disk page的大小，然后再落盘。而B+树的节点天然为OS page（或者多个page），此时不需要考虑切分问题，内存中的节点与系统是统一的

B+树的所有操作：search，sequential access，insert，delete的时间复杂度都为$O(logn)$，而sequential access的时间复杂度还与最终所需的数据量有关
B+树结构示意图：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406051132036.png)

B+树的性质，以M阶B+树为例：
- 每个非根节点能存储$\lceil M / 2 \rceil-1  ～M - 1$个key
- 根节点至少存储1个key，至多M-1个key
- 每个非叶子节点的children数量为$\lceil M / 2 \rceil ～ M$
- 有k个key的节点必定有k+1个children
- 严格平衡，所有叶子节点在同一层，也就是叶子节点的深度相同
- 叶子节点以双向链表的形式串联
用于插入/删除导致节点key，children太多/太少，将触发分裂/合并操作。由于这是相对耗时的操作，所以DBMS可能会推迟操作的执行

叶子节点存储的实际数据可以是：
- Record ID：存储主键，通过主键再去索引tuple（回表）
- Tuple Data：存储所有数据，主键索引等价于表结构

在复合索引中，假设`<A, B>`，那么在查找时应该避免`(*,  B)`这样的左模糊搜索出现
左模糊可能导致全表扫描/跳跃搜索，效率低
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406051526444.png)

如果用来建立索引的列值存在重复：
- 将key设置为key+行id（page ID+slot ID）
- 用溢出节点存储相同key值的value
聚簇索引 vs 非聚簇索引
聚簇索引的B+ Tree，对sequential access友好，因为数据在节点上的顺序与数据在磁盘上的数据是一致的，只需要对磁盘进行顺序I/O
而非聚簇索引只存储tuple的主键，这些主键对应的tuple在磁盘上不连续，无法进行顺序读取。但是可以将tuples所在页号先保存起来，最后进行统一的读取

### B+ Tree Design Choices
为了提高B+树的性能，需要考虑以下几个方面
- Node Size
- Merge Threshold
- variable-Length Keys
- Intra-Node Search
#### Node Size
B+树的节点大小应该和文件页大小相同，或是文件页大小的整数倍。这样在存取节点时，能够存取整数个文件页
磁盘速度越低，B+树节点应该越大：存取一次，应该将更多的数据放入内存
工作负载也影响节点大小，如OLAP场景下，经常进行叶节点的扫描，此时节点大小应该大，用更少的I/O次数读取更多的数据
而OLTP场景下，经常进行点查询，从根节点到页节点的遍历，此时节点大小应该小，尽可能少地读取无效数据
#### Merge Threshold
合并的时机：适当的延迟合并能够提高B+树的性能
#### variable-Length Keys
如何处理变长的key？
- 存储key的增长
- padding：用无效数据填充不够长的key
#### Intra-Node Search
节点内部的搜索策略：
- 线性搜索
- 二分
- 推断Interpolation：基于每个key值之间的差值，推断目标key值可能的位置
#### 其他优化
如果需要插入多个节点，将key先排序再插入，这将极大地提高插入效率

