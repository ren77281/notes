```toc
```
## 渐进式遍历
keys能够通过通配符匹配规则，将redis中所有符合条件的key筛选出来
```
keys pattern
```
但是这个命令很危险，若redis中的数据很多，导致keys的执行时间过长。由于redis是单线程，所以keys大概率将引起**阻塞**

通过渐进式遍历，就可以做到在保证服务器不阻塞的情况下，获取所有的key
渐进式遍历不是一次新地将所有key拿到，而是每执行一次命令，只获取其中一部分的key。这样就保证了每次操作不会阻塞服务器，要想获取所有的key，需要多次执行渐进式遍历

渐进式遍历其实是一组命令，这些命令的使用方法是一样的，其中的代表命令为scan

## scan
```
scan cursor [pattern] [COUNT count] [TYPE type]
```
cursor：光标，执行当前遍历的位置，为0表示从头开始遍历。不能将其理解为下标，它只是一个字符串
count：限制这次的遍历要获取的元素数量，默认是10。和MySQL的limit不一样，count只是一个建议，返回结果的数量并不一定和count相同，只是和count差不多
type：根据key对应的value类型进行进一步的筛选
返回值分两部分：一部分为提示，下一次遍历时光标要从哪开始
一部分为遍历到的具体内容

当前已有的key
![image.png](https://s2.loli.net/2023/11/01/9tlSDob7OcE5zQa.png)

输入`scan 0`
![image.png](https://s2.loli.net/2023/11/01/E7K8WtelDUyPbBn.png)
`7`为下次遍历时，需要指定的光标位置，剩下的数据为遍历到的元素

当scan返回空集合或者光标为0，说明遍历结束
![image.png](https://s2.loli.net/2023/11/01/gGMcNH1OJDSPB6F.png)
指定count参数为5，返回了6个结果
![image.png](https://s2.loli.net/2023/11/01/RVf13v7CJ6dUhAT.png)

scan可以随时终止，不会对服务器产生任何副作用
若执行scan的过程中，key发生了变化（增加，删除），那么scan可能会出现遗漏或重复
## 数据库管理
MySQL中，可以创建多个数据库database。而在redis中，只能使用现成的（已经存在的）数据库，无法删除/创建数据库。redis提供了16个数据库（0~15），并且默认使用的是0号数据库
通过
```
select index
```
index为0~15，选择数据库
实际使用中很少切换数据库，一般都是直接使用0号数据库

获取当前数据库下的key数量：
```
dbsize
```

删除当前数据库中的所有key
```
flushdb
```
删除所有数据库中的所有key
```
flushall
```

## 使用C++调用redis
redis自定义了一个通信协议：**RESP**（redis serialization protocol）
要实现一个自定义客户端，就必须清楚服务端使用的协议。由于redis将协议的内容公开，所以我们能实现自己的redis客户端

RESP协议
优点：
1. 简单好实现
2. 快速进行解析
3. 可读性高
传输层基于TCP，但是与TCP没有强耦合关系
请求和相应之间的通信模型是一问一答的
客户端以bulk string数组的形式将命令发送给服务器，不同的命令返回结果不一样

服务端返回的数据：
- 对于简单字符串（Simple String），第一个字节为"+"
- 对于错误（Errors），第一个字节为"-"
- 对于整数（Integers），第一个字节为":"
- 对于Bulk String，第一个字节为"$"
- 对于数组（Arrays），第一个字节为"\*"

Bulk String和Simple String的区别，Bulk String可以用于传输二进制数据，因为其实现了二进制安全，而Simple String只能用于传输简单字符串

这些协议的序列化与反序列化已经有第三方库实现，我们不需要手动实现

### 安装redis-plus-plus
安装hiredis：
```bash
apt install -y libhiredis-dev
```
通过源码编译安装redis-plus-plus
下载源码：
```bash
git clone https://github.com/sewenew/redis-plus-plus.git
```
进入目录：
```bash
cd redis-plus-plus/
```
创建目录并进入：
```bash
mkdir build
cd build
```
安装cmake：
```bash
apt install -y cmake
```
执行：
```bash
cmake ..
```
最后安装：
```bash
make         # 编译生成动静态库
make install # 将动静态库拷贝到系统目录中
```
安装完成后，在`/usr/local/include`目录下会出现sw目录，存储了redis的头文件
`/usr/local/lib`目录下也存储了redis的库文件
```bash
ls /usr/local/include/sw
ls /usr/local/lib | grep redis
```
![image.png](https://s2.loli.net/2023/11/01/TXBI1mMUsJPDzjW.png)
使用find命令查找相关文件也可以
```bash
find / -name "redis*"
find / -name "libredis*"
```

编写demo连接redis服务器，使用ping命令检测连通性
```cpp
#include <iostream>
#include <string>
#include <sw/redis++/redis++.h> // 包含redis头文件
using std::cout;
using std::cin;
using std::endl;

int main()
{
	// 创建Redis对象时，需要指定redis服务的IP和端口
	// 字符串为一个url，RESP协议是基于TCP应用层协议
	sw::redis::Redis redis("tcp://127.0.0.1:6379"); 
	std::string res = redis.ping();
	cout << res << "\n";
	return 0;
}
```
要想与redis进行交互，就要初始化一个sw::redis::Redis对象，在redis命令行中输入的命令都能通过sw::redis::Redis进行发送。其中sw::redis是一个命名空间，其中定义了很多redis相关的类和方法
需要将一个url作为构造函数的参数传入，比如："tcp://127.0.0.1:6379"，这个url中，tcp表示用tcp协议进行通信（redis是基于tcp的），127.0.0.1表示redis服务所在的IP地址，6379表示redis服务所在的端口号
redis.ping()等价于在redis命令行中输入ping，由于这个命令返回一个字符串，用string将其接收并打印

编写makefile文件，需要将使用到的动态库放到编译指令中（hiredis, redis++, pthread）
```
demo:demo.cc
	g++ -o $@ $^ -std=c++17 -lpthread -lhiredis -lredis++

.PHONY:clean
clean:
	rm -rf demo
```
配置系统文件，使链接器能找到hiredis的动态库，其他动态库已经位于系统路径下
```
cd /etc/ld.so.conf.d
```
创建配置文件，将`find / -name libhiredis*`返回的目录写入配置文件中
![image.png](https://s2.loli.net/2023/11/01/Bb6dIVcAZanChkm.png)
写入动态库所在目录即可
```bash
vim redis.conf
```
![image.png](https://s2.loli.net/2023/11/01/Ixc6L4O1jaUdpKe.png)
执行
```
sudo ldconcig
```
此时配置完成
执行demo返回`PONG`说明redis-plus-plus安装完成
![image.png](https://s2.loli.net/2023/11/01/rvU75CuY9yNWOz1.png)

## get, set, exists , del, keys
set：
```cpp
void set(const sw::redis::StringView &key, const sw::redis::StringView &val);
```
其中StringView是sw::redis中定义的一个只读字符串（相比于普通字符串，StringView做了更多的优化），以字符串的形式传入key和value的值即可，如`redis.set("k1", "111");`

get：
```cpp
sw::redis::OptionalString get(const sw::redis::StringView &key)
```
将key以字符串的形式传入，get将返回key对应的value值。返回值为sw::redis::OptionalString，该类重载了bool，若key不存在，那么OptionalString的bool值为false。若key存在，那么OptionalString的bool值为true
由于该类没有重载`<<`运算符，所以不能直接输出，但该类的value()方法可以以字符串的方式返回value值，value()的返回值可以直接输出
```cpp
// get set的使用
void test1(sw::redis::Redis& redis)
{
	redis.flushall(); // 清空当前数据库，慎用

	redis.set("k1", "111");
	redis.set("k2", "222");
	redis.set("k3", "333");

	auto v1 = redis.get("k1");
	if (v1) cout << "k1-" << v1.value() << "\n";

	auto v2 = redis.get("k2");
	if (v2) cout << "k2-" << v2.value() << "\n";

	auto v3 = redis.get("k3");
	if (v3) cout << "k3-" << v3.value() << "\n";

	auto v4 = redis.get("k4");
	if (v4) cout << "k4-" << v4.value() << "\n";
}
```

![image.png](https://s2.loli.net/2023/11/01/yQ1zK8V6T4bsYtx.png)

exists：
```cpp
long long exists(const sw::redis::StringView &key);
```
以字符串的方式传入key，exists将返回存在的key的数量
用初始化列表存储多个字符串，并将初始化列表作为参数传入，就能询问这些key是否在数据库中存在

```cpp
void test2(sw::redis::Redis& redis)
{
	redis.flushall();
	redis.set("k1", "111");
	redis.set("k2", "222");
	redis.set("k3", "333");

	auto t = redis.exists("k1");
	cout << "k1:" << t << "\n";

	t = redis.exists({"k1", "k2", "k3"});
	cout << "k1, k2, k3:" << t << "\n";

	t = redis.exists("t4");
	cout << "k4:" << t << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/01/Jx5DcAanCOWe9r2.png)

del和exists的使用相同，参数一样，也能使用初始化列表作为参数删除多个key，返回值为成功删除的元素数量
```cpp
void test3(sw::redis::Redis& redis)
{
	redis.flushall();
	redis.set("k1", "111");
	redis.set("k2", "222");
	redis.set("k3", "333");

	auto t = redis.exists({"k1", "k2", "k3"});
	cout << "k1, k2, k3:" << t << "\n";
	redis.del("k2");
	t = redis.del({"k1", "k2", "k3"});
	cout << "del->k1, k2, k3:" << t << "\n";

	t = redis.exists({"k1", "k2", "k3"});
	cout << "k1, k2, k3:" << t << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/01/azdpfTInYky9qV6.png)

keys：
```cpp
void Redis::keys(const StringView &pattern, Output output)
```
key的第一个参数为字符串形式的规则表达式，用来匹配key
keys的第二个参数为一个插入迭代器，我们需要先创建一个存储结果的容器，再创建一个指向该容器的插入迭代器，将其作为keys的第二个参数，keys的返回结果将保存到之前创建的容器中
STL中有五种迭代器：
1. 输入迭代器（只读）
2. 输出迭代器（可写）
3. 前向迭代器
4. 双向迭代器
5. 随机访问迭代器

插入迭代器属于一种输出迭代器（用来将数据输出），有三种插入迭代器
1. front_insert_iterator（将数据输出到容器的开头）
2. back_insert_iterator（将数据输出到容器的末尾）
3. insert_iterator（将数据输出到容器的任意位置，指向位置之前）

每种迭代器都有相应的方法，调用这些方法以构造相应的迭代器
如back_inserter()，将容器作为参数传入，该方法返回相应的迭代器
使用：
```cpp
vector<string> res;
auto it = std::back_inserter(res);
```

```cpp
template <class T>
void printContainer(const T& container)
{
    for (const auto& elem : container)
    {
        cout << elem << "\n";
    }
}

void test4(sw::redis::Redis& redis)
{
	redis.flushall();
	redis.set("k1", "111");
	redis.set("k2", "222");
	redis.set("k3", "333");
	redis.set("k4", "444");
	redis.set("k5", "555");
	redis.set("k6", "666");

	vector<string> res;
	auto it = std::back_inserter(res);
	redis.keys("*", it);
	printContainer(res);
}
```
![image.png](https://s2.loli.net/2023/11/01/UqOo1PdeB8HW2xA.png)
## expire, type

expire
```cpp
bool Redis::expire(const StringView &key, const std::chrono::seconds &timeout)
```
以字符串的形式将key作为第一个参数，将时间作为第二个参数
```cpp
#include <chrono>
using namespace std::chrono_literals;
```
引入头文件和声明命名空间：C++11之后，支持了更多的字面值，如3s，7min，以及h, ms, us, ns。这些字面值可以提高程序的可读性
expire的第二个参数就可以使用7s这样的字面值，前提是引入头文件和声明命名空间

ttl：
```cpp
long long sw::redis::Redis::ttl(const sw::redis::StringView &key)
```
ttl返回long long表示key的剩余存活时间

```cpp
void test5(sw::redis::Redis& redis)
{
	using namespace std::chrono_literals;
	redis.flushall();
	redis.set("k1", "111");

	redis.expire("k1", 7s);
	std::this_thread::sleep_for(3s); // 线程休眠3s

	auto time = redis.ttl("k1");
	cout << "k1:" << time << "s" << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/01/g2r6cnMxmhp7Yeo.png)


type：
```cpp
std::string sw::redis::Redis::type(const sw::redis::StringView &key)
```
返回key所属的类型，返回值是类型是std::string

```cpp
void test6(sw::redis::Redis& redis)
{
	redis.flushall();
	redis.set("k1", "111");
	redis.lpush("k2", "222");

	auto res = redis.type("k1");
	cout << "k1:" << res << "\n";

	res = redis.type("k2");
	cout << "k2:" << res << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/01/XiEe93MxHvsIBhc.png)

## string的set, mset, mget
超时设置与NX|XX
```cpp
redis.set("k1", "111", 5s, sw::redis::UpdateType::EXIST)
```
第三个参数为超时时间，引入了chrono头文件并使用命名空间std::chrono_literals后，即可使用这样的字面值
第四个参数有两个可选项：sw::redis::UpdateType::NOT_EXIST和sw::redis::UpdateType::EXIST。分别对应着NX和XX模式，即只创建与只更新两种模式

若不想使用第三个参数，但是要使用第四个参数，将第三个参数设置为0s即可

mset：
```cpp
// 第一种方法
redis.mset({std::make_pair("k1", "111"), std::make_pair("k2", "222")});

// 第二种方法
vector<std::pair<string, string>> keys = {
	{"k1", "111"},
	{"k2", "222"},
	{"k3", "333"}
};
redis.mset(keys.begin(), keys.end());
```
第一种方法：直接用初始化列表存储多个pair，将初始化列表作为参数传入
第二种方法：将多个键值对保存到容器中，将容器的头尾迭代器传入

mget的使用和keys类似，需要先创建一个保存结果的容器，再创建一个指向该容器的插入迭代器并将该容器传入
至于说mget的第一个参数，用初始化列表存储要查询的key即可，具体见demo

```cpp
void test1(sw::redis::Redis& redis)
{
    redis.flushall();
    // mset的第一种使用方法
	redis.mset({std::make_pair("k1", "111"), std::make_pair("k2", "222")});
	// mset的第二种使用方法
    vector<std::pair<string, string>> keys = {
		{"k1", "111"},
		{"k2", "222"},
		{"k3", "333"}
	};
	redis.mset(keys.begin(), keys.end());
	
	vector<sw::redis::OptionalString> res;
	auto it = std::back_inserter(res); 
	redis.mget({"k1", "k2", "k3"}, it);
	
	printContainerOptional(res);
}
```
![image.png](https://s2.loli.net/2023/11/01/fNz3apJ5WlgEPXK.png)

## getrange, setrange
getrange 
```cpp
std::string sw::redis::Redis::getrange(const sw::redis::StringView &key, long long start, long long end)
```
传入key与一个闭区间的左右端点，getrange将返回key对应value的子字符串。若区间方法，将返回空字符串
```cpp
void test2(sw::redis::Redis& redis)
{
    redis.flushall();
	redis.set("k1", "helloworld");
	string res = redis.getrange("k1", 3, 7);
	cout << "helloworld(3, 7):" << res << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/02/8mt7apuS64zvGCZ.png)

setrange
```cpp
long long setrange(const sw::redis::StringView &key, long long offset, const sw::redis::StringView &val)
```
将key对应的value，从offset个字符开始往后替换为val字符串
```cpp
void test2(sw::redis::Redis& redis)
{
    redis.flushall();
	redis.set("k1", "helloworld");
	string res = redis.getrange("k1", 3, 7);
	cout << "helloworld(3, 7):" << res << "\n";

	redis.setrange("k1", 3, "xxxxx");
	auto val = redis.get("k1");
	cout << val.value() << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/02/Ndw6liKkyj31rts.png)
## incr, decr
```cpp
long long sw::redis::Redis::incr(const sw::redis::StringView &key)
```
将传入的key对应的value值自增1，并且返回自增后的结果，类型为long long
如果用get获取其value，那么其类型为OptionalString，需要调用其value方法并将字符串转换成整数才能使用
所以要使用incr的结果，推荐接收incr的返回值
```cpp
void test3(sw::redis::Redis& redis)
{
    redis.flushall();
	redis.set("k1", "2");
	auto res = redis.incr("k1");
	cout << res << "\n";

	res = redis.decr("k1");
	cout << res << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/02/3IY9HTSDqzJiPR2.png)

## lpush, lrange, 
lpush：指定要插入的key，可以插入一个/多个key
插入多个key时，可以通过初始化列表也可以通过迭代器的方式进行
```cpp
long long lpush(const sw::redis::StringView &key, const sw::redis::StringView &val)
long long lpush<T>(const sw::redis::StringView &key, std::initializer_list<T> il)
long long lpush<Input>(const sw::redis::StringView &key, Input first, Input last)
```

lrange：
```cpp
void lrange<Output>(const sw::redis::StringView &key, long long start, long long stop, Output output)
```
lrange获取list中指定范围内的元素，参数和命令行相同，只是需要添加一个插入迭代器以保存多个元素

```cpp
void test1(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.lpush("k1", "111");
    redis.lpush("k1", "222");
    vector<string> data = {"333", "444", "555"};
    redis.lpush("k1", data.begin(), data.end());

    vector<string> res;
    auto it = back_inserter(res);
    redis.lrange("k1", 0, -1, it);
    printContainer(res);
}
```

![image.png](https://s2.loli.net/2023/11/02/SCK5iyTc9lrzJ1t.png)
## lpop, rpop
lpop：
将key对应list的头部元素删除并返回，返回类型为OptionalString，需要判断其bool值是否为真，再调用其value方法，输出被删除的值
```cpp
sw::redis::OptionalString lpop(const sw::redis::StringView &key)
```

```cpp
void test1(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.lpush("k1", "111");
    redis.lpush("k1", "222");
    vector<string> data = {"333", "444", "555"};
    redis.lpush("k1", data.begin(), data.end());
    vector<string> res;
    auto it = back_inserter(res);
    redis.lrange("k1", 0, -1, it);
    printContainer(res);

    auto t = redis.lpop("k1");
    if (t) cout << "pop:" << t.value() << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/02/q2t8pEUrVRaNFJc.png)

## blpop, llen
阻塞版本的头删除，可以用第二个参数设置阻塞时间，默认为0秒
第一个参数可以是一个key，也可以是多个key（初始化列表或者迭代器），但是只会删除第一个有数据key
```cpp
sw::redis::OptionalStringPair blpop(const sw::redis::StringView &key, const std::chrono::seconds &timeout = std::chrono::seconds{0})
```
返回类型为OptionalStringPair，可以理解为`pair<string, string>`，第一个元素为被删除元素所属key，第二个元素为被删除元素
假设有一个OptionalStringPair对象v，获取第一个参数`v.value().first`，或者`v->first`，该类重载了`->`运算符，可以通过`->`直接获取参数值

```cpp
void test1(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.lpush("k1", "111");
    redis.lpush("k1", "222");
    vector<string> data = {"333", "444", "555"};
    redis.lpush("k1", data.begin(), data.end());
    vector<string> res;
    auto it = back_inserter(res);
    redis.lrange("k1", 0, -1, it);
    printContainer(res);

    auto t = redis.lpop("k1");
    if (t) cout << "pop:" << t.value() << "\n";

    auto v = redis.blpop("k1", 10s);
    if (v) cout << "key:" << v->first << " val:" << v.value().second << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/02/1oCn4UhVwIxmzkl.png)

llen：返回指定key的长度，类型为long long
```cpp
long long llen(const sw::redis::StringView &key)
```

```cpp
void test1(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.lpush("k1", "111");
    redis.lpush("k1", "222");
    vector<string> data = {"333", "444", "555"};
    redis.lpush("k1", data.begin(), data.end());
    vector<string> res;
    auto it = back_inserter(res);
    redis.lrange("k1", 0, -1, it);
    printContainer(res);

    auto t = redis.lpop("k1");
    if (t) cout << "pop:" << t.value() << "\n";

    auto v = redis.blpop("k1", 10s);
    if (v) cout << "key:" << v->first << " val:" << v.value().second << "\n";
    
    cout << "len:" << redis.llen("k1") << "\n";
}
```

![image.png](https://s2.loli.net/2023/11/02/4pYH9JkX6GgUIrS.png)
## sadd, smembers
sadd同样支持一次插入一个/多个元素，方法与之前的函数类型，这里不再赘述
smembers：
```cpp
void smembers<Output>(const sw::redis::StringView &key, Output output)
```
指定要获取的key，该函数将value值保存到插入迭代器指向的容器中，和之前的获取函数的使用类似
```cpp
void test2(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.sadd("k1", {"111", "222", "333"});

    std::set<string> res;
    auto it = inserter(res, res.begin());
    redis.smembers("k1", it);
    printContainer(res);
}
```
![image.png](https://s2.loli.net/2023/11/02/BebKWn62GRaphkr.png)

## sismember, scard
sismember：
```cpp
bool sismember(const sw::redis::StringView &key, const sw::redis::StringView &member)
```
函数返回一个bool，表示member是否为key的元素
```cpp
void test2(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.sadd("k1", {"111", "222", "333"});

    std::set<string> res;
    auto it = inserter(res, res.begin());
    redis.smembers("k1", it);
    printContainer(res);

    cout << "111 is k1 member? :" << redis.sismember("k1", "111") << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/02/zlQFHNivL2r3Vhd.png)

scard：获取set中的元素个数
```cpp
long long scard(const sw::redis::StringView &key)
```

```cpp
void test2(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.sadd("k1", {"111", "222", "333"});

    std::set<string> res;
    auto it = inserter(res, res.begin());
    redis.smembers("k1", it);
    printContainer(res);

    cout << "111 is k1 member? :" << redis.sismember("k1", "111") << "\n";
    cout << "scard:" << redis.scard("k1") << "\n";
}

```
![image.png](https://s2.loli.net/2023/11/02/k2P3fWhVZsUcn1d.png)

spop：
```cpp
sw::redis::OptionalString spop(const sw::redis::StringView &key)
```

```cpp
void test2(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.sadd("k1", {"111", "222", "333"});

    std::set<string> res;
    auto it = inserter(res, res.begin());
    redis.smembers("k1", it);
    printContainer(res);

    cout << "111 is k1 member? :" << redis.sismember("k1", "111") << "\n";
    cout << "scard:" << redis.scard("k1") << "\n";
    auto t = redis.spop("k1");
    if (t) cout << "spop:" << t.value() << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/02/TAyMRB9dV1vY6sk.png)
## sinter, sinterstore
sinter：
```cpp
void sinter<T, Output>(std::initializer_list<T> il, Output output)
```
初始化列表表示用求并集的key，output为保存并集的插入迭代器
```cpp
void test3(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.sadd("k1", {"111", "222", "333"});
    redis.sadd("k2", {"222", "333", "444"});

    std::set<string> res;
    auto it = inserter(res, res.begin());
    redis.sinter({"k1", "k2"}, it);
    printContainer(res);
}
```
![image.png](https://s2.loli.net/2023/11/02/l8qckEr6SBs5DnY.png)

sinterstore：
```cpp
long long sinterstore<T>(const sw::redis::StringView &destination, std::initializer_list<T> il)
```
第一个参数为存储并集的key，第二个参数为求哪些集合并集
```cpp
void test3(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.sadd("k1", {"111", "222", "333"});
    redis.sadd("k2", {"222", "333", "444"});

    std::set<string> res;
    auto it = inserter(res, res.begin());
    redis.sinter({"k1", "k2"}, it);
    printContainer(res);

    redis.sinterstore("k3", {"k1", "k2"});
    std::set<string> t;
    it = inserter(t, t.begin());
    redis.smembers("k3", it);
    printContainer(t);
}
```
![image.png](https://s2.loli.net/2023/11/02/D7BzrHhZVlwCW3A.png)

## hset, hget, hexists, hdel, hmget
hset：
```cpp
long long hset(const sw::redis::StringView &key, const sw::redis::StringView &field, const sw::redis::StringView &val)
long long hset(const sw::redis::StringView &key, const std::pair<sw::redis::StringView, sw::redis::StringView> &item)
long long hset<T>(const sw::redis::StringView &key, std::initializer_list<T> il)
long long hset<Input>(const sw::redis::StringView &key, Input first, Input last)
```
支持一次插入一个/多个元素，由于hash存储的是field和value，所以初始化列表中的元素需要为pair，容器中的元素也需要为pair

hget：获取key的field对应的value
```cpp
sw::redis::OptionalString hget(const sw::redis::StringView &key, const sw::redis::StringView &field)
```

```cpp
void test4(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.hset("k1", {std::make_pair("kk1", "111"), std::make_pair("kk2", "222")});
    auto res = redis.hget("k1", "kk1");
    if (res) cout << res.value() << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/02/CoTRWjJKOudbIBH.png)

hexists：判断key的field是否存在
```cpp
bool hexists(const sw::redis::StringView &key, const sw::redis::StringView &field)
```

```cpp
void test4(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.hset("k1", {std::make_pair("kk1", "111"), std::make_pair("kk2", "222")});
    auto res = redis.hget("k1", "kk1");
    if (res) cout << res.value() << "\n";
    cout << "k1的kk2是否存在? :" << redis.hexists("k1", "kk2") << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/02/wyRHjT6pkNdGt1E.png)

hdel：
```cpp
long long hdel(const sw::redis::StringView &key, const sw::redis::StringView &field)
long long hdel<T>(const sw::redis::StringView &key, std::initializer_list<T> il)
long long hdel<Input>(const sw::redis::StringView &key, Input first, Input last)
```
支持一次删除一个/多个key的field，返回值为成功删除的field数量


hkeys：获取key下的fields
```cpp
void hkeys<Output>(const sw::redis::StringView &key, Output output)
```
hvals：获取key下的values
```cpp
void hvals<Output>(const sw::redis::StringView &key, Output output)
```

```cpp
void test5(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.hset("k1", {
        std::make_pair("kk1", "111"), 
        std::make_pair("kk2", "222"),
        std::make_pair("kk3", "333"),
        std::make_pair("kk4", "444"),
        });

    vector<string> keyres;
    auto it = back_inserter(keyres);
    redis.hkeys("k1", it);
    cout << "keys:\n";
    printContainer(keyres);

    vector<string> valres;
    it = back_inserter(valres);
    redis.hvals("k1", it);
    cout << "vals:\n";
    printContainer(valres);
}
```
![image.png](https://s2.loli.net/2023/11/02/AFdnkBwxyqHLT9i.png)

hget只支持获取单个key下field对应的value
而hmget支持获取一个/多个key下field对应的value
```cpp
void hmget<T, Output>(const sw::redis::StringView &key, std::initializer_list<T> il, Output output)
void hmget<Input, Output>(const sw::redis::StringView &key, Input first, Input last, Output output)
```

```cpp
void test6(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.hset("k1", {
        std::make_pair("kk1", "111"), 
        std::make_pair("kk2", "222"),
        std::make_pair("kk3", "333"),
        std::make_pair("kk4", "444"),
        });

    vector<string> res;
    auto it = back_inserter(res);
    redis.hmget("k1", {"kk1", "kk2"}, it);
    printContainer(res);
}
```
![image.png](https://s2.loli.net/2023/11/02/OTfMqkKlRUg8X9D.png)

## zadd, zrange
zadd
```cpp
long long zadd(const sw::redis::StringView &key, const sw::redis::StringView &member, double score)
long long zadd<T>(const sw::redis::StringView &key, std::initializer_list<T> il)
long long zadd<Input>(const sw::redis::StringView &key, Input first, Input last)
```
zadd支持一次向有序集合中添加一个/多个元素，添加多个元素时，元素的类型为`pair<string, double>`

zrange：
根据指定范围查询，可以只查询member，也可以查询member和score
```cpp
void zrange<Output>(const sw::redis::StringView &key, long long start, long long stop, Output output)
```
若插入迭代器指向的容器类型为`<string, double>`，那么此时查询member和score
若插入迭代器指向的容器类型为`string`，那么此时查询member

```cpp
void test7(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.zadd("k1", {
        std::make_pair("aa", "99"),
        std::make_pair("bb", "98"),
        std::make_pair("cc", "97")
    });

    vector<std::pair<string, double>> resms;
    auto itms = back_inserter(resms);

    vector<string> resm;
    auto itm = back_inserter(resm);
    
    redis.zrange("k1", 0, -1, itms);
    redis.zrange("k1", 0, -1, itm);

    cout << "只查询member:\n";
    for (auto t : resm) cout << t << "\n";
    cout << "查询member和score:\n";
    for (auto t : resms) cout << t.first << ' ' << t.second << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/02/NqIW25QlcZFBSba.png)

## zrem, zsocre, zrank
zrem
```cpp
long long zrem(const sw::redis::StringView &key, const sw::redis::StringView &member)
long long zrem<T>(const sw::redis::StringView &key, std::initializer_list<T> il)
long long zrem<Input>(const sw::redis::StringView &key, Input first, Input last)
```
支持一次删除key下的一个/多个member，返回成功删除的数量

zscore：返回key下member对应的score
```cpp
sw::redis::OptionalDouble zscore(const sw::redis::StringView &key, const sw::redis::StringView &member)
```

```cpp
sw::redis::OptionalLongLong zrank(const sw::redis::StringView &key, const sw::redis::StringView &member)
```

```cpp
void test8(sw::redis::Redis& redis)
{
    redis.flushall();
    redis.zadd("k1", {
        std::make_pair("aa", "99"),
        std::make_pair("bb", "98"),
        std::make_pair("cc", "97")
    });

    auto res = redis.zscore("k1", "bb");
    if (res) cout << "zscore(k1.bb):" << res.value() << "\n";
    auto t = redis.zrank("k1", "bb");
    if (t) cout << "zrank(k1.bb)" << t.value() << "\n";
}
```
![image.png](https://s2.loli.net/2023/11/02/CU9GciIKPlLms6M.png)

