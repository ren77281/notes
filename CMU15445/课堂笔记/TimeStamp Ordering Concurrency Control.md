```toc
```
基于时间戳顺序，也是一种实现串行化的方案
二阶段锁是一种悲观的并发控制，在问题出现之前，就阻止问题的发生。同时，锁会导致其他事务的等待，将影响数据库的整体性能表现

TO则是一种乐观的并发控制，时间戳较小的事务必须先提交，否则大时间戳的事务无法提交
怎么确定时间戳：
- 系统时钟
可能存在问题，硬件设备的时钟不精确，需要通过网络手段同步时钟。若同步后的时间变慢，将导致事务的时间戳回溯，使得时间戳不是一个递增的序列
- 逻辑计数器，在分布式系统中存在同步问题
- Hybrid，系统时钟+逻辑计数器

TO无需加锁
给每一个事务赋予一个时间戳，每个对象（一般是tuple）具有两个时间戳：
- R-TS
- W-TS
分别表示上一次读取/修改它的事务时间戳
每次读取/修改事务时，都会将自己的时间戳与tuple的时间戳进行比较，基于**不能操作未来数据**的原则，对数据进行操作
也就是不能操作比自己时间戳大的数据，因TO系统中，时间戳小的事务将先被提交
## Basic T/O
### 读取
如果事务时间戳小于数据的写时间戳，回滚重启自己，以新的时间戳重新执行事务
如果读取了来自过去的数据，需要更新其时间戳（max即可，因为可能被未来的事务读取），并保存副本到本地（因为该数据可能被新事务修改）
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407240823964.png)
### 写入
只要两个时间戳有一个大于事务时间戳，回滚重启——数据被未来事务读过/写过，当前事务不能写入
只有两个时间戳都小于事务时间戳，才能写入，事务需要将数据副本拷贝到本地
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407240825271.png)

Thomas写入规则：如果事务时间戳小于数据写时间戳，此时事务不用回滚重启，因为就算自己写了这条数据，未来也会被修改。而现在未来已经修改了这个数据，当前事务就没必要修改，自己骗自己：把数据往本地写，之后的读取也从本地读
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231935518.png)

存在的问题：
- 长事务饥饿
- 操作数据后需要拷贝，性能开销大
- 无法恢复

无法恢复：如果无法保证当前事务正在对已经提交事务产生的数据做修改，那么当前调度无法恢复
下面的例子：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231948773.png)
T2操作的数据由T1产生，而T1回滚。此时数据库崩溃，我们无法恢复T2，因为T1未提交
而DBMS需要先恢复T1再恢复T2
## Optimistic Concurrency Control
OCC是Basic T/O的优化
三个步骤
**Read**：只读取数据库的数据，对数据库的修改先暂存在本地
为什么需要暂存：为了可重复读
**Validation**：校验，保证小时间戳先发生
用户发送commit，开始校验，此时才会**获取事务时间戳**
检查读写，写写冲突没有构成环
校验存在两个方向：向后校验与向前校验
向后校验：与过去commit事务检查，是否发生冲突，若冲突则回滚重启
我们不能回滚过去事务，因为其已经commit
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407232013266.png)

向前校验：与未来事务有交集的部分校验。由于双方都未提交，可以选择其中一个事务重启
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407232014469.png)
由于未来事务未提交，也就是还未执行完。我们只能校验其已经执行的部分
向前校验中，有两种情况：
1. 未来事务未进入begin阶段，此时当前事务与未来事务是真正的串行调度
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407240907728.png)
2. 未来事务未进入Validate阶段
如果Ti要写入的数据与Tj读取/写入的数据不同（交集为空），那么校验通过
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407240915540.png)

如下图，Ti要写入的数据被Tj读取，若Ti commit，而Tj未commit，虽然Ti不知道未来Tj将执行什么SQL，但是有可能基于错误的数据执行SQL，所以校验失败
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407240848701.png)

**Write**：如果没有冲突，再将本地结果写入数据库
该阶段将锁全表，因为需要写入的数据已经准备好，写入时间短
如果锁行，可能导致未修改的行被未来事务修改的情况，这将使得并发变得复杂
同时也增加了锁管理的开销

什么时候使用OCC：
- 只读事务多
- 事务操作不同数据，冲突少，回滚次数少
数据库大，查询均匀不倾斜（无热点数据）

缺陷：
- 依然需要拷贝数据到本地
- Validate阶段开销大，逻辑复杂
- Write阶段锁全表，如果发生冲入，将是一个完全串行的调度
- 事务在Validate阶段回滚，此时事务已经执行完所有SQL，回滚产生的性能浪费比2PL大。2PL在死锁检测中回滚事务，此时的事务未执行完所有SQL

## 隔离级别
2PL和OCC依然存在幻读问题，它们只考虑了update和select的并发，并没有考虑insert和delete的并发
幻读：第二次读，读取到了不同数据
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407240934548.png)

为什么存在幻读问题？
对于2PL，上行锁保护了行，但是没有保护表，其他事务可以向表新增行
上表锁？粒度太大，性能表现不好

### 谓词锁
对select的where子句的谓词加S锁，对insert，update，delete的where子句的谓词加X锁
个人理解是：加列锁
如下：select后，status列被了S锁，此时insert语句将插入新的status值，必须获取X锁。但是被阻塞，因为执行select的事务未释放status的S锁
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407240942299.png)
### 索引锁
如果列存在索引，对索引加锁。新增tuple后必然更新索引，虽然主表被更新，但是数据的查询需要经过索引（可能存在问题，一般只有点查询会走索引），而索引没有被更新，所以无法查询到新增行
如果列没有，将对所有page加锁，或者直接上表锁
### 间隙锁
MySQL使用间隙锁解决幻读，可以理解为特殊的索引锁。比如要查询age列的max值，假设max值为96，那么97~+∞这段间隙将被锁上
### 隔离级别
如果上层业务允许不可串行化的事务调度，那么我们就没有必要使用可串行化的事务调度，因为这会降低数据库的性能表现
不同的隔离级别，事务能互相看到的数据也不同
可能发生的问题：
- 脏读
- 不可重复读
- 幻读
针对这三种问题，有四种隔离级别
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241010235.png)
Serializable：可串行化，无问题
Repeatable Reads：可重复读，存在幻读
Read Committed：读提交，不可重复读，幻读
Read Uncommitted：脏读，幻读，不可重复读
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241013507.png)

可串行化：执行SQL语句前，需要严格加锁
可重复读：没有索引锁，将导致幻读问题
读提交：提前释放S锁，将导致不可重复读问题
读未提交：所有的查询操作都不使用S锁，将导致脏读问题
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407241019524.png)

大部分数据的默认隔离级别为RC，读提交。这使得上层业务的开发**需要注意**：我可能会读到其他事务刚刚commit的数据，这个数据与我之前读取的数据不同

有些数据库将对事务进行分析，如果发现这是一个只读事务，将以更宽松的条件去校验它