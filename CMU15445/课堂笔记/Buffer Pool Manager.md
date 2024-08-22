```toc
```
### 数据的动态流动
根据计算机体系结构，cpu无法直接访问磁盘，系统需要将磁盘上的文件先加载到内存中，再由cpu进行读写
DBMS页需要将数据库文件先加载到内存，具体来说，磁盘上的数据将被加载到DBMS管理的内存池中

内存池两大作用：
1. 缓存用户需要的数据
2. 缓存用户进行的更新

数据的动态流动又分为两方面：空间控制（Spatial Control）和时间控制（Temporal Contorl）
空间控制：决定page写入磁盘的哪个位置，将常常使用的page写入到磁盘的连续位置
时间控制：何时将数据写回磁盘，何时读取数据到内存，使得磁盘I/O次数最少，提高I/O效率
#### Buffer Pool Manager
DBMS启动时，向系统申请一片较大的内存区域，作为自己的内存池。内存池被划分成大小相同的page（和disk page大小相同），我们将其内存池的page称为帧
当DBMS请求一个disk page时，需要将其复制到Buffer Pool的一个frame中
DBMS将维护一张页表（与进程的页表不同），负责记录每个page在内存中的位置
DBMS将在页表中添加元信息，表示对应的page是否为脏页（Dirty Flag），是否被引用，以及引用计数等
当page table中的page被引用时，会记录引用数，表示该page被使用，空间不够时不应该移除。当请求的page不在page table中时，DBMS会先申请一个latch，表示该entry被占用，然后从磁盘中读取page到Buffer Pool，记录地址到entry上，最后释放latch

**Locks vs Latches/Mutex**
- 一般情况下，Locks指宏观/逻辑意义上的锁，比如锁一张表
- 而Latches表示具体/底层的锁，为了Lock某张表，需要把这张表相关的所有page加上Latch

使用一些策略提供缓存池性能：
- Multiple Buffer Pools
- Prefetching
- Scan Sharing
- Buffer Pool Bypass

**Multiple Buffer Pools**
DBMS在不同维度上维护多个Buffer Pool：
1. 多个缓存池实例，根据page id将page哈希到不同缓存池上
2. 每种page类型分配一个Buffer Pool
3. 每个数据库使用一个Buffer Pool

这样做能够降低latch的粒度，减少事务之间的竞争
**Pre-Fetching**：根据局部性原理或者索引结构，从磁盘读取连续的页

**Scan Sharing**：如果两个事务并发执行，需要扫描同一张表，那么DBMS会将两次扫描合并为一次
具体的说：事务A需要扫描一张表，page 0-page5，0-2已经被扫描，准备扫描3-5，此时事务B页需要扫描page 0-page5，DBMS会让事务B先跟着事务A扫描3-5，之后再扫描0-2
注意：如果事务添加了limit限制，那么Scan Sharing可能导致错误的结果。可以添加order by防止Scan Sharing带来的错误

**Buffer Pool Bypass**：将页存储到未池化的内存区域，绕过DBMS的内存池，执行器使用完这块内存后直接释放（通常是sort和join的结果）。也叫做**Light Scans**，隔离这块内存对缓存池的影响

**OS Page Cache**：通常情况下，操作系统也会缓存page，而数据库实现自己的缓存策略。导致数据被存储了两份，可以使用O_DIRECT告诉系统不要缓存page
#### Replacement Policies（LRU-K）
当Buffer Pool的内存不够时，需要驱逐一些页，以加载新的页。那么应该怎么定制Buffer Pool的替换策略？
替换策略的特征：
- 正确：替换不需要的页
- 准确：不会误操作
- 速度快：每次替换操作都需要latch，尽快释放latch能提高DBMS的并发度
- 决策不需要使用过多的元信息

思路：**Least-Recently Used**，如果一个page最近被使用过，那么将来被使用的可能性更高。所以每次驱逐最长时间未访问的page（访问时，记录page的时间戳）

具体实现：
**Clock时间轮算法**
近似LRU策略，每次驱逐较早的page
为所有的page赋予一个计数器，将计数器排列成数组，遍历整个数组
- 如果计数器为0，驱逐对应page
- 如果计数器不为0，将计数器设置为0
重复以上遍历。想象成一个时钟，每次的遍历是时针的转动

LRU存在的缺陷：
在**Sequential Flooding**中，最近使用的页可能是未来最不需要的页，LRU将驱逐点查询过的热点数据

**LRU-K**
记录页的最近k次使用时间，驱逐页时，根据最近k次访问时间，计算页的权重，淘汰权重最低的页

**Localization**：事务本地化，当前事务将可能地驱逐只被自己使用过的页
**Priority Hints**：优先级提示，根据某些算法为页添加提示，尽可能地不去驱逐被提示的页
#### Dirty Page
不能简单的驱逐脏页，而需要将其写回磁盘，这是一个开销非常大的操作，Replacement Policies需要考虑脏页的问题
一般采用异步（Backgroud Writing）的方式，定期集中地将脏页写回磁盘。Replacement Policies就可以忽略脏页
