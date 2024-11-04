## 名称映射规则
结构体名称->表
结构体成员->表字段

有三种不同的IMapper：
![QQ_1730699046189.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730699046189.png)

可以使用其他的方法修改映射规则：
![QQ_1730699460850.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730699460850.png)

xorm有默认的类型对应：
![QQ_1730699748604.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730699748604.png)

但可以直接在tag中直接定义字段的属性，其中也能定义字段的类型：
http://xorm.topgoer.com/chapter-02/4.columns.html

以及一些默认的字段映射规则：
![QQ_1730699922255.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730699922255.png)

## 信息获取
engine.DBMetas()将获取数据库中所有表的信息，返回一个切片`[]*Table`
## 数据同步
一般使用Sync2()来同步数据
## 数据插入
1. 防止插入的数据条数过多
2. 批量插入的操作不是事务
## 数据查询
Get()需要传入一个对应结构体指针，其中的非空field将自动成为查询条件，和前面的条件组合在一起查询

相比于Get(), Exist()的性能更好，它用来判断某个数据是否存在
## 数据更新
乐观锁不是为了保证原子性，而是为了保证并发环境下，业务逻辑存在的数据覆盖问题。比如一个事务需要给工资+1000，另一个事务+2000，原工资为1000，那么结果一定为4000。不可能是2000或者3000
## 数据删除
软删除一般比实际的删除快。

