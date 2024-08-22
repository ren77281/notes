
```toc
```
## string
redis的string，直接按照二进制（不做任何的转换，存的是什么取出来的依旧是什么）的方式存储。所以string不仅仅可以存储文本数据，还可以存储整数，JSON，xml甚至音视频。但是string的大小最大为512M

二进制存储：减少了存取数据时的**编码转换**和乱码问题
### set和get
如果key不存在，则创建新的键值对。如果key存在则让新的value覆盖旧的value，可能会改变原有的数据类型，此时原有key的ttl将失效
```bash
set key value [expiration EX seconds | PX milliseconds] [NX|XX]
```
比如
```bash
set k1 1
expire k1 10
```
等价于
```bash
set k1 1 ex 10
```
px设置的超时时间单位为毫秒
nx：如果key不存在，则设置value（相当于新建）。存在则不设置（返回nil）
xx：如果key不存在，则不设置value。存在则设置（相当于更新）

get没有扩展使用，只需要注意key对应的value必须是string，否则get将报错
### mset和mget
```bash
mset key value[key value...]
mget key [key]
```
### setnx，setex，pset
```bash
setnx key value
setex key seconds value
psetex key milliseconds value
```
### incr和incrby
incr               针对value+1
incrby           针对value+/-n
decr              针对value-1
decrby           针对value+/-n
incrbyfloat     针对value+/- 小数
这些操作的时间复杂度为$O(1)$
```bash
incr key
```
key必须是整数，若+1后整数值超过了64位，那么将报错
返回值是+1后的结果
![image.png](https://s2.loli.net/2023/10/30/gt9rxIapoLdO7yN.png)

若incr的key若不存在，那么默认set一个value位0的key，并进行incr操作
![image.png](https://s2.loli.net/2023/10/30/d2D1kBejUum7yTO.png)

incrby key n，n可以是负数
![image.png](https://s2.loli.net/2023/10/30/lcUkW1tEJsm8oY4.png)

### append
```bash
append key value
```
若key已经存在，则将value拼接到原有value后面。若key不存在，那么append等同于set
时间复杂度$O(1)$，返回值是设置完成的value长度
![image.png](https://s2.loli.net/2023/10/30/IubPtyOL8nofMqv.png)
当前使用的终端，使用utf-8编码格式，其中一个汉字占三个字节
由于redis的string不会对字符做任何处理，所以“你好”在string中占用6个字节
get k2则获取“你好”的二进制编码，其中\\x表示十六进制，可以看到一共是12个十六进制数，也就是6个字节

在启动redis客户端时，加上--raw选项，就能使redis客户端自动对二进制数据进行翻译
![image.png](https://s2.loli.net/2023/10/30/QGAlNxLfe3bhm5g.png)
### getrange
```bash
getrange key start end
```
start和end为字符串的下标
返回指定区间（左闭右闭）的字符串，可以使用负数，如-1表示倒数第一个字符，-2表示倒数第二个字符
![image.png](https://s2.loli.net/2023/10/30/lnpoPLbYVHFUj16.png)

下标从0开始，若end越界，则end为最后一个字符的下标
如果字符串为汉字，那么可能返回乱码
### setrange
```bash
setrange key offset value
```
offset：从第几个字节开始替换
value：替换的内容
返回值为新字符串的长度
![image.png](https://s2.loli.net/2023/10/30/SZ6XUrCFpanDAzs.png)
k1从"helloworld"的第二个字节开始，被"wwwwwwwwwwwwwwwwwww"替换

若key之前不存在，或者value为空串，那么setrange会将偏移量之前的字节用0填充
![image.png](https://s2.loli.net/2023/10/30/dE6oITh5jbZKuga.png)
### strlen
```bash
strlen key
```
返回值为字符串的长度（以字节为单位）
![image.png](https://s2.loli.net/2023/10/30/lHSaV3nB8vcKkZ6.png)
### string内部编码
- int：8字节的长整型
- embstr：大于等于39字节的字符串
- raw：大于39字节的字符串
redis会根据当前值的类型自动选择适合的内置类型进行存储

使用`object encoding key`查看key对应的内部编码
![image.png](https://s2.loli.net/2023/10/30/oHjUFPxWnvZqz7O.png)
redis用字符串存储小数
![image.png](https://s2.loli.net/2023/10/30/od8gNmfnFEzi9x3.png)
对于小数的计算，需要将字符串转换成小数再将小数转换成字符串进行存储，所以遇到使用小数的场景时，需要考虑清楚
### string的应用场景
作为缓存：应用服务访问数据时，先访问redis
若数据在redis中存在，则不需要访问底层数据库，直接访问redis
若数据在redis中不存在，则访问底层数据库，访问完成后底层数据库还需要将数据写入redis

为防止redis中的热点数据越来越多，底层数据库在将数据写入redis时，需要设置一个过期时间
同时redis在内存不足时，也有相应的淘汰策略

作为计数器：对于频繁访问数据库的操作，如统计视频的播放次数，使用redis再合适不过
需要注意的是：redis不擅长数据分析和处理，对于逻辑复杂的数据分析与处理，redis需要将数据**异步**写入到底层数据库（MySQL）中，将任务交给底层数据库

共享会话：redis存储session信息，应用服务器不再独立地存储属于自己的session，而是共享redis的session信息，这将优化客户端的体验，也是比较符合逻辑的

手机验证码：
如设置一分钟之内最多只能获取5次验证码
将用户的手机号作为key，`set key 1 ex 60 nx`，nx表示若key不存在才能成功set
接收set语句的返回值，若返回true，说明该用户之前未获取过验证码
若返回false，说明该用户之前获取过验证，此时将key对应的value+1（incr）。若key对应的value超过5，说明用户在一分钟之前已经获取了5次验证码，此时禁止获取验证码，否则执行生成验证码的逻辑

伪代码：
```cpp
string 发送验证码(string& phonenum)
{
	key = phonenum;
	bool t = redis::set key 1 ex 60 nx;
	if (!t)
	{
		long long cnt = redis::incr key;
		if (cnt > 5)
			return nullptr;
	}
	// 生成验证码并发送
}
```
生成验证码后，将用户的电话和验证码作为键值对存储到redis中，并设置过期时间
若用户请求登录，将用户的电话作为key，得到其value，判断是否存在且存在是否相等
## hash
### hset, hget, hexists和hdel
hset：设置hash中指定字段（field）的值（value）
```bash
hset key field value [field value ...]
```
返回设置成功的键值对（field和value）数量
![image.png](https://s2.loli.net/2023/10/31/BoZ9NAXQ4PFRIvL.png)

hget：获取hash中指定字段的值
```bash
hget key field
```

hexists：判定hash是否有指定字段
```bash
hexists key field
```
返回值：1表示存在，0表示不存在
三者的时间复杂度为$O(1)$

hdel：删除key中的field
注意和del进行区别，del删除的是key
```bash
hdel key field [field ...]
```
返回值：本次操作删除的字段个数
时间复杂度为$O(n)$，n为field的数量

### hkeys和hvals
hkeys：获取key中所有的field
```bash
hkeys key
```

![image.png](https://s2.loli.net/2023/10/31/PYvzHFsUdBaGAQZ.png)
![image.png](https://s2.loli.net/2023/10/31/bv1SfmUnINgP38A.png)

hvals：获取key中所有的field对应的value值
```bash
kvals key
```
两者的时间复杂度为$O(n)$，n为field的数量
两个操作存在一定的风险，和keys \*的负作用差不多
![image.png](https://s2.loli.net/2023/10/31/PYvzHFsUdBaGAQZ.png)
![image.png](https://s2.loli.net/2023/10/31/HDMzNVxy3jWw8b9.png)
### hgetall和hmget
hgetall：获取指定key的键值对（field和value）
```bash
hgetall key
```
![image.png](https://s2.loli.net/2023/10/31/PYvzHFsUdBaGAQZ.png)
![image.png](https://s2.loli.net/2023/10/31/r2U8tHowXa1g6mq.png)
时间复杂度为$O(n)$，n为field的数量，谨慎使用！

hmget：一次查询key中的多个field
```bash
hmget key field[field ...]
```
给出的多个value顺序和指定的field顺序是匹配的
![image.png](https://s2.loli.net/2023/10/31/PYvzHFsUdBaGAQZ.png)
![image.png](https://s2.loli.net/2023/10/31/nlYvPtdVNFRhfj9.png)

虽然有hmset，但是其作用和hset相同，hset就能一次性设置多个键值对，此时没有必要使用hmset
### hlen, hsetnx, hincrby和hincrbyfloat
hlen：获取key中field的数量
```bash
hlen key
```
时间复杂度为$O(1)$
![image.png](https://s2.loli.net/2023/10/31/6RXmZslxUC5K1Dh.png)
![image.png](https://s2.loli.net/2023/10/31/zCbftNuBoXDMqVG.png)

hsetnx：当前key中不存在对应的field，则设置成功，否则设置失败
```bash
hsetnx key field value
```
成功返回1，失败返回0
![image.png](https://s2.loli.net/2023/10/31/6RXmZslxUC5K1Dh.png)
![image.png](https://s2.loli.net/2023/10/31/j85pgf24OtDKG3k.png)

hincrby和hincrbyfloat：对当前key中某个field对应的value+/-整数/小数
```bash
hincrby key field n
hincrbyfloat key field n
```
上述命令的时间复杂度为$O(1)$

### hash的内部编码
- ziplist（压缩列表）：hash的本质是一个数组，有些位置有元素有些位置无元素，将无元素的位置进行压缩。ziplist节省了空间，但是读写速度较慢，所以需要满足：1. 哈希中的元素数量较少 2. 每个value值的长度较短。配置文件中修改hash-max-ziplist-entries（默认512个）和hash-max-ziplist-value（默认64字节）的值，改变hash的内部编码方式
- hashtable（哈希表）

### hash的使用场景
作为缓存：存储结构化的数据，类似关系数据库中的表结构。相比于使用string作为缓存，hash作为缓存时，可以根据field修改对应的value。而用string作为缓存时，修改某个value时，就需要将整个字符串反序列化，找到对应value所处的位置，修改完成后再将结构化数据进行序列化

使用hash作为缓存时，修改/读取数据很方便，也更高效，但是hash比较占用空间
使用string作为缓存时，修改/读取数据不方便，但是string比较节省空间

## list
redis中的list类似于数组/顺序表，支持头尾的插入删除（实现方式类似deque）
下标从0开始，且支持负数下标，-1表示倒数第一个元素
list是“有序”的，即将list所有元素顺序颠倒，得到的list和原来的list不等价（顺序很关键）
并且list中的元素**允许重复**，对比hash，hash中的元素则不允许重复
而list支持头尾的插入删除，所以可以将list作为栈 / 队列使用，虽然list支持下标的索引（最早的时候，redis通过list类型实现了消息队列，现在redis通过stream类型实现消息队列）
### lpush和lrange
lpush：头插，支持多次插入，多次插入时，按照顺序进行头插
```bash
lpush key element [element ...]
```
时间复杂度$O(1)$
返回值：插入完成后list的长度

lrange：查看指定范围内的元素
```bash
lrange key start stop
```
start和stop为闭区间的起点与重点，并且下标支持负数
![image.png](https://s2.loli.net/2023/10/31/MmPYJSByNXsO4rb.png)
### lpushx, rpush, rpushx
lpushx：如果key不存在，则直接返回，只有key存在时才将元素插入
```bash
lpushx key element [element ...]
```

rpush同lpush的使用相同：插入方向是list的尾端
redis中没有rrange，lrange中的l指的是list不是left
rpushx同lpushx的使用相同：插入方向是list的尾端
### lpop, rpop
lpop：删除list左侧第一个元素
```bash
lpop key 
```
返回值为被删除元素

rpop：
```bash
rpop key [count]
```
在redis6.2之后，新增了count参数，表示要删除元素的数量
### lindex, linsert
lindex：获取从左数第index个位置（从0开始）的元素
```bash
lindex key index
```
时间复杂度为$O(n)$，如果下标非法，返回nil

linsert：列表的任意位置插入元素
```bash
linsert key <before | after> pivot element 
```
将element插入左数第一个pivot的前/后
返回插入完成后list的长度，时间复杂度为$O(n)$
![image.png](https://s2.loli.net/2023/10/31/x2cGOBpdiYUP7MZ.png)

llen：获取list的长度
```bash
llen key
```

### lrem, ltrim, lset
rem = remove
```bash
lrem key count element
```
count：要删除的个数
element：要删除的值
- count > 0 ：从左往右删除count个
- count < 0 ：从右往左删除-count个
- count = 0 ：删除所有的element

ltrim：保留范围内的所有元素，删除其他元素
```bash
ltrim key start stop
```

lset：根据下标（从0开始），修改元素
```bash
lset key index element
```
### blpop, brpop
阻塞版本命令
和lpop，rpop类似，只是多了个阻塞的特性，b = block
若list中存在元素，那么blpop，brpop与lpop，rpop相同
若list中不存在元素，那么blpop，brpop会根据**timeout**阻塞一段时间，期间redis可以执行其他命令。直到list中被插入元素，bolpop立即返回

blpop，brpop可以设置多个key，redis将从左往右遍历这些key，哪个key中存在元素，立即删除并返回
如果多个客户端对同一个键执行pop，最先执行pop命令的客户端将执行pop操作
```bash
blpop key [key ...] timeout 
```
timeout以秒为单位
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202310311148757.png)
blpop返回两个值，第二个值表示被删除元素，第一个值表示被删除元素来自哪个list
### list内部编码
- quicklist：较新的redis实现中，采用quicklist代替ziplist和linkedlist，其中ziplist是压缩链表，使用连续的内存块存储数据，当数据大小超过阈值时，将使用linkedlist。linkedlist则是普通双向链表，插入和删除元素的时间复杂度为$O(1)$，而ziplist的插入和删除元素的时间复杂度为$O(n)$。quicklist集合了ziplist和linkedlist的优点

## set
set是无序的（顺序不重要），变换顺序后，仍然认为两集合是相同的
set中的元素是唯一的
和list类似，set中的每个元素都是string类型
### sadd, smembers, sismember
```bash
sadd key member [member ...]
```
返回成功添加的元素数量
![image.png](https://s2.loli.net/2023/10/31/1nIA5yxjs6rbJcD.png)

查看集合中的元素
```bash
smembers key
```
两者的时间复杂度为$O(1)$

判断集合中是否存在元素
```bash
sismember key member
```
存在返回1，不存在返回5
![image.png](https://s2.loli.net/2023/10/31/zkjRJio3qvBSYlA.png)
### spop, srandmember
spop：**随机**删除count个元素
```bash
spop key [count]
```
返回被删除元素的值
![image.png](https://s2.loli.net/2023/10/31/QtuHqopPCEljksW.png)

![image.png](https://s2.loli.net/2023/10/31/pD6IWomZ3VFzPuy.png)

srandmember：随机获取set中的一个元素

```bash
srandmember key 
```
![image.png](https://s2.loli.net/2023/10/31/IoKcmiN9btuAMYB.png)
### smove, srem

smove：将元素从source中移动到destination
```bash
smove source destination member
```
返回1表示成功，0表示失败
时间复杂度$O(1)$

若目标set中存在相同元素，那么smove会忽略插入操作，只执行源set中的删除操作
若源set中不存在元素，那么smove将失败

srem：从集合中删除元素
```bash
srem key member [member ...]
```
返回成功删除元素的个数
### sinter, sinterstore
对集合求交集
```bash
sinter key [key ...]
```
每个key对应一个集合
返回交集，时间复杂度$O(n * m )$，n为最小集合的元素数量，m为最大集合的元素数量
![image.png](https://s2.loli.net/2023/10/31/J2gyEGweq5uLj34.png)

sinterstore：将交集存储到目标集合中
```
sinterstore destination key [key ...]
```
返回交集的元素个数
若目标集合中存在元素，那么这些元素将被删除/覆盖
### sunion, sunionstore, sdiff, sdiffstore
sunion：返回并集的结果
```
sunion key [key ...]
```
时间复杂度$O(n)$，n为总的元素个数
![image.png](https://s2.loli.net/2023/10/31/2uOpCFWwoM5i1qd.png)

![image.png](https://s2.loli.net/2023/10/31/xKFWTOeS9DCco7n.png)

sunionstore：将并集存储到目标集合中
```
sunionstore destination key [key ...]
```
返回并集的元素个数

sdiff：求集合的差集
```
sdiff key [key ...]
```

![image.png](https://s2.loli.net/2023/10/31/2uOpCFWwoM5i1qd.png)
![image.png](https://s2.loli.net/2023/10/31/NmEL8rqQR2HeAbW.png)

sdiffstore：将查集存储到集合中
```
sdiffstore destination key [key ...]
```
时间复杂度都是$O(n)$
### set的内部实现
- inset：当元素为整数且数量不是很多时使用
- hashtable

![image.png](https://s2.loli.net/2023/10/31/l5gIe2miVPfpuOa.png)

### set应用场景
1. 用set保存用户的"标签"，用户画像。分析出个人的特征后，投其所好地投放消息。将搜集到的标签转换成简短的字符串，保存到set中。用set进行集合计算，可以衍生出"用户关系"
2. 计算用户之间的共同好友，可以做到好友推荐
3. 用set统计UV，user view，每个用户访问服务器都会产生一个UV，同一用户多次访问不会增加UV。PV，page view，用户每次访问服务器都会产生PV。UV需要按照用户进行去重

## zset
有序集合，这里的有序指的是顺序/降序
zset中的member是唯一的，但是对应的分数可以重复，分数只是为了辅助排序
如果分数相同，按照元素的字典序排列
zset内部实现为升序
### zadd, zrange
zadd：向有序集合中添加元素和分数
```
zadd key [NX | XX] [GT | LT] [CH] [INCR] score member [score member ...]
```
元素和分数作为一对pair，可以通过元素找到其分数，也能通过分数找到元素
XX：只更新元素，不会添加新的元素
NX：只添加元素，不会更新元素
默认：不存在就添加，存在就更新
LT：less than，只有当分数小于当前分数时，才会更新元素
GT：greater than，只有当分数大于当前分数时，才会更新元素，两者都不会添加新元素
CH：changed，zadd返回添加的元素数量，加上CH后，返回添加与更新的元素数量
INCR：对分数进行+/-运算，只能对一个分数进行运算。如`zadd key INCR 4 member`，对member的分数进行+4运算

时间复杂度为$O(logN)$，N为原集合的元素个数

zrange：返回指定范围内的元素
```
zrange key start stop [withscores]
```

![image.png](https://s2.loli.net/2023/10/31/UIBbZLEdT6SaYow.png)

### zcard, zcount
获取集合中的元素个数
```
zcard key
```

获取指定区间的元素个数
```
zcount key min max
```
默认是闭区间，要表示开区间：`(min (max`
![image.png](https://s2.loli.net/2023/10/31/ZmFWUuRXvHE6Q9r.png)

可以使用inf和-inf作为min和max
时间复杂度为$O(logN)$
### zrevrange, zrangebyscore
zrevrange按照降序返回指定范围内的元素
```
zrevrange key start stop [withscores]
```
![image.png](https://s2.loli.net/2023/10/31/Tw5X3voGIhpV91A.png)

按照分数查找元素，返回member，使用方法和zcount类似
该命令可能被废弃，将合并到zrange中
```
zrangebyscore key min max [withscores]
```
### zpopmax, bzpopmax
zpopmax：删除并返回分数最高的count个元素（包括member和score）
```
zpopmax key [count]
```
如果存在多个元素的score相同，那么将删除member字典序最高的元素
时间复杂度为$O(logN * M)$，N为zset中元素数量，M为count
redis底层实现中，使用了一个通用的删除函数来完成zpopmax，该删除函数的时间复杂度为$O(logN)$，当然可以优化成$O(1)$，只需要特殊记录分数最高元素的位置即可。但是$logN$其实也没有很慢，所以redis也就没有优化了

bzpopmax，阻塞版本的zpopmax
```
bzpopmax key [key ...] timeout
```
每个key都是一个zset，所以zset为空时，陷入阻塞。一旦有zset不为空，立即删除并返回
timeout的单位为秒，可以是小数的形式
时间复杂度$O(logN)$
![image.png](https://s2.loli.net/2023/10/31/dVS7g1RyBtJACN2.png)

### zpopmin, bzpopmin
zpopmin：删除并返回分数最低的count个元素
```
zpopmin key [count]
```
bzpopmin：，zpopmin的阻塞版本，和bzpopmax使用方法相同
### zrank, zrevrank, zscore
zrand：返回member在zset中的排名
```
zrank key member
```
![image.png](https://s2.loli.net/2023/10/31/dCw9lX45DgmMOPF.png)
时间复杂度$O(logN)$

zrevrank：返回member在zset中的排名（倒排）
```
zrevrank key member
```

zscore：查询指定member的分数
```
zscore key member 
```
时间复杂度$O(1)$，redis对于这个操作做了特殊优化（付出了额外的空间代价）
### zrem, zremrangebyrank, zremrangebyscore
zrem：根据member删除元素
```
zrem key member [member ...]
```
时间复杂度$O(logN * M)$

zremrangebyrank：根据排名的范围删除元素
```
zremrangebyrank key start stop
```
区间为闭区间
时间复杂度$O(logN + M)$

zremrangebyscore：根据分数删除元素
```
zremrangebyscore key min max
```
区间为闭区间
时间复杂度$O(logN + M)$
### zincrby
zincrby：将指定member的score+/-值
```
zincrby key n member
```
### zinterstore
求交集并保存到集合中
```
zinterstore destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [ARRGEGATE <SUM | MIN | MAX>]
```
numkeys：后续有几个key参与运算
WEIGHTS：key在运算中的权重，相当于一个系数，每个集合的分数都要乘以这个系数，可以为小数，默认为1
AGGREGATE：进行运算时，根据member判断元素是否相同。如果member相同，score不同，最终分数如何计算？根据AGGREGATE的值计算，默认为相加
### zunionstore
求zset的并集并存储到目标集合中
```
zunionstore destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [ARRGEGATE <SUM | MIN | MAX>]
```
可选参数和zinterstore相同，这里不再赘述
### zset编码方式
- ziplist：若元素较少或者元素的体积较小，使用ziplist存储
- skiplist：否则采用skiplist存储
### zset应用场景
排行榜系统：微博热搜、游戏天梯排行、成绩排行
关键：排行中的“分数”是实时变化的，需要我们高效地更新排行

