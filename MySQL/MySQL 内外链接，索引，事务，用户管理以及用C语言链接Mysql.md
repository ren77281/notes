```toc
```
## 内外链接
两张表直接做笛卡尔积为内连接，之前使用的都是内连接
两张表：stu和exam
![image.png](https://s2.loli.net/2023/10/09/qHAxXojv6MUhsGO.png)

将两张表进行连接：
![image.png](https://s2.loli.net/2023/10/09/PQD3IGi1bsgtM92.png)
内连接：
```
select * from 表1 inner join 表2 on 约束条件
```
表与表之间用inner join连接，之后用on表示内连接的约束条件
外连接：左外连接
```
select * from  表1 left join 表2 on 约束条件
```
以左表为主，根据约束显示右表的信息，若左表没有对应右表的信息，那么右表显示null
![image.png](https://s2.loli.net/2023/10/09/rH4nRSAWoNzMBh2.png)
右外连接
```
select * from  表1 right join 表2 on 约束条件
```
以右表为主，根据约束显示左表的信息，若右表没有对应左表的信息，那么左表显示null
***
## 索引
根据冯诺依曼体系，mysql中的所有操作，本质上都是对内存的操作，在一段时间后，mysql会将数据刷新到磁盘中（进行持久化操作）
mysql的所有操作都会被转换成对文件系统的操作，最终转换成对硬件的操作
所以数据库的底层依赖于文件系统，要了解数据库必须要先了解文件系统
读取文件就意味着读取磁盘，相比于读取内存，读取磁盘非常慢，所以在进行文件查找时，尽可能地减少随机查找次数（减少IO次数）是关键

总之，索引以降低增删改的速度为代价，提高了查询速度。索引用于支持海量数据的查询

磁盘的基本单位为512字节，mysql的innodb引擎使用16KB进行IO交互，这个基本单位由mysql自己维护，通过以下命令查看mysql交互的基本单位
```
show global status like 'innodb_page_size';
```

![image.png](https://s2.loli.net/2023/10/10/82lAVmXZWa31Hxb.png)

**主键的特性**：若建表时指定主键，那么所有的记录都会按照主键的大小进行排序
![image.png](https://s2.loli.net/2023/10/10/GcnLhT1wBorpugt.png)

为什么要排序？**为了之后的索引做准备**

**主键索引**：
1. 具有主键的表，每张表有一个B+树
2. 若没有主键，mysql会生成隐藏主键（可以认为所有的数据都是线性组织的，隐藏主键的索引效率低，因为主键隐藏，所以无法根据主键进行查找）
3. B+树也是按需加载到内存的

**B+树vsB树**：
B+树的路上节点没有保存实际数据，而只保存目录信息，这是为了使这颗树更加矮胖
B+树的叶子节点相连，利于范围查找
**聚簇vs非聚簇**：
B+树和数据耦合在一起，这种索引被成为**聚簇**索引，典型引擎Innodb
**非聚簇**索引，如MyISAM，叶子节点存储数据的起始地址，根据地址进行二次寻址

**选择索引的两个原则**：1. 列的使用频率高 2. 重复的数据较少（对于非主键索引）
InnoDB中，对于非主键索引，B+树叶子节点存储记录的主键值，先通过非主键索引查询该记录的主键，再进行**回表**，通过主键索引查询整条记录
为什么非主键索引只存储主键值？相比于存储整条记录，更节约空间（时间换空间）
MyISAM中，创建一个索引就构建一颗B+，和InnoDB一样，但是其叶子节点存储的是数据文件的地址指针，不是主键值，因为MyISAM是非聚簇索引

### 索引的相关操作
**主动创建索引**：
```
alter table 表名 add index(列名); // 对表中的列建立索引
```
![image.png](https://s2.loli.net/2023/10/09/PNWpywcBktAsTxI.png)

**被动创建方式**：
添加主键约束后，自动生成主键索引
建表时，添加主键约束
```
create table 表名(id int primary key, name varchar(30));
create table 表名(id int, name varchar(30), primary key(id));
```
建表后，添加主键约束
```
alter table 表名 add primary key(id);
create index 索引名 on 表名(列名)
```
建表后，删除主键约束
```
alter table 表明 drop primary key:
```

唯一索引，添加unique约束后，自动添加唯一索引（普通索引）
删除普通索引：`
```
alter table drop index 含有索引的列名/索引名
```

普通索引Key属性显示为：MUL
![image.png](https://s2.loli.net/2023/10/10/f2zdAijesZ81VF7.png)

查看表的索引
```
show index from 表名 \G
```
![image.png](https://s2.loli.net/2023/10/10/nYR3kE2h5GO4dlT.png)

其中：key_name为索引名，除了主键索引，其他索引的名字都是列名（用alter创建的索引）
![image.png](https://s2.loli.net/2023/10/10/bi26waCS5fJPrv3.png)

用create可以创建自定义的索引名，不过不建议这样
创建复合索引时，需要注意，索引名为第一个索引的名字
![image.png](https://s2.loli.net/2023/10/10/tWcgIbSNQ169u8i.png)

### 全文索引
**创建全文索引**：
```
alter table 表名 add fulltext 索引名(列名);
```
MyISAM支持全文索引，而InnoDB不支持
有大量的查找需要时，使用MyISAM引擎（博客系统），大量的修改需要时，使用InnoDB引擎（购物系统）
全文索引是指：当列信息很大时，where（模糊）筛选该列的效率变得很低，因为每个数据都很大，此时就需要使用全文索引
```
select * from 要搜索的表名 where match(包含文本的列名) against('用来搜索的内容');
```
***
## 事务
> 一个或者多个sql语句的集合就是事务

（直接输入的一条命令也是事务）
事务这个概念不是数据库天然具有的，而是后续引入的，目的是为了简化上层的编程逻辑

> InnoDB支持事务，MyISAM不支持事务

**为什么需要事务？保证操作的原子性**
事务的四个属性：
1. 原子性：事务要么执行完，要么没有被执行，不存在中间态，若事务在执行过程中发生**错误**，将进行**回滚**，回滚到事务执行前的状态
2. 持久性：事务处理完成后，对数据的修改是永久（持久）的，即数据刷新到磁盘，若系统故障也不会影响之前的事务
3. 隔离性：事务之间互相独立，互不影响
4. 一致性：事务开始之前与结束的时候，数据库的完整性不会被破坏，即事务的行为是可预测的

前三个特性由数据库实现，最后一个特性——一致性是实现前三个特性后的表现，也是一个上层概念
从技术上看，其他三个性质保证了一致性。但是从数据上看，一致性由业务逻辑决定，只有业务的逻辑保证了数据的一致，才能体现一致性
### 事务的操作
开启事务：`begin`或者`start transaction`
结束事务：`commit`
设置保存点：`savepoint 保存点名字`
回滚事务：`rollback`，默认回滚到最开始，即使存在很多保存点
回滚到指定保存点：`rollback to 保存点名字`
事务被提交后，不能`rollback`
若事务执行过程中，客户端崩溃，事务自动回滚到最开始

其中事务分为手动提交和自动提交两种方式
查看自动提交是否开启
```
show variables like 'autocommit'
```
关闭自动提交
```
set autocommit=0
```
`begin`启动事务后，无论是否设置自动提交`autocommit`，之后的语句都不会提交，只有执行`commit`后才会提交事务。若`commit`之前发生异常，那么未`commit`的语句将被撤销
自动提交只影响单语句（不使用begin）的sql事务，若关闭自动提交，执行完事务一定要记得`commit`

### 事务的隔离级别
隔离级别有三类：全局隔离级别、会话隔离级别、默认隔离级别
**隔离级别的查看**
```
select @@global.tx_isolation;   // 查看全局隔离级别
select @@session.tx_isolation;  // 查看会话隔离级别
select @@tx_isolation;          // 查看当前隔离级别
```
每次会话时，隔离级别的初始化顺序：全局隔离级别->会话隔离级别->默认隔离级别
**隔离级别的设置**
```
set global transction isolation level 隔离级别;   // 设置全局隔离级别
set session transaction isolation level 隔离级别; // 设置会话隔离级别
set transaction isolation level 隔离级别;         // 设置默认隔离级别
```
隔离级别为：read uncommitted、read committed、repeatable read或 serializable中的一个

会话隔离级别指的是：本次与mysql连接（会话）设置的隔离级别，不会影响下一次会话
全局影响会话和默认，会话影响默认，但不影响全局隔离级别

设置全局隔离级别后，需要退出重新登录，此时会话隔离级别和默认隔离级别才会被修改
设置会话隔离级别后，直接影响默认隔离级别

### 隔离级别
- 读未提交(read uncommitted)：一个客户端的事务执行到一半，另一个客户端能看到你未提交的数据。隔离级别最低，等价于没有隔离，容易引起并发问题，读到其他客户端“读未提交”的数据时，被称为“脏读”
- 读提交(read committed)：也叫不可重复读，一个客户端的事务提交后，不论其他客户端是否在事务中，都能看到提交的数据。若事务未提交，其他客户端无法看到未提交的数据。该隔离级别下，不同时间点读取的数据可能会不同
- 可重复读(repeatable read)：只要当前事务不提交，无论何时读取到的数据都是一样的。有些数据库在可重复读状态下，无法屏蔽其他事务的insert提交，此时读取到的数据与之前相比增加了，这种现象称为幻读。而mysql解决了幻读问题
- 串行化(serializable)：除了读取语句select之外，其他事务都会串行执行，不允许并发，即一个事务未提交之前，其他事务的更新或者插入操作都将被阻塞，直到该事务提交

RR在并发访问时，多个客户端进行修改操作，只要不更新出**相同主键的记录**或者**同时修改同一条记录**就不会导致阻塞

### 3个记录隐藏列字段
向表插入记录时，还插入了一些不可见的数据
- DB_TRX_ID：6 byte，最近修改/插入的事务ID
- DB_ROLL_PTR：7 byte，回滚指针，执行这条记录的上一个版本，一般在undo log中
- DB_ROW_ID：6 byte，隐含的自增ID（隐藏主键），若没有指定主键，InnoDB会自动以DB_ROW_ID产生一个聚簇索引
- 还有一个隐藏字段flag：用来表示数据是否被删除

日志不仅仅是执行记录，mysql中的日志具有功能性和数据保存能力
以下说法不准确，但是好理解：
修改（update）记录时，加锁，将数据拷贝到undo log中，原数据的DB_ROW_PTR指向undo log中拷贝记录的地址，修改原数据的DB_TRX_ID为修改其的事务ID，并修改该记录
若其他事务或者当前事务再修改该记录，那么undo log中的拷贝记录和被修改记录之间就会形成一条版本链，由DB_ROW_PTR链接
那么回滚的本质就是用版本链中的记录覆盖当前记录

但undo long保存的是与修改语句相反的语句（不是上一条记录），回滚操作就是不断执行相反操作
这些相反操作为版本链中的一个个版本，也被称为快照
若当前事务commit，所有快照被清理

insert不会形成版本链，update和delete会形成版本链
为select维护版本链没有意义，但是select读取的是什么时候的数据？
读取分为两种：当前读与快照读。当前读需要加锁，而增删改也是一种特殊的当前读
由隔离级别决定当前读与快照读，快照读时，可以做到**读写分离**，提高并发性
虽然事务的执行顺序可能交织在一起，但事务的到来时间有先有后，事务到来的先后顺序由事务ID体现，越新的事务，ID越大
理论上说，先来的事务不应该看到后来事务的修改，后来的事务应该要看到先来事务的修改，让不同事务看到该看的内容。但让不同事务看到什么程度，由隔离级别决定

事务在读取时，会生成一个**Read View类**的对象，用来决定事务的可见版本
几乎同时启动的事务，后者能否看到前者**修改并且提交**的记录，取决于后者什么时候进行快照读（Read View对象何时生成）
Read View中存在这几个成员变量：
```
m_ids;          // 保存Read View生成时，mysql中活跃（begin但未commit）的事务ID
up_limit_id;    // 保存m_ids中的最小值
low_limit_id;   // 保存Read View生成时，系统尚未分配的最小事务ID，即已经分配过的事务ID+1
creator_trx_id; // 创建该Read View的事务ID
```
在RR隔离级别下，第一次select操作将形成快照，生成Read View对象。接着select将根据记录的历史版本链（含有对该记录操作过的事务ID），读取**最近可见**事务ID的修改记录
可见ID：
- ID小于`up_limit_id`（比活跃ID的最小值还小）
- ID大于等于`low_limit_id`（比活跃ID的最大值还大）
- `m_ids`为空（形成快照时，没有活跃事务）
- ID没有出现在m_ids（不是活跃ID）

在所有可见ID中选择最近（最后一次修改该记录）的版本读取
所以在RR隔离级别下，select快照读读取的是哪个版本的记录，取决于第一次select的时间
而RR和RC的区别在于是否会重复形成Read View
- RC隔离级别下，每一次select都会形成快照
- RR隔离级别下，只有第一次select会形成快照

正是因为RC每次select时都会形成快照，所以每次读取的数据不同（每次的活跃ID不同，可见范围随之变化），产生了不可重复读问题
而RR只有第一次select时才会形成快照，所以没有不可重复读问题
***
## 用户管理
mysql自带一个名为`mysql`的数据库，其中存在一张名为`user`的表，里面存储了当前mysql下的用户信息
进入名为`mysql`的数据库，输入
```
select * from user\G
```
可查看所有的用户信息
![image.png](https://s2.loli.net/2023/10/12/8EKmdRtMLk7BzCU.png)
user：用户名
host：用户的登录IP，localhost表示本机（127.0.0.1）
authentication_string：经过md5摘要后，用户的登录密码

所以，所有对用户的管理工作，本质就是对user表进行的增删查改
我们可以直接insert，以添加用户，但是这样要设置很多其他的字段同时容易写错，不推荐这样做

mysql提供了一系列的语法以更快的添加/管理用户

**创建用户**
```
create user '用户名'@'登陆主机/ip' identified by '密码';
```
![image.png](https://s2.loli.net/2023/10/12/Ttdorsmazh7LMpC.png)
ip可以是localhost，也可以是%(所有IP)，也可以是具体的IP

若添加用户失败，则执行刷新当前权限
```
flush privileges
```

**删除用户**
```
drop user '用户名'@'主机名'
```

**修改用户密码**
```
set password for '用户名'@'主机名'=password（'新的密码') 
alter user '用户名'@'主机名'identified by '新的密码'  // 推荐使用
```

### 权限修改
```
grant 权限列表 on 库名.表名 to '用户名'@'主机名' [identified by '密码'](可省略)
```
表名用\*代替，表示该库下的所有表
若权限列表有多个权限，那么权限之间用`,`分隔，用all代替权限列表，表示所有权限
关于具体的权限，大概列举几个：
![image.png](https://s2.loli.net/2023/10/13/N8OqFZxiejM7cLw.png)

**显示用户权限**
```
show grants for '用户名@主机名'
```

**回收用户权限**
```
revoke 权限列表 on 库.对象名 from '用户名'@'主机名'；
```
执行以下命令，使修改生效
```
flush privileges
```
***
## 使用C语言链接数据库
搜索`linux发行版本+安装C语言连接mysql的API库`，下载相关环境
`rpm -qa | grep mysql`或`dpkg -l | grep mysql`以查看系统中是否已经安装了相关安装包
`ls /usr/include/mysql`，查看相关库的头文件

连接mysql之前需要初始化MYSQL结构体，与FILE类似
`MYSQL *msql = mysql_init(NULL)`，参数一般是空指针
若函数返回NULL，表示是初始化失败
`mysql_close(MYSQL *msql)`，不需要该结构体时，要释放该结构体

初始化MYSQL后，连接mysql
```cpp
MYSQL *mysql_real_connect(MYSQL *mysql, const char *host,
								const char *user,
								const char *passwd,
								const char *db,
								unsigned int port,
								const char *unix_socket,
								unsigned long clientflag);
```
- mysql：初始化成功的MYSQL结构体
- host：mysql服务端所在的IP地址
- user，passwd：用来登录的用户名与密码
- port：mysql对外开放的端口号
- unix_socket，clientflag：相应地设置为nullptr和0即可
函数返回空指针表示连接失败

```cpp
mysql_set_character_set(MYSQL *mysql, char *charset)
```
有时会遇到字符集（编码格式）不匹配的情况，连接成功后需要设置当前字符集，通常设置为utf8，`mysql_set_character_set(msql, "utf8");`

```cpp
mysql_query(MYSQL *mysql, char *query*)
```
query：需要执行的sql语句，与命令行操作输入的语句相同
函数返回0表示语句执行成功
使用`mysql_query`进行增删改时，通过返回值是否为0判断操作是否成功即可
但是使用`mysql_query`进行查操作（如select）时，就比较麻烦
查操作时，若mysql_query成功，那么查询结果将保存到一开始初始化的MYSQL结构体中
```cpp
MYSQL_RES *res = mysql_store_result(MYSQL *mysql);
```
以该结构体指针作为参数调用`mysql_store_result`函数，将查询结果保存到MYSQL_RES结构体中
```cpp
int rowcnt = mysql_num_rows(MYSQL_RES *res);
int fieldcnt = mysql_num_fields(MYSQL_RES *res);
```
以该结构体指针为参数调用`mysql_num_rows`和`mysql_num_fields`，返回查询结果的行数与列数
```cpp
MYSQL_FIELD *fname = mysql_fetch_fields(res);
```
以该结构体指针为参数调用`mysql_fetch_fields`函数，列属性的名称将被保存到MYSQL_FIELD结构体中，fname相当于一个数组，数组成员为`MYSQL_FIELD`，该结构体中的name字段为列属性的名称，即`fname[j].name`表示第j列的列名

```cpp
MYSQL_ROW row = mysql_fetch_row(res); // 注意参数名的row没有s
```
以该结构体指针为参数调用`mysql_fetch_row`函数，将行信息保存到结构体`MYSQL_ROW`中，这里用结构体对象接受函数返回值，可以将`MYSQL_ROW`看成一个数组，长度为列的数量，`row[j]`第j个成员为第j列的数据，以字符串的形式保存
***
以下是一个简单使用的demo，该程序将读取标准输入的数据，若是select语句，则将结果打印到标准输出中
```cpp
#include <iostream>
#include <cstdlib>
#include <cstdio>
#include <cstring>
#include <string>
#include <mysql/mysql.h>
using namespace std;

string host = "127.0.0.1";
string user = "root";
string password = "xx557223";
string db = "test_db";
unsigned int port = 3306;

int main()
{
    // cout << "version: " << mysql_get_client_info() << endl;
    MYSQL *msql = mysql_init(nullptr);
    if (msql == nullptr)
    {
        cerr << "init fail\n";
        return 0;
    }
    if (mysql_real_connect(msql, host.c_str(), user.c_str(), password.c_str(), db.c_str(), port, nullptr, 0) == nullptr)
    {
        cerr << "登陆失败\n";
        return 0;
    }
    mysql_set_character_set(msql, "utf8");
    // string sql = "insert into class_tb values(22, '一年b班', 'xxx')";
    // string sql =  "select * from class_tb";
    char str[1024];
    while (true)
    {
        cout << "mysql> ";
        fgets(str, sizeof str, stdin);
        int t = mysql_query(msql, str);
        if (strcasestr(str, "select") && t == 0)
        {
            MYSQL_RES *res = mysql_store_result(msql);
            int rows = mysql_num_rows(res);
            int fields = mysql_num_fields(res);

            MYSQL_FIELD *fname = mysql_fetch_fields(res);
            for (int j = 0; j < fields; ++ j) cout << fname[j].name << "\t|\t";
            cout << "\n";
            MYSQL_ROW line;
            for (int i = 0; i < rows; ++ i)
            {
                line = mysql_fetch_row(res);
                for (int j = 0; j < fields; ++ j)
                    cout << line[j] << "\t|\t";
                cout << "\n";
            }
        }
    }
    // 关闭
    mysql_close(msql);
    return 0;
}
```
