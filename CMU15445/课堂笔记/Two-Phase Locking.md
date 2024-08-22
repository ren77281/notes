```toc
```
上一节探讨了调度的Serializable，但是判断是否Serializable却是在事务commit后
我们需要在调度时，确定调度是否可串行化，保证数据一致性

二阶段锁是一种解决方案：
lock表示宏观概念上的锁，行锁，表锁，获取了lock之后我们还需要获取latch，对底层结构加锁（如哈希表的bucket，缓存池的page_guard），读写锁
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231507633.png)

对于lock的“读写锁”：共享锁和排他锁
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231510448.png)

事务在修改/读取数据前，需要先获取锁，但是加锁后，不可串行化调度就会成为可串行化调度吗？比如下面这个例子：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231520726.png)
这是一个不可串行化的调度，普通的加锁并不会改变什么
我们需要使用二阶段锁，将不可串行化转换成可串行化
## Tow-Phase Locking
似乎是因为加锁粒度不同？
第一个阶段：Growing，只能加锁，不能解锁
第二个阶段：Shrinking，只能解锁，不能加锁

增加锁的粒度后，就能将不可串行化调度转换
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231534208.png)

依然存在的问题：**级联回滚**
### 级联回滚
只要事务未提交，且调度存在冲突，无论采取什么样的方法，都会存在级联回滚问题
也就是读未提交：脏读

强二阶段锁：将锁的释放集中，释放所有锁后，直接commit。还是说在commit后才执行第二阶段？
SS-2PL和2PL的比较：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231557285.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231557092.png)
SS-2PL其实就是在2PL的基础上继续增加锁的粒度
如果所有的事务都在操作同一行数据，那么SS-2PL就退化成了真正的串行执行

如果一个事务写入的值不会被读取或者重写，直到该事务commit。我们称这样的调度是严格的
在SS-2PL控制下，回滚事务很方便。因为当前事务写入后，没有其他事务的修改，所以我们直接回滚到该数据的上一版本即可
以下是不同调度类型之间的关系：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231610771.png)

除了级联回滚，2PL还存在死锁问题，SS-2PL也存在死锁问题
### 死锁
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231611457.png)
2PL在未进入Shrinking阶段时，不会释放锁。SS-2PL不处理完数据，也不会释放锁。两者都会导致死锁问题

有两个解决方案：死锁检测与死锁预防
#### 死锁检测
维护锁的等待图：以事务为节点，有向边表示锁的等待
数据库将周期性地检查这张图，如果存在环，则需要将其解开
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231617775.png)
发现死锁：选择一个事务，将其回滚
如果是上层事务，不需要重启，上层会有执行事务失败的逻辑
如果是自己的事务，则需要重启（如已经设置好的定时任务，触发某些SQL语句）

我们需要平衡死锁检测与死锁导致的阻塞
如何选择事务：
- 执行时间
- 已经执行的SQL数量
- 持有锁的数量
- 回滚影响的事务数量
- 被回滚的次数

回滚也分彻底回滚与最小回滚
#### 死锁预防
根据事务执行时间，赋予事务优先级，越早执行的事务优先级越高，越老
有两个思路：
1. 老事务可以等待年轻事务，年轻事务不等待老事务，直接回滚
2. 老事务抢夺年轻事务的锁，并使其回滚（老事务不等待）。年轻事务可以等待老事务
为什么这样的策略能预防死锁？
不存在循环依赖，永远是老等年轻，或者年轻等老
被回滚的事务，优先级不需要改变，依然是第一次执行的时间戳，否则将造成饥饿问题
### Lock Granularity
当事务想要获取锁时，锁的粒度应该有DBMS决定
需要权衡锁的数量与锁的粒度，更少的锁，更粗的粒度。更多的锁，更细的粒度
锁的粒度越粗，数据库的性能越低
但是加锁解锁的开销，也会降低数据库的性能
一般最细的粒度就是行锁
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231646766.png)

获取表锁时，需要先判断是否存在行锁，如果表特别大，直接遍历所有行并判断是否存在行锁的开销将十分巨大
在表上保存信息，记录表内的行是否被锁
这样的信息，叫做**意向锁**
### Intention Lock
意向锁表明当前结构之下的低层结构被加了锁
意向锁的三种类型：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231656766.png)

SIX：用共享锁锁了整张表，同时用排他锁锁了表里的几行数据
意向锁的兼容矩阵：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231745484.png)

获取S，IS时，必须持有父节点的IS
获取X，IX，SIX时，必须持有父节点的IX
这种分层锁，可以有效地减少锁的数量，提高锁的管理性能

当获取太多低层级锁时，锁管理器将自动请求高层级锁，以减轻锁管理器的工作压力
意向锁对于用户是透明的，DBMS将根据SQL语句，自动进行锁的申请与释放
但是有些数据库依然提供了锁的接口给用户使用，用户可以显式加锁以改善/提高并发性能
表锁：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407231843886.png)



