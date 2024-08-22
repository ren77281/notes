```toc
```
## 前言
redis最重要的概念：在内存中存储数据
为什么要设计一个在内存中存储数据的数据库，定义的变量天然就存储在内存中，为什么不直接定义变量？
另一个重要的概念：分布式，在单机系统中，使用redis显得多此一举，只有在分布式系统中使用redis才能发挥它真正的性能

可以这样理解：redis基于网络，将内存中的变量，分享给其他进程甚至其他主机的进程使用

为什么不直接使用MySQL？
MySQL最大的问题在于**慢**，因为它使用硬盘进行存储。而某些应用对于速度相应的要求极高，此时只能使用redis

而redis最大的劣势在于**内存有限**，能存储的数据量小。将redis和MySQL结合，将20%的热点数据用redis存储，而80%的非热点数据用MySQL存储，就能使结合后的数据库的又大又快。此时redis相当于MySQL的cache，这也是冷热分离架构的关键，但是这样做的代价就是大大提升了系统的复杂程度

（redis最开始的目的是研发一个消息中间件**streaming engine**，即分布式系统下的**生产消费**模型。但随着慢慢地发展，大家发现将redis作为数据库来使用是更好的选择。即使redis也支持消息中间件的功能，但市面中存在更多专业的中间件供我们使用，因此对于redis消息队列相关的知识不作为重点学习）

以上文字中，“分布式”被频繁提及，事实也是如此：要谈Redis就离不开分布式
## 关于分布式系统
要谈分布式就离不开互联网产品架构的演进过程
早期的单机架构：应用服务和数据服务部署在一台主机上，而一台主机的资源有限，包括但不限于：CPU、内存、磁盘、网络...随着用户的请求增多，主机的资源不够使用时，如何处理？
两个方面：1. 开源：简单粗暴，增加更多的硬件资源 2. 节流：从软件上优化，找到性能凭借并解决，节省软件的资源消耗（难）

对于开源，虽然增加硬件资源更加容易，但是一台主机硬件资源存在上限，这个上限由主板的扩展能力决定
既然垂直扩展不行，那就水平扩展，水平扩展就是引入**更多的主机**，而主机一多，就需要在软件上进行相应的调整和适配
围绕着软件的调整和适配，互联网应用的架构就开始了演进，分布式系统架构就属于演进中的关键一环（不太准确的说：就是使用多台主机/节点存储数据）。至于具体的演进，这里不再赘述，有兴趣可以看我写的上一篇文章
## Redis特性
根据官网给出的资料：
- 使用内存存储数据结构（In-memory data structures）：Redis作为非关系型数据库，用数据结构存储数据。数据结构一般都是键值对，以string作为key，string, hash, list, set, zset...作为value。相比于关系型数据库以“表”的方式存储数据，非关系型数据库的存储方式显然更简单
- 可编程（Programmability）：对于Redis，可以使用简单的命令进行交互，也可以使用脚本语言Lua执行一些带有简单逻辑的操作
- 可扩展（Extensibility）：Redis提供了一组API，可以使用C，C++，Rust语言调用这些API。因此我们能自主地扩展Redis的功能（本质上是使用API编写动态链接库）
- 持久化（Persistence）：Redis主要使用内存存储数据，并且以硬盘作为辅助。Redis会将内存中的数据持久化到硬盘中，若系统发生问题导致了重启，那么Redis会从硬盘读取备份数据
- 集群化（Clustering）：类似于分库分表，可以部署多个Redis节点用来存储数据，每个节点存储整体数据的一部分
- 高可用（High availability）：即数据的备份。Redis支持主从结构，主节点崩溃，从节点马上顶上代替主节点，这个过程用时极短，用户甚至感知不到

最重要的一个特性是：快，为什么？
1. Redis存储在内存中
2. Redis作为非关系型数据库，核心功能的逻辑简单，执行效率高
3. 从网络上看，Redis使用IO多路复用（epoll）
4. Redis为单线程模型，减少了不必要的线程竞争开销（为什么Redis中，单线程比多线程快？多线程更快的场景为：CPU密集型任务，此时多线程能充分地利用CPU多核资源。而Redis的任务只是简单的操作数据结构，不是CPU密集型任务，使用多线程反而会变慢）
## Redis应用场景
- 实时数据存储（Real-time data store）：将Redis作为数据库使用。对于大多数应用，它们的数据存储需要优先考虑“大”，但是对于一部分应用，它们的数据存储需要优先考虑“快”。如搜索引擎对于性能的要求非常高，需要将所有的数据存储在内存中，此时可用考虑Redis
- 缓存与会话存储（Caching & session storage）：在冷热分离架构中，Reids作为MySQL的辅助，存储热点数据以提高访问热点数据的速度。此时Redis只是一个辅助，若Redis崩溃了，数据还能从MySQL中恢复。会话存储：访问web应用时，本地浏览器用cookie保存了用户的身份信息，而远端服务器用session存储用户信息，只有两者的信息匹配，才能成功验证用户信息。但在应用服务集群中，每次访问的应用服务器可能都不相同，此时将出现一台服务器保存了session，而其他服务器没有保存session。为了减少用户的验证次数，采用Redis保存用户session，服务器从Redis中更新用户session
- 消息队列（Streaming & messaging）：基于Redis的消息队列可以实现一个基于网络的生产消费模型。若当前应用中已经使用了Redis，又不想引入其他的中间件时，可以考虑Redis的消息队列

 Redis不能做的事：存储**大规模**数据
## Redis5安装
安装
```bash
apt install -y redis
```
验证
```bash
redis-server --version
netstat -nltp | grep redis
```
![image.png](https://s2.loli.net/2023/10/30/DsbY6QWF9GmqfzI.png)

发现redis只开放了本地环回的端口，意味着不能从外网连接redis服务，这里修改配置文件，使外网IP也能访问redis服务
```bash
cd /etc/redis
```
![image.png](https://s2.loli.net/2023/10/30/F96I3ZSLBatvOWy.png)

其中的`redis.conf`就是redis的配置文件（通过配置文件开启/关闭/设定软件的某些功能）
编辑该文件
```bash
vim redis.conf
```
找到bind这句话
![image.png](https://s2.loli.net/2023/10/30/Jfijz9QO2F7qP38.png)
将127.0.0.1修改成0.0.0.0即可
![image.png](https://s2.loli.net/2023/10/30/zJnI2Nk9fUSMQL5.png)
再找到protected-mode，将yes修改为no
![image.png](https://s2.loli.net/2023/10/30/kJUHN7MYD1eVXTa.png)

最后重启服务器使配置生效
```bash
service redis-server restart  # 重启redis服务
service redis-server status   # 查看redis运行状态
```
![image.png](https://s2.loli.net/2023/10/30/jITWbBvZOmnrG8X.png)
active（running）表示正常运行，此时修改配置文件成功

使用redis客户端连接服务器，输入以下语句即可
```bash
redis-cli
```
![image.png](https://s2.loli.net/2023/10/30/8LoSmekUj2bOMJV.png)
输入ping，返回PONG说明连接成功
ctrl+d退出redis客户端
## redis命令
同mysql一样，redis也是客户端-服务器程序
redis的快体现在：与关系型数据库相比（如MySQL），redis快很多。但是与直接在内存中定义数据结构存储数据相比，redis就慢得多了。因为redis是客户端服务器程序，相比于直接操作内存，redis还需要先进行网络传输以获取操作命令

通过官网[Redis](https://redis.io/)中的搜索框，输入命令就能查看命令的相关文档
![image.png](https://s2.loli.net/2023/10/30/UHyJpLC6i2WZdtg.png)
### 最核心的两个命令：get和set
> 哈希表怎么用，redis就怎么用

set：将键值对存入数据库
```bash
set key value
```
![image.png](https://s2.loli.net/2023/10/30/4mTdcetHOkwNGzE.png)

对于key和value，没有必要加**引号**

get：根据key取value
如果kv不存在，将返回nil
![image.png](https://s2.loli.net/2023/10/30/gxTqfB62U9YekZm.png)

### keys
key固定为字符串，但value有多种类型，常见的有五种：字符串、哈希表、列表、集合、有序集合，操作不用的数据结构，就有不同的命令。全局命令则能操作任意数据结构，keys就是一个全局命令

keys：通过一些特殊符号（通配符）描述key，匹配描述的key将被返回
```bash
keys pattern
```
pattern用来描述key需要符合的**条件**
- ？匹配一个字符
- \*匹配一个/多个字符
- \[aef\]只能匹配a或者e或者f（给出固定的选项）
- \[^e\]除了e都能匹配（排除e）
- \[a-e\]a到e之间的字符都可选，包含a和e

![image.png](https://s2.loli.net/2023/10/30/jxmRrcDupQO1oHV.png)

![image.png](https://s2.loli.net/2023/10/30/M9OfFGiWcyaC4NS.png)

![image.png](https://s2.loli.net/2023/10/30/ICPl9J5zT2awtWy.png)

keys的时间复杂度为$O(n)$，生产环境中进行使用keys，特别是`keys *`
因为redis是单线程程序，keys将导致redis被阻塞，使得MySQL的负载升高，可能导致redis和MySQL都一起挂掉，甚至系统崩溃
### exits
返回key存在的个数（针对n个key，只会返回0~n之间的数）
```bash
exits key [key ...]
```
时间复杂度$O(n)$，n为输入的key的个数
redis的很多命令都支持多个key/完成多个操作，目的是为了减少网络IO次数
![image.png](https://s2.loli.net/2023/10/30/isLac74BwhVD3J5.png)
### del
一次删除一个/多个key，返回删除key的个数
```bash
del key [key, ...]
```
时间复杂度$O(n)$，n为输入的key的个数
![image.png](https://s2.loli.net/2023/10/30/oHVbJ2BjAQUXxkE.png)
若将redis作为缓存，删除个别数据对整体的业务影响不大，因为redis可以从mysql中重新读取数据。但是依然建议，对于删除操作要小心谨慎
### expire
给key设置过期时间（单位：秒），存活时间超过这个值就删除key（应用场景如验证码，基于redis实现分布式锁）
```bash
expire key seconds
pexpire key 毫秒
```
时间复杂度$O(1)$
返回值为1表示成功，0表示失败
注意：对key设置expire之前，key必须**存在**，否则将设置失败

### ttl
time to live，查看当前key的过期时间，单位：秒
pttl，单位：毫秒
```bash
ttl key
```
时间复杂度$O(1)$
返回值：剩余过期时间，-1表示没有关联过期时间（无穷大），-2表示key不存在
### redis中key的过期策略
两个策略相结合：
1. 定期删除：每次抽取一部分key，并保证将这些key全部遍历一遍的时间足够快（单线程）
2. 惰性删除：时间到了但不删除这个key，用户下次服务这个key时触发删除同时返回nil

然而redis并没有采用定时删除的策略，可能因为引入了定时删除就需要引入多线程
（定时删除策略的实现方式有两种：1. 时间轮 2. 优先级队列）
### type
返回key对应value的类型（none, string, set, zset, list, hash)
```bash
type key
```
![image.png](https://s2.loli.net/2023/10/30/ohAlwt51igVUYWE.png)
## redis数据类型的内部实现方式
redis会根据value的具体数值，采用不同的内部类型进行存储。如string有多种内部类型，但是这些内部类型暴露给外部的接口都是相同的

- string
  - raw：redis实现的最基本字符串
  - int：当value就是整数类型时，redis直接使用int进行存储。int通常也用来实现一些特定功能（如计数）
  - embstr：针对短字符串进行的特殊优化
- hash
  - hashtable：redis实现的最基本哈希表
  - ziplist：哈希表中，元素比较少时，使用ziplist（压缩列表）节省空间
- list
  - linkedlist：redis实现的最基本list
  - ziplist
  - 从redis3.2开始，引入quicklist代替linkedlist和ziplist，兼顾两者的优点
- set
  - hashtable
  - intset：集合中都是整数
- zset
  - skiplist：跳表，查询效率为$O(logn)$
  - ziplist

使用 
```bash
object encoding key
```
查看key对应value的**内部类型**
![image.png](https://s2.loli.net/2023/10/30/xyAIE3kZW6tDPYQ.png)

## redis的单线程
redis的单线程并不是说redis内部真的只有一个线程，只是说redis只用一个线程处理所有的命令请求。redis中也有多线程，如网络IO时采用了多线程

**为什么redis能够使用单线程**？核心业务简单，不消耗cpu资源

若两个客户端“同时”发起一个命令，两个命令相同，都是对某个变量进行自增。虽然两个客户端几乎同时发起命令，但是它们的到达时间存在先后。对于单线程，先到达的先执行，显然不存在线程安全问题

**为什么redis比其他数据库效率高？速度快？**（和关系型数据库MySQL，Oracle对比）
1. redis访问内存，其他数据库访问磁盘
2. redis的核心功能比其他数据库简单。关系型数据库支持了更复杂的功能，比如约束。为了维护这些功能，每次的操作都会导致额外的开销
3. redis的单线程模型，避免了不必要的线程竞争开销，由于redis的**核心功能**简单，不消耗cpu资源，采用单线程足以满足需求
4. redis处理网络IO时，使用epoll这样的IO多路复用机制（在TCP连接中，使用一个线程管理多个socket。因为同一时刻只有少数socket是活跃的，所以可以充分利用等待socket进行IO的时间）

