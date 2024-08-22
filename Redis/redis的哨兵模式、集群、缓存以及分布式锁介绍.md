```toc
```
## 哨兵
哨兵机制通过独立的进程体现，redis-sentinel不负责存储，只是对多个redis-server进行监控
redis-sentinel并没有集成到redis-server中，而是一个独立的进程，我们称redis-sentinel为哨兵节点

监控是如何实现的？sentinel和server之间保持tcp长连接，定期发送心跳包检测server是否正常工作

当主server异常时，哨兵将如何处理（故障转移的过程）？
1. 当一个sentinel发现主节点异常，不会产生警报，这是为了防止误判，只有多个sentinel发现主节点异常，才会报警
2. 若主节点确实发生异常，多个sentinel将从多个从节点中挑选出一个主节点，对其执行slaveof no one，并且修改其他从节点的slaveof，使其成为新主节点的从节点
3. sentinel通知客户端新的主节点信息，客户端之后的写操作将对新的主节点进行

哨兵核心功能：
1. 监控server节点
2. 自动故障转移
3. 通知客户端

为什么要使用多个sentinel？
1. 部署多个sentinel提高监控程序的可用性
2. 降低误判的产生（分布式系统中，避免使用“单点”）

最好部署3个sentinel
### 哨兵的部署
由于我只有一台服务器，所以这里用docker隔离环境，部署3个redis-server容器与3个redis-sentinel容器监控redis-server进程
部署docker容器就要编写docker-compose.yml文件
可以只写一份docker-compose.yml文件，对每个redis-sentinel容器设置依赖关系，使所有的redis-server容器启动，redis-sentinel再启动
也可以对sentinel和server分别写两份docker-compose.yml文件，手动控制先启动server再启动sentinel

这里选择手动控制的方式
redis-server的docker-compose文件：
```
version: '3.7'
services:
  master:
    image: 'redis:5.0.9'
    container_name: redis-master
    restart: always
    command: redis-server --appendonly yes
    ports:
      - 6379:6379
  slave1:
    image: 'redis:5.0.9'
    container_name: redis-slave1
    restart: always
    command: redis-server --appendonly yes --slaveof redis-master 6379
    ports:
      - 6380:6379
  slave2:
    image: 'redis:5.0.9'
    container_name: redis-slave2
    restart: always
    command: redis-server --appendonly yes --slaveof redis-master 6379
    ports:
      - 6381:6379

```
创建目录redis-server，并将该文件放入。执行`docker compose up -d`后，docker将启动三个容器，一个主节点，两个从节点
主节点的端口为宿主机的6379，两个从节点的端口为宿主机的6380和6381，并指定从节点从属于的主节点信息(`--slaveof redis-master 6379`，redis会将redis-master解析成容器的IP地址)

redis-sentinel的docker-compose文件：
```
version: '3.7'
services:
  sentinel1:
    image: 'redis:5.0.9'
    container_name: redis-sentinel-1
    restart: always
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel1.conf:/etc/redis/sentinel.conf
    ports:
      - 26379:26379
  sentinel2:
    image: 'redis:5.0.9'
    container_name: redis-sentinel-2
    restart: always
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel2.conf:/etc/redis/sentinel.conf
    ports:
      - 26380:26379
  sentinel3:
    image: 'redis:5.0.9'
    container_name: redis-sentinel-3
    restart: always
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel3.conf:/etc/redis/sentinel.conf
    ports:
     - 26381:26379
networks:
  default:
    external:
      name: redis-data_default

```
同样，创建redis-sentinel目录，并将该文件放入。执行`docker compose up -d`后，docker将启动三个redis-sentinel程序监控已经启动的主节点。并将当前目录下的配置文件挂载到容器中，启动redis-sentinel时将使用这个配置文件

redis-sentinel的配置文件：
```
bind 0.0.0.0
port 26379
sentinel monitor redis-master redis-master 6379 2
sentinel down-after-milliseconds redis-master 1000 # 主节点超过1000ms没有响应，将认为其主观下线
```
表示允许所有IP向26379端口的连接，并且指明监控的主节点信息：
```
sentinel monitor 主节点名 主节点ip 主节点端口 法定票数
```
法定票数：当超过法定票数的哨兵节点认为主节点主观下线时，而可以认为主节点客观下线

redis-master将被docker解析成IP地址。需要注意：docker启动容器时，可以指定容器所处的网络，若两个容器处于不同的网络，redis-master将无法被解析成IP地址。所以配置文件的最后指明了网络信息：
```
networks:
  default:
    external:
      name: redis-data_default
```

执行`docker stop redis-master`，手动停止主节点，模拟主节点陷入异常
执行`docker compose logs`，观察`redis-sentinel`的监控日志
![image.png](https://s2.loli.net/2023/11/07/nEbWqmGsy3hzuSf.png)
可以看到三个监控节点各自的ID，2号节点先发现主节点的异常，先发送sdown主观下线并进行投票
当票数超过了法定票数，所有节点都认为主节点主观下线，则主节点客观下线odown，此时将进行主从节点的切换
1. 多个哨兵节点进行投票，从中选择一个哨兵节点作为leader，由leader负责主从节点的切换
2. leader根据以下规则，选择一个从节点作为新的主节点
  - 优先级：若在redis节点的配置文件中设置了节点的优先级(slave-priority)，那么选择优先级高的节点
  - offset，若优先级相同，此时选择offset最大的节点。offset的大小表示了从节点与主节点之间的数据同步进度，越大表示数据越同步
  - run id，若offset依然相同，此时选择哪个节点都可以。此时选择run id最大的从节点，run id为redis节点启动时，自动生成的一串数字
3. 选择好从节点后，leader将会控制该节点，执行slave no one，使其成为master，并控制其他节点，执行slaveof，使其成为新主节点的从节点
4. 最后通知客户端，告知新的主节点信息

关于leader的选举：哨兵节点之间互相投票，当某个哨兵节点的票数超过一半时将成为leader
第一个发送sdown的节点会将票投给自己，并向其他节点发送拉票请求。在主节点客观下线的情况下，其他节点将直接投票给发送拉票请求的节点
所以一般是第一个发送sdown的节点成为leader

### 总结
1. 哨兵节点不能只有一个，多个哨兵节点将提高系统可用性
2. 哨兵节点的数量最好是奇数个，方便投票，一般3个就够了
3. 哨兵节点不存储数据，所以对运行哨兵节点的服务器配置要求不高
4. 哨兵+主从将解决系统可用性问题，但无法解决“数据在极端情况下丢失”的问题
## 集群
哨兵+主从无法提高数据库的存储容量，要提高数据库的存储容量就要使用集群

广义：多个机器构成了分布式系统，称之为集群，主从和哨兵也是集群
狭义：redis提供的集群模式，用来解决存储空间不足的问题

将数据分成多份，每台机器存储一部分的数据。此时涉及到系统可用性问题，所以需要使用**主从结构**存储这一部分的数据，我们将存储一部分数据的主从结构称为分片
### 数据分片算法
三种主流的分片方式：
1. 哈希求余
对于键值对中的key，通过哈希函数的映射得到其哈希值，将哈希值对N求余（N为分片数量）得到分片下标并将数据存储到对应分片中（这N个分片从0开始编号）
但哈希求余可能带来的问题有：1. 数据倾斜（分片存储的数据不均匀） 2. 难以扩容

2. 一致性哈希算法
哈希求余中，连续的哈希值将存储到不同的分片中，导致扩容时的大量数据移动
一致性哈希算法将连续的哈希值存储到相同的分片中，即某一连续范围内的哈希值对应相同分片
扩容时，划分出一个连续范围作为新的分片，此时只需要迁移这个范围内的数据即可
如：现在有3个分片，每个分片承担1/3的数据，每个分片存储的哈希值范围（假设最大哈希值为1）：0~1/3、1/3~2/3、2/3~1，扩容时将0~1/6的哈希值范围划分给新的机器
但是这样扩容也会导致数据倾斜问题，除非增加与原分片数相同的分片

3. 哈希槽分区算法
该算法将哈希求余与一致性哈希算法结合，先将key进行哈希映射得到一串数字，再将其对16384取模，得到**槽位号**
同时，一致性哈希算法最大的问题是：连续。如果将连续的槽位作为一个分片，将导致扩容时的数据倾斜问题。如何突破连续的限制？
用二进制表示分片的拥有槽位，每个分片使用额外的空间维护自己拥有的槽位，扩容时将原分片的槽位均等地划分给新分片即可

为什么是16384个槽位？
集群中的节点通过心跳包通信以检查是否存在节点异常，心跳包中需要包含节点的槽位信息，16384用二进制表示后将占用2kB的空间，若增加了槽位，在频繁的网络心跳包通信中将是一个不小的开销。若减少了槽位，将导致数据倾斜问题

### 使用docker模拟redis集群

编写generate.sh文件，用脚本完成准备工作：
```
for port in $(seq 1 9); \
do \
mkdir -p redis${port}/
touch redis${port}/redis.conf
cat << EOF > redis${port}/redis.conf
port 6379
bind 0.0.0.0
protected-mode no
appendonly yes
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.30.0.10${port}
cluster-announce-port 6379
cluster-announce-bus-port 16379
EOF
done
# 注意 cluster-announce-ip 的值有变化. 
for port in $(seq 10 11); \
do \
mkdir -p redis${port}/
touch redis${port}/redis.conf
cat << EOF > redis${port}/redis.conf
port 6379
bind 0.0.0.0
protected-mode no
appendonly yes
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.30.0.1${port}
cluster-announce-port 6379
cluster-announce-bus-port 16379
EOF
done
```
执行脚本
```
bash generate.sh
```
将生成以下目录
![image.png](https://s2.loli.net/2023/11/08/PbgRFCJeiLp9Zzl.png)
每个目录都保存着redis服务的配置文件redis.conf
脚本将针对不同的redis服务器编写不同的redis.conf
![image.png](https://s2.loli.net/2023/11/08/1NmWQp9FlabJtI6.png)

格式大致为：
```
port 6379
bind 0.0.0.0
protected-mode no
appendonly yes
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.30.0.101
cluster-announce-port 6379
cluster-announce-bus-port 16379
```
每个redis服务器的配置文件都存在着差异（如IP地址），其中某些字段的含义：
- cluster-enabled yes：开启redis集群
- cluster-config-file nodes.conf：指定存储配置信息的文件名
- cluster-node-timeout 5000：设置节点的超时时间（和主观下线有关）
- cluster-announce-ip 172.30.0.101：节点自身IP
- cluster-announce-port 6379：节点的业务端口
- cluster-announce-bus-port 16379：节点的管理端口

业务端口和管理端口：
- 业务端口用来进行业务的数据通信，如redis的业务端口用来接收和响应客户端的请求
- 管理端口用于监视、管理和配置程序，如redis的管理端口用来更改配置与监控使用情况

编写docker-compose.yml文件：
```
version: '3.7'
services:
  redis1:
    image: 'redis:5.0.9'
    container_name: redis1
    restart: always
    volumes:
      - ./redis1/:/etc/redis/
    ports:
      - "6371:6379"
      - "16371:16379"
    command: redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.101
  redis2:
    image: 'redis:5.0.9'
    container_name: redis2
    restart: always
    volumes:
      - ./redis2/:/etc/redis/
    ports:
      - "6372:6379"
      - "16372:16379"
    command: redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.102
  redis3:
    image: 'redis:5.0.9'
    container_name: redis3
    restart: always
    volumes:
      - ./redis3/:/etc/redis/
    ports:
      - "6373:6379"
      - "16373:16379"
    command: redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.103
  redis4:
    image: 'redis:5.0.9'
    container_name: redis4
    restart: always
    volumes:
      - ./redis4/:/etc/redis/
    ports:
      - "6374:6379"
      - "16374:16379"
    command: redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.104
  redis5:
    image: 'redis:5.0.9'
    container_name: redis5
    restart: always
    volumes:
      - ./redis5/:/etc/redis/
    ports:
      - "6375:6379"
      - "16375:16379"
    command: redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.105
  redis6:
    image: 'redis:5.0.9'
    container_name: redis6
    restart: always
    volumes:
      - ./redis6/:/etc/redis/
    ports:
      - "6376:6379"
      - "16376:16379"
    command: redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.106
  redis7:
    image: 'redis:5.0.9'
    container_name: redis7
    restart: always
    volumes:
      - ./redis7/:/etc/redis/
    ports:
      - "6377:6379"
      - "16377:16379"
    command: redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.107
  redis8:
    image: 'redis:5.0.9'
    container_name: redis8
    restart: always
    volumes:
      - ./redis8/:/etc/redis/
    ports:
      - "6378:6379"
      - "16378:16379"
    command: redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.108
  redis9:
    image: 'redis:5.0.9'
    container_name: redis9
    restart: always
    volumes:
      - ./redis9/:/etc/redis/
    ports:
      - "6379:6379"
      - "16379:16379"
    command: redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.109
  redis10:
    image: 'redis:5.0.9'
    container_name: redis10
    restart: always
    volumes:
      - ./redis10/:/etc/redis/
    ports:
      - "6380:6379"
      - "16380:16379"
    command: redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.110
  redis11:
    image: 'redis:5.0.9'
    container_name: redis11
    restart: always
    volumes:
      - ./redis11/:/etc/redis/
    ports:
      - "6381:6379"
      - "16381:16379"
    command: redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.111
networks:
  mynet:
    ipam:
      config:
        - subnet: 172.30.0.0/24

```
执行命令，启动容器
```bash
docker compose up -d
```

redis.conf文件写的是docker生成的容器中，redis-server的相关配置
而docker-compose.yml文件写的则是容器的相关配置，需要将容器的端口与宿主机的端口进行映射（注意这些端口不能冲突）

服务端启动后，需要执行一条特殊的命令启动客户端，这条命令只用执行一次
```bash
redis-cli --cluster create 172.30.0.101:6379 172.30.0.102:6379 172.30.0.103:6379 172.30.0.104:6379 172.30.0.105:6379 172.30.0.106:6379 172.30.0.107:6379 172.30.0.108:6379 172.30.0.109:6379 --cluster-replicas 2
```
执行完该命令后，redis集群才算真正建立完成。解释：
- --cluster create：表示建立集群，需要填写redis-server的IP和端口
- --cluster-replicas：表示一个分片中的主节点需要2个从节点
注意，如果redis-server的数量不能使所有主节点都拥有2个从节点，那么该命令无法执行

执行该命令后，打印的调试信息：
![image.png](https://s2.loli.net/2023/11/08/LSzAUlVRqsQ8gkh.png)
redis将给出集群的配置情况，觉得没问题后输入yes即可
通过这些配置情况，可用验证之前所说的哈希槽分区算法，集群中一共有9个节点，每个分片中一主两从，所以一共有三个主节点
一共有16384个槽位，三个主节点平均分摊这些槽位：0 ~ 5460、5461 ~ 10922、10923 ~ 16383
之后的信息表明了从节点与主节点之间的隶属关系
![image.png](https://s2.loli.net/2023/11/08/TencK71hR4DtoPv.png)

连接上9个节点中的任意一个节点
```bash
redis-cli -p 6371
```
执行命令，查看当前集群的配置情况
```bash
cluster nodes
```
![image.png](https://s2.loli.net/2023/11/08/eBgZiXVEFOk5ruQ.png)

此时插入数据
![image.png](https://s2.loli.net/2023/11/08/6sWBfYTymzAkRop.png)
redis报错，因为这个数据的key不属于当前主节点所在的分片，需要用客户端连接对应主节点所在分片，才能插入该数据
在启动客户端时，加上`-c`选项，redis将自动连接/跳转到key对应主节点的客户端中并执行插入
![image.png](https://s2.loli.net/2023/11/08/uaGjSmFU8lT6EeX.png)
此时位于的客户端也发生了变化，若当前节点是从节点，那么redis将跳转到正确的主节点并执行插入操作

但是，如果一条命令中包含了多个key，而这些key分散在不同的分片中，那么redis将无法执行这条命令，我们只能将命令拆解，使每个命令包含的key位于同一分片，依次执行命令（或者使用特殊方法hash tag解决）

### 集群的故障处理
cluster nodes打印的信息中，如果出现fail，则说明该节点出现异常
![image.png](https://s2.loli.net/2023/11/08/uh9cKIBx46Uk8wt.png)
FAIL状态的节点重启后，将成为从节点，无论其之前是主节点还是从节点

如何监控节点是否出现异常？
集群中的监控机制：
1. 节点之间互相发送心跳包，发送ping包，对方返回pong包。心跳包中包含了集群的配置信息（节点的id，主节点or从节点，属于哪个分片，持有哪些槽位）
2. 每个节点，每秒中都会向一小部分节点发送心跳包。而不是向所有节点发送心跳包，因为这将占用大量的网络带宽
3. 若发送方在超时时间内未得到接收方的响应，将尝试重置与其的tcp连接。若连接失败，将认为其主观下线（PFAIL状态）
4. 若发送方发现数量超过一半的节点也认为该节点主观下线，将认为其客观下线（FAIL状态），同时发送方将同步该消息给其他节点，使其他节点将该节点状态也设置为FAIL

故障迁移：
- 如果异常节点是从节点，那么就不需要进行故障转移
- 如果异常节点是主节点，那么就要进行故障转移

故障迁移的具体过程：
1. 从节点判定自己是否有资格成为主节点，根据主从节点之间的数据同步情况进行资格判定。若从节点太久没有与主节点进行数据同步，那么该节点不具有资格
2. 具有资格的节点将休眠一段时间，休眠时间=500ms基础时间+\[0, 500ms]随机时间+排名 \* 1000ms。排名由数据同步情况决定，越同步则排名越靠前
3. 先被唤醒的从节点将向其他分片中的主节点发送拉票请求（只有主节点拥有投票权）
4. 主节点进行投票，从节点的票数超过了主节点数目的一半，该从节点将成为分片中的主节点（自己执行slaveof no one，并让其他从节点执行slaveof操作）
5. 成为主节点后，该节点需要将信息同步给其他分片

注意集群和哨兵的故障迁移区别：哨兵选举主节点时，会先选出一个leader，再选出主节点。而集群则直接选举主节点

对于以下三种情况，将认为整个集群都宕机：
1. 分片中的主节点和从节点都挂了
2. 分片中主节点挂了，但分片中没有从节点
3. 整个集群中，同一时刻超过半数的主节点挂了

主要是根据slots是否可用来判定整个集群的宕机，所以某个分片出现异常将影响整个集群

### 集群扩容
向集群中添加主节点
```bash
redis-cli --cluster add-node 新节点的IP:port 集群中任意一节点的IP:port
```
该命令需要指定新节点的IP与端口，同时指定集群中任一节点的IP与port，这是为了指定集群的信息

执行添加节点的命令：
![image.png](https://s2.loli.net/2023/11/09/oTxXycUptKfOaDG.png)

随便连上一个节点，查询集群配置情况
![image.png](https://s2.loli.net/2023/11/09/gY4VkL6pU52owBZ.png)
可用看到集群中的主节点从3个扩容到了4个，并且新添加的节点为主节点
但是该主节点没有slots，即不存储任何数据，我们要分配slots给该节点
```bash
redis-cli --cluster reshard 集群中的任一节点IP:port
```

reshard表示重新切分，注意拼写：`shard`不是`shared`
告知集群中的任意节点的信息以告知集群的信息
执行后，redis会询问以下信息：
1. 需要为新节点重新切分多少的slots
2. 由哪个节点接收重新切分的slots
3. 从哪些节点上重新切分slots
  - 写all表示从所有主节点中重新切分
  - 还能指定其他主节点的ID，最后写done表示指定的结束
![image.png](https://s2.loli.net/2023/11/09/VjoL3MEsGPIhquZ.png)

最后redis将打印哪些槽位被重新切分，并让我们确定是否要这样扩容
![image.png](https://s2.loli.net/2023/11/09/nuqNEL4apkd6ZjO.png)
输入yes，完成扩容
连接集群中的任一节点，查询集群的配置信息
![image.png](https://s2.loli.net/2023/11/09/JOm36DAjTsucVfH.png)
可用看到新节点的槽位信息，一共是4096个槽位，并且这些槽位都来自由旧节点的靠前槽位

为新节点分配槽位后，还需要为其分配从节点：
```bash
redis-cli --cluster add-node 从节点IP:port 集群中任一IP:port --cluster-slave --cluster-master-id 主节点ID
```
![image.png](https://s2.loli.net/2023/11/09/Ijr2oPyZpBVgwTK.png)
至此，集群扩容的操作完成
总结下，大概是三步：
1. 添加节点
2. 重新切分slots给新节点
3. 为新节点添加从节点
## 缓存
>通常将redis作为mysql的缓存

**为什么关系型数据库的性能不高？**
从硬件角度说，数据库将数据存储到硬盘上，而硬盘的IO速度很慢，特别是随机访问。同时，如果查询操作无法命中索引，那么就需要遍历整张表，此时将进行大量的硬件IO

从软件角度说，关系型数据库在插入数据时，需要进行一系列的解析，优化，校验操作
。如果是复合查询，效率将降低更多

mysql数据库的效率较低，一旦请求的数量增加，将导致数据库的压力增加，当请求消耗的资源超过了系统的资源上限时，将引发宕机

那么**如何提高mysql的效率？**
开源：使用mysql集群
节流：使用高性能的数据库作为mysql的缓存，存储热点数据，让缓存承担大部分压力

### 缓存的更新策略
如何判断数据是否为热点数据？
1. 定期更新：将用户查询的数据以日志的形式记录下来，根据查询次数最多的数据，将数据对应的结果写入到缓存中。通常编写脚本完成定期更新的任务
2. 实时更新：用户查询的数据，如果在redis中，则直接返回。如果不在redis中，从数据库中查询并**写入**到redis中

随着实时更新的数据写入，缓存中的数据越来越多。而redis的存储空间有限，需要需要使用内存淘汰策略不断地更新内存

几种常见的更新策略：
1. FIFO，先进先出，缓存中存在时间最久的数据将被淘汰
2. LRU，缓存中最久未使用的数据将被淘汰
3. LFU，缓存中访问次数最少的数据将被淘汰

可以在redis配置文件中设置相关字段：
- volatile-lru：当内存不足以容纳新写入数据时，从设置了过期时间的key中使用LRU（最近最 少使用）算法进行淘汰 
- allkeys-lru：当内存不足以容纳新写入数据时，从所有key中使用LRU（最近最少使用）算法进行淘汰
- volatile-lfu：4.0版本新增，当内存不足以容纳新写入数据时，在过期的key中，使用LFU算法进淘汰
- allkeys-lfu：4.0版本新增，当内存不足以容纳新写入数据时，从所有key中使用LFU算法进行淘汰
- volatile-random：当内存不足以容纳新写入数据时，从设置了过期时间的key中，随机淘汰数据 
- allkeys-random：当内存不足以容纳新写入数据时，从所有key中随机淘汰数据
- volatile-ttl：在设置了过期时间的key中，根据过期时间进行淘汰，越早过期的优先被淘汰. (相当于 FIFO, 只不过是局限于过期的 key)

### 缓存带来的一系列问题
redis和mysql的关系：redis抗压能力比较强，站在mysql身前为mysql抗下大部分的压力，对于mysql只需要承担少部分的压力。redis就像mysql的保护伞，保护着mysql。当redis的保护能力降低，mysql的运行将受到极大的威胁
以下是redis掉链子的几种情况：

1. 缓存预热（Cache preheating）：
首次使用redis作为缓存，或者redis中大量key过期时，redis几乎不存储数据。此时用户将直接访问mysql，给mysql带来极大压力，极有可能导致mysql的崩溃
所以此时需要对热点数据进行合理的预测（基于之前的热点数据），将合理的数据存储进redis，以降低mysql的压力
2. 缓存穿透（Cache penetration）：
访问的数据在redis和mysql中都不存在，此时用户的请求将穿过redis递达mysql，但是在mysql中依然无法得到对应数据。所以这是一个非法的请求，当出现大量非法请求时，不仅mysql的压力会骤增，连redis的压力也会骤增。但理论上mysql会先崩溃
为什么会存在缓存穿透？大概率的原因是业务设计的不合理，比如缺少必要的参数校验环节，导致非法数据混了进来。小概率原因是误删操作
如何解决？在业务中增加对请求数据的严格校验（也能使用布隆过滤器对数据进行校验）。对于不存在的数据，也存储在redis中（存储一个无效值），以减轻mysql的压力
3. 缓存雪崩（Cache avalanche）：
短时间内，redis中的**大量数据**失效，导致所有请求直接访问mysql，mysql的压力骤增
为何数据会突然失效？redis节点挂了，或者redis中的key同一时间过期
如何解决？提高redis的高可用性，完善故障转移机制。不设置过期时间或者为过期时间设置随机因子
4. 缓存击穿（Cache breakdown）：
翻译的有问题，更应该将缓存瘫痪（或者崩溃？）。缓存雪崩的特殊情况，某个热点数据失效，当大量请求同时访问该数据时，mysql的压力骤增（缓存击穿只影响**热点数据**）
如何解决？进行必要的服务降级。使用分布式锁，限制数据库的访问频率
或者设置热点数据永不过期

## 分布式锁
分布式锁本质是一个/一组单独的服务器进程，给其他服务器提供加锁服务，redis就是一种典型的用来实现分布式锁的方案
除了使用redis作为分布式锁，还能使用MySQL，Zookeeper作为分布式锁

加锁时，往redis中设置一个特殊的key-value。解锁时，删除这个key-value
当其他服务器想要访问服务时，发现该服务的锁（key-value）已经存在（或者是尝试设置key-value但是设置失败），此时服务器就无法使用加锁的服务
加锁往往使用redis中的`setnx`命令：不存在就创建，存在则报错

问题一：
加锁的服务器没来得及解锁就崩溃了，重启后却忘记自己曾经加过锁。那么这把锁就永远无法释放，如何解决这样的问题？**设置过期时间**
使用`set ex nx`命令完成过期时间和锁的设置。注意：不能分别使用setnx和expire加锁和设置过期时间，因为这样设置无法保证**原子性**

问题二：
服务器A执行了加锁（设置kv），虽然服务器B无法加锁，但是可以解锁（删除kv）。这样的话，加锁无效，如何解决这样的问题？
引入校验机制。给每个服务器一共唯一的身份标识，如key表示资源的唯一标识，而value表示服务器的唯一标识，删除kv时，redis需要判定value是否和执行删除命令的服务器的唯一标识相同。只有相同才会删除，否则不执行删除

问题三：
解锁校验时，判定身份与实际的删除操作不是原子的。当服务器使用多线程时，线程A和线程B同时执行解锁操作。两者通过了先后身份的判定，接着先后执行删除操作。线程A成功解锁，此时又有一个线程C重新加锁，准备访问服务，但是线程B将其解锁。如何解决这个问题？
使解锁操作具有原子性。将判定身份和删除kv的操作打包成一个事务
通常使用lua脚本编写程序，作为redis的事务，而不直接使用redis的原生事务，因为lua脚本执行的功能更多，且能写出逻辑性更强的命令

### 看门狗
对于问题一，设置过期时间为多长合适？用于时间不好掌控，这里引入看门狗来动态续约
使用一个专门的线程负责设置过期时间（续约），这个线程被称为**看门狗**
看门狗将先设置一个不长不短的过期时间，当快要过期时判定是否要续约（服务是否访问完成）。如果需要续约，则再设置一个时间，这个时间可以根据当前进度灵活调整。直到服务访问完成，看门狗将不再续约
同时，若当前服务器崩溃，这台服务器上的看门狗也就无法续约，超过过期时间，锁将自动释放

### redis的redlock
作为分布式锁的redis，是否有可能崩溃？如果崩溃了，服务器就无法使用锁（加锁出现错误）
虽然有哨兵机制对节点进行监控，若主节点崩溃，从节点将成为主节点，但是从节点的数据和主节点完全同步吗？是否有可能丢失了加锁数据？
对于这样的场景，后续引起的bug是十分复杂的，如何解决redis节点崩溃问题？
官方给出了一个算法：redlock
由多个节点负责一把锁，比如使用5个节点负责一把锁，加锁时需要按照顺序对这5个节点设置kv。当超过一半的节点被成功设置了kv，我们就能认为加锁成功。此时某个节点挂了对最后的结果影响不大，并且节点挂了也是一个小概率事件
同理，解锁时将按照顺序对节点进行依次解锁

这样，就能保证redis作为分布式锁的高可用性