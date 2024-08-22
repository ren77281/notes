```toc
```
## 为什么需要数据库？
使用原始文件存储两张表：专辑表与音乐家表
- 如何保证专辑表中的音乐家名字与音乐家表的音乐家名字相同？
- 如何保证音乐家名字不存在非法字段？
- 如果专辑中有多个音乐家，专辑表应该如何存储？
- 如何在专辑表中删除一个歌手？
- 如何实现两个人同时对表的修改？
- 如何查找指定的记录？
- 如何用这些数据创建一个新的应用？
- 修改数据时，系统崩溃如何解决？
此时数据的物理层和逻辑层高度耦合在一起，可用性低，比如：逻辑上的查询和物理上的存储是耦合在一起的，我们需要知道数据的存储方式才能进行查询
使用原始文件无法解决以上的问题，这些问题都是数据库（database management system）DBMS需要解决的问题
一个数据库应该具有的功能：
- 在简单的数据结构上存储数据
- 通过高级语言管理这些数据，DBMS需要知道最高效的数据访问方式
- 物理层面的问题应该留给数据库开发者考虑，用户无需关心这些问题

关系型数据库中表的本质是**关系**
tuple指的是表中的一组字段
DML语言分为两种：Procedural和Non-Procedural（Decleartive）
Procedural：告知数据库，我要查找的数据，同时告知数据库要怎么去执行这条查询。如关系代数
Decleartive：只告知数据库我需要什么数据，具体怎么查，数据库自己想办法。类似SQL语句
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406031304257.png)
该关系代数将影响数据库的执行
SQL语句中，我们只需要告知数据库：将R和S以b_id=102的条件进行连接，并没有告知连接方式，数据库可以自行优化
所以语言和数据是解耦的，数据的存储不会影响语言。所以我们能从语言地角度管理数据库
## 面向磁盘的DBMS
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406031315567.png)

通常情况下，DBMS需要将数据落盘，即保存到非易失磁盘上，这样的架构被称为**disk-oriented architecture**
那么DBMS一定需要管理数据在易失区域和非易失区域之间的移动，这个模块叫做**磁盘管理模块**

关于易失介质与非易失介质：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406031321757.png)
内存往上的存储介质都是易失的，但支持随机访问，速度快。按字节存储（可以精确快速地访问某个字节）
内存往下的存储介质都是非易失的，但只能顺序访问，速度慢。按块存储，块的大小一般为4KB。每次的存取数据必须以块为基本单位，就算只需要访问一个字节，也需要将访问4KB的块再访问里面的数据
不同存储介质的访问时间（将访问L1的时间提升为秒级别）：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406031326265.png)

**对于非易失存储来说，顺序存取的速度远小于随机存取**。所以我们需要尽可能地将随机存取转换成顺序存取

我们需要理解两个问题
- 为什么数据需要在易失与非易失介质之间流动？
- 为什么数据库要自己管理数据的移动，不能使用OS的磁盘管理模块吗

虽然非易失介质空间大，能够存储大量数据。但是访问速度慢，我们需要快速地处理数据，只能使用易失介质
具体来说：磁盘远大于内存，我们无法将磁盘中所有的数据一次性加载到内存中，但可以先加载部分数据，处理部分数据，再加载部分数据，再处理部分数据...直到所有数据被处理完

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406031335304.png)
**为什么要实现自己的磁盘管理模块，不能依赖于OS的mmap吗？**
首先OS的mmap能够将虚拟内存映射为物理内存，使数据库对于虚拟内存的访问等同于对物理内存的访问，不同关心物理内存的情况（也就是将物理内存交给OS管理）。而OS的磁盘管理策略是一个通用的策略，考虑的都是通用的情况，时间空间的效率平庸

补充：在多线程并发的环境下，mmap存在数据不一致的问题。我们需要使用madvise，mlock，msync来保证数据一致性
完全使用mmap实现的数据库有：levelDB，部分的有：mongoDB（现在已经放弃mmap），SQLite

而自己实现内存管理，可以自定义以下策略，这些策略能够显著提高数据库的性能：
- 以正确的顺序进行刷脏（脏页指的是被修改的页）
- 基于局部性原理的预读取
- 内存替换策略
- 多线程控制

所以磁盘管理模块需要解决两个问题：
1. DBMS用什么样的方式在磁盘上体现数据？（静态的表现）
2. DBMS如何在内存和磁盘上存储和移动数据？（动态的调度）
### 存储的静态表现
#### File Storage
DBMS以专用的格式一个或多个文件的方式存储数据库，这个过程经过了操作系统，但操作系统不会关心文件的内容，只是负责存储数据

这里引申一个问题：**既然数据库实现了磁盘管理模块，为什么不再实现一个文件系统？而是依赖OS的文件系统？**
定制文件系统难度大于定制磁盘管理模块，且带来的性能提升低于磁盘管理模块。所以文件系统不值得我们去定制

**Database Page**
OS将文件切分成页进行管理，数据库也将文件切分成页进行管理
页的概念区分：
- Hardware Page（512B/4KB）：我们通常称为块，它是最小单位（数据库的页存储失败，可能是其中某个Hardware Page存储失败）
- OS Page（4KB）：系统以OS Page为基本单位，组织文件，与磁盘进行交互
- Database Page（512B-16KB）：数据库以Database Page组织文件

数据库以页为基本单位进行数据（tuples，index，log）存取，每个页有不同的ID，每个页只存储一类数据，不同类型的页不会混合使用。有些系统要求：页能够**自解释**，这样的页同时存储了页的结构信息与数据信息

不同DBMS管理pages的方式不同：
- Heap File Organization
- Sequential File Organization
- Hashing File Organization
无论怎样管理，DBMS都需要支持页的增删查改，遍历功能

`Heap File`：无序的`pages`的集合，pages的管理模块需要记录元数据：哪些页被使用，哪些未被使用？具体实现有两种：Linked List和Page Directory
**Linked List**
header page在heap file的开头，维护了两个page列表：
- free page list
- data page list
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406041633822.png)
**Page Directory**
管理模块维护着特殊的页，这些页记录了其他页的元数据。如使用情况，directory pages需要与data pages保持同步
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406031514147.png)
#### Page Layout
分为Header和Data两个部分
Header存储：
- 页大小
- 校验和（如MD5）
- 数据库系统的版本号
- 事务相关信息
- 压缩信息 
这里需要引入自解释这个概念：
而自解释：将表结构和页数据同时存储
非自解释：将表结构和页数据分开存储，系统需要先访问元数据页获取表结构，再获取数据
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406031514805.png)
#### Data Layout
page的Data部分记录着真实有效的数据，那么**页的数据如何存储？**
1. Tuple-Oriented：记录数据本身
2. Log-structured：记录数据的操作日志

原始的想法：用header记录tuple的数量，然后将tuple不断往下append
存在的问题：
- 无法存储变长tuple
- 删除tuple后容易出现内存碎片
- 如果想利用删除tuple后空出的空间，需要从头遍历找足够长的空位（费时）
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406041652829.png)
最常见的布局为：slotted pages
**Slotted Pages**
Slot Array中的每个slot存储相应tuple的信息：指针，大小等
Slot Array从前往后增长，tuples从后往前增长
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408021245058.png)

在Header存储tuple的更多元数据，就能使得tuple布局更加灵活
**Log-structured**
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406031557466.png)
存储数据的操作日志，将随机写转换成了顺序写，增删改的效率极高
但是查询效率较低：需要从新数据开始往旧数据读取，直到遇到该数据的插入记录（最初记录），接着执行这些操作语句（回放），得到最终的数据
为了加快查询速度，通常会在操作日志的key值上建立索引，保存同一key值的相关操作日志

这样的日志存储方式将使得数据库体积急速膨胀，但是存在着很多冗余数据。我们需要定时压缩：经过删除或修改，留下最少的，能反应出数据情况的语句，删除之前的冗余操作
- 按层压缩（rocksDB能压缩到7层）：每层的文件是上层文件的压缩结果。数据在某个文件insert，又在某个文件update，此时无法将文件单独压缩，需要将文件合并压缩，并在下一层产生新的文件。读取数据时，从低层向高层开始读，如果当前层没有数据，该数据可能压缩到下一层
- 普通压缩：直接将文件合并压缩，没有层的概念
典型数据库如：levelDB，rocksDB，通常用在**kv数据库**中
#### Tuple Layout
> 已经讨论了Page Layout，Data Layout，接着讨论Tuple Layout

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406031533726.png)
每个tuple有自己的header，存储tuple的元数据
- 事务相关信息
- NULL位图：如果某个字段为NULL，需要用位图表示。因为所有字段都紧挨在一起，不特别表示NULL可能导致数据解析错误
其中：每个tuple拥有ID，一般都是page_id+offset/slot，这个id只在内部可见，外部无法使用
当然，tuple不需要保存表结构数据
#### 小结
- 数据库由页组成
- 页布局：Page Directory，Linked List
- Data布局：slot array，header，tuples，相对增长
- tuple布局：header，数据紧挨着存储，NULL位图
#### Tuple Storage
tuple在文件就是一串字节，DBMS需要解读其中隐含的数据类型和数据本身。DBMS的catalog保存着数据表的元数据信息，DBMS依赖这些信息解读数据
##### Data Representation
数据库中的数据类型有：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406041728264.png)

C++/Go中无法精确表示浮点数，需要使用特殊的方式表示精确浮点数，可能的表示方式有：
```cpp
// MySQL的decimal的存储方式
struct decimal_t {
	int intg, frac, len; // 数据在小数点前的位数，数据在小数点后的位数，buf数组的长度
	bool sign;           // 数据的正负
	decimal_digit_t *buf;// 指向整数类型的指针（忽略浮点数的前导0和小数点）
}
```
**Large Value**
如果某个字段特别长，将使用溢出页存储溢出数据，tuple存储一个指针，指向这个溢出页
通常情况下，不允许tuple的大小超过单个页
MySQL中，当行的大小超过页的一半时，被视为溢出，使用溢出页
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406041732961.png)
**External Value Storage**
通常不会在数据库存储特别大的数据，如视频，图片。这些资源一般存储在外部网盘中，可以在数据库存储指针，通过数据库获取指针再访问资源
#### System Catalogs
DBMS需要存储不同数据库的元数据：
- 表，列，索引，视图
- 用户，权限
- 内部统计信息
DBMS基本使用自身的表存储元数据，可以通过INFORMATION_SCHEMA数据库查询信息，获取所有数据库的元数据
#### Database Workloads
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406041747766.png)
- On-Line Transaction Processing (OLTP)：读写量较小的事务，简单的读写语句，操作速度较快，操作对象较小（用户使用）
- On-Line Analytical Processing (OLAP)：复杂查询，大量读取数据（公司内部使用，统计信息，如app年终总结）
- Hybrid Transaction + Analytical Processing（HLAP）：OLTP和OLAP的混合
正常的数据库系统结构：交易集群（OLTP Data Silos）+数据仓库（OLAP Data Warehoust）
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406032108502.png)

交易集群负责实时响应用户的请求，经过一段时间后，交易集群将数据**导出并传输**给数据仓库，数据仓库**加载**数据并进行分析，分析完成，将数据写回交易集群
交易集群只能处理简单的SQL事务，无法复杂查询，这可能导致机器崩溃
#### Storage Model
常见的Data Storage Model有：`N-ary Storage Model`与`Encomposition Storage Model`

N-ary Storage Model（NSM）：行存，将一个tuple的所有attributes在page中连续的存储，适合OLTP场景

在OLAP场景下，我们只要某些attributes，但必须遍历所有的tuple才能得到我们想要的attributes，这是一个费时的操作（遍历的无效数据太多）
优点：高效插入、更新、删除，只涉及小部分tuples
缺点：不利于检索大部分tuples与只需要一部分attributes的查询

Encomposition Storage Model（DSM）：列存，将所有tuple的单个attributes连续地存储在一个page中，适合OLAP场景
DSM可以优雅地处理NSM查询浪费I/O的问题。但是新的问题是：如何追踪attributs属于的tuple？
- Fixed-length Offsets：相同attributes定长存储，通过offset追踪（常用）
- Embedded Tuple Ids：在每个attributes前加上tuple Id
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406041951876.png)
优点：OLAP场景下，能够有效减少磁盘I/O次数，利于数据查询与压缩
缺点：由于tuple过长导致的切分/合并。点查询，更新，删除，插入速度慢
**现在的数据库基本都实现了列存引擎**
#### 小结
存储引擎和数据库紧密耦合在一起，对于不同的工作负载我们需要选择合适的存储引擎

