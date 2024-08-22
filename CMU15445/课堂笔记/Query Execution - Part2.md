```toc
```

**为什么需要并行查询？**
- 提供吞吐量（单位时间内能处理的查询语句的数量）
- 降低执行一条查询语句的延迟
- 增加系统可用性与响应能力
- 省钱
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160855330.png)
## Process Models
处理模型定义了在多用户的情况下，数据库要怎样支持并发请求
worker负责执行任务并返回结果
### Process per DBMS Worker
每个worker分配一个进程
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160952683.png)
- 依赖于OS的调度器
- 进程间使用共享内存进行通信
- 一个进程的崩溃不会影响整个系统
缺点：进程调度开销大，请求越多，进程资源的消耗越多
Dispatcher拿到客户端的请求，分配给一个进程后，进程与数据库进行交互，并将结果直接返回给用户
### Process Pool
每个worker使用进程池中可用进程
- 控制了进程资源的消耗
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160958706.png)
Dispatcher拿到客户端请求，分配给进程池中可用进程，进程将结果返回给Dispatcher，由Dispatcher返回给用户
### Thread Pre Worker
每个worker分配给一个线程
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161000181.png)
- 线程调度开销小
- 线程间通信方便，天然的共享内存
- 但是一个线程的崩溃将导致进程的崩溃，可靠性降低
通过线程的并行只是查询语句之间的并行，并不意味着查询语句内也在并行

对于查询语句，DBMS的控制粒度应该比OS更细
比如：
- 查询语句需要使用多少worker？
- 需要使用多个cpu核心？
- worker应该在哪存储其输出结果
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161005563.png)

查询语句内部的线程并发，应该将数据分区，将不同数据喂给不同线程，再整合不同线程的处理结果
比如grace hash join中，hash完产生了n个桶，那么可用使用n个线程分别对每个桶进行Nested Loop Join
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161013050.png)
## Query Parallelism
### 水平切分
将数据分成多个部分，由不同线程处理不同部分，不同线程的执行逻辑相同
DBMS需要引入exchange算子，用来整合结果/分割数据
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161022224.png)
exchange算子应该等待所有线程执行完毕，整合完结果后，再向上层算子输出数据

exchange算子的种类大致分为三种：整合，分割，重新分割
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161025017.png)

查询语句的内部并发：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161028000.png)
### 垂直切分
每个线程负责一个算子，所以不同线程的执行逻辑不同
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161033150.png)
### Bushy
每个算子由不同的线程执行，实现了算子与算子间的并发。同时每个算子中，通过exchange算子实现并发
四张表做Join，使用三个Join算子就能完成，底层的两个Join使用两个线程执行
整合结果后，再将结果切成不同部分，使用两个线程对结果进行Join，最后再整合
所以这里是先水平切分，再垂直切分
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161034223.png)
## I/O Parallelism
将任务切分给不同线程执行，必须要考虑磁盘的情况。也就是需要考虑I/O并行
比如一个线程在读取磁盘的某个分区，而另一个线程在读取同一硬盘下不同的分区，虽然这两个线程在并发执行，但是磁盘的执行效率会很低（一会读这里一会读那里，每次读取都是随机，那么这块硬盘将疲于应对）

如何对I/O进行优化？将数据切分到不同的磁盘中
- 同一数据库，使用不同硬盘存储
- 一个数据库存一个盘
- 一张表存一个盘
- 将表切成不同部分存到一个盘
以上优化，数据库能够感知，也就是需要配置数据库，使数据库主动将数据放到不同的盘中
### Multi-Disk Parallelism
多磁盘并发
使用**RAID磁盘阵列**，在OS层面上将多个磁盘视为一个磁盘（结合成一个逻辑单元）
我们只能看到一块硬盘，但是存储文件后，文件将被切分成多份，落到不同的盘中，这个操作由OS和硬件完成，由于DBMS的存储基于OS的文件系统，所以DBMS作为使用者对这个操作无感
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160929032.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160933149.png)

RAID 0切分粒度很细，将进行文件切分，因为没有冗余，所以数据容易丢失
但RAID 0能够提升并行读和并行写的性能

RAID 1：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160933135.png)
数据冗余，具有可靠性，能够提供并行读效率
但磁盘利用率低

RAID 5：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160935757.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160935916.png)
假设现在只有4个文件页：1，2，4，5，这些文件被分割成两份存到不同硬盘中，这是RAID 0
page3作为page1和page2的异或，page6作为4和5的异或，存储在不同盘中
提供了可靠性，但是写性能较差，需要额外维护异或盘
### Data Partitioning
将不常用且长的数据切开（垂直），存储在不同盘中，能够提供读性能。如果数据库不支持分表，可用在业务层面上手动分表
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160940489.png)

将表水平切分，需要告知数据库切分依据（partitioning key），如
- 按某列的hash值
- 按某列的range
- 或者是按照其他谓词
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407160942744.png)
