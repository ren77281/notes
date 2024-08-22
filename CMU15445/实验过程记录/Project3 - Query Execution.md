```toc
```
每个执行器（也就是课上讲的算子），需要实现Next方法。Next讲返回一个tuple，或者是表示没有tuple的指示符。每个执行器需要在一个循环中，不断调用子节点的Next函数，拽出tuple后，一个一个地进行数据处理
在BusTub中，Next函数返回行id(RID)，这是一种延迟物化的方式。每个tuple的RID都是不同的
执行器被执行计划创建，执行计划实现在`/src/execution/executor_factory.cpp`
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407170846477.png)
主要是执行器的实现，以及一些函数与优化器
## Task0 - Read The Source Code
### Schema
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171636005.png)
表的结构
length_：一个tuple的**列属性**占用的字节数，对于non-inlined属性，只会计算其指针大小
columns_数组：存储Column，Column表示列属性
tuple_is_inlined_：是否所有tuple都是inline
uninlined_columns_：non-line的列号

需要注意：length_不是指tuple的字节数，而是其所有列属性的字节数。Value的GetLength则是具体列的字节数
inline数据：列属性的字节数和具体数据的字节数相同
non-inline数据：则不同
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407180939609.png)
不懂为什么要+4

ToString：简约模式将调用Column的ToString的简约模式，每个Column的String以`,`分割
详细模式，列数量，是否内联，长度，后面跟着每一列的ToString

`CopySchema(const Schema *from, const std::vector<uint32_t> &attrs)`
拷贝一张表的某些属性
### Column
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171540012.png)
column_name_：列名
column_type_：列的类型
fixed_length_：如果是inline类型，表示该类型所占字节数，如果是non-inline类型，表示其指针所占字节数
variable_length_：如果是inline类型，表示0，如果是non-inline类型，表示该类型所占字节数
column_offset_：在tuple中的偏移量，Schema存储Column时，将设置该变量
在BusTub中，non-inline只表示表示varchar类型
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407170935922.png)

ToString：简约模式只返回列名与列的类型
详细模式则返回列名，类型，偏移量，长度，并且每个值之前打印该值的说明
### Catalog
似乎保存了库下不同表的元信息？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171646789.png)
tables_：oid到TableInfo的映射
table_names_：表名到oid的映射
next_table_oid_：与BufferPool的next_page_id类似，表示下一个可使用的talbe_oid
indexes_：oid到TableIndex的映射
index_names_：索引名到oid的映射
next_index_oid：下一个可用的index_oid
#### TableInfo
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171418948.png)
schema_：表的结构
name_：表名
table_：指向TableHeap的指针
oid_：表的ID
#### TableHeap
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171421289.png)
bpm_：缓存池
first_page_id_与last_page_id_：分别指向第一张表和最后一张表
latch_：保护最后一张表
这个Heap是之前课上讲过的：DBMS管理page的方式，根据TablePage的实现，可以知道page之间以链表的方式连接。似乎能将TablePage理解成链表的节点，而TableHeap是一个链表？
#### TablePage ☆☆☆
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171427140.png)
和之前实现的page_guard放在同一个文件夹下，将获取的Page资源视为Table结构
page_start_：实际存储数据区域的首地址，这是一个占位符
next_page_id_：当前table使用的下一张page的id，通过链表的方式连接每个TablePage
num_tuples_：当前table的tuple数量
num_deleted_tuples_：当前table删除的tuple数量
tuple_info_柔性数组：std::tuple，保存了tuple的元信息——tuple在page中的偏移量，字节大小，TupleMeta

TablePage的布局：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171521772.png)
为什么HEADER_SIZE为8字节？看TablePage的成员，第一个成员是个占位符，不占空间。接下来四个成员就是Header的组成，一共是8字节。最后一个成员为柔性数组，数组长度可变

以下函数框出部分是在获取上图中的free_space_pointer：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171451855.png)
减去tuple的字节大小后，就能得到插入该tuple后，free_space_pointer的位置
之后定义的offset_size则是在获取表头Header+tuple元信息的偏移量，tuple元信息用TupleInfo表示
#### TupleMeta
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171447890.png)
ts_：tuple被创建的时间
is_deleted_：tuple是否被删除
#### RID
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171502704.png)
page_id_：tuple所在page的id
slot_num_：tuple所在page的slot号，用来标识tuple（其实叫id更好，但是num与id数值相等，这样也没错）

根据无符号64位整数rid，创建RID对象，高32位page_id，低32位为slot_num_，也就是tuple在page中的下标
重载了<<运算符，可以直接打印RID
#### Tuple
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171526794.png)
rid_：唯一标识tuple的信息
data_：用来存储实际的数据
tuple的两个序列化，反序列方法
将data_的大小以及实际数据序列化到storage以及从storage反序列化数据到data_中
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171532400.png)
GetValue函数：根据schema和列号，获取存储数据的指针以及数据类型，最终获取序列化后的Value值
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171629318.png)
而Value的背后是Type，上面的序列化函数最终调用的是Type的序列化：知道了数据类型才知道如何序列化
使用Value构造tuple对象：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407180916380.png)
首先，这些Value一定是tuple的列值
#### Value
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171559827.png)
value_：联合体，最大8字节，存储实际的数据
size_：联合体，表示value_的大小。这里猜测只有指针（non-inlined）类型的大小为len_，其他类型的大小为elem_type_id_
manage_data_：不知道什么用
type_id_：value_的类型
##### TypeId
枚举类型
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171255689.png)
##### Type
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171607171.png)
type_id_：表示具体的类型
k_types：静态数组，每个成员都是一个封装好的类
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171611744.png)
经常用到的函数：获取类型实例
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171615991.png)
### ExecutorContext
运行执行器所需的数据
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171009425.png)
catalog_：保存了TableInfo，通过表名能获取到表的所有信息
### AbstractExecutor
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407180821518.png)
所有执行器的基类，作为纯虚函数，需要重写其Init，Next，GetOutputSchema方法
exec_ctx_：保存了执行器运行所需的数据
具体的执行器有自己的执行计划与子节点：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407180842208.png)
### AbstractPlanNode
执行计划（火山模型）中的节点，火山模型是一颗树，节点之间以树形方式连接，每个节点都有指针指向子节点：`std::vector<AbstractPlanNodeRef> children_`
节点需要向上输出tuple，为了知道tuple的结构是什么样的，就需要保存：`SchemaRef output_schema_;`
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407180826550.png)
BusTub中，有哪些具体的执行计划？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171032524.png)
具体的执行计划中，还有`表达式`成员
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407180833204.png)
### AbstractExpression
两个成员
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407171300796.png)
通常用树表示一个表达式，AbstractExpression则是树中的一个节点
children_：当前节点的子节点
ret_type_：以当前节点为根的子树，返回数据的类型是什么
似乎以下两个函数是用来进行表达式计算与判断是否能进行等值连接的
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407180839758.png)

总结获取tuple的过程：通过executor的ExecutorContext获取catalog，得到TalbeInfo，进而获取TableHeap。通过TableHeap就能获取到所有TablePage，而TablePage则保存了具体的tuple
调用TableHeap的MakeIterator接口，获取TableIterator
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407181319814.png)

## Task1 - Access Method Executors
### SeqScanExecutor
添加`TableIterator`成员
- 初始化`TableIterator`，使之指向table的第一条tuple
`Next`：
Fall 2023在`SeqScanPlanNode`中添加了谓词，也就是说明`SeqScanExecutor`算子在扫描表的时候还能筛选value
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407181508932.png)
所以我们不仅需要判断tuple是否被删除，还需要判断tuple是否满足筛选条件
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407181512143.png)
`TableIterator`的`GetTuple()`将返回`<TupleMeta, Tuple>`，通过TupleMeta的is_deleted_判断是否被删除
如果执行计划中存在谓词筛选：`this->plan_->filter_predicate_ != nullptr`，调用其`Evaluate()`方法，`GetAs()`转换其返回值，判断是否满足筛选条件

bug：需要注意vector的迭代器失效问题
### InsertExecutor
该执行器具有子执行器，通过子执行器的`Next()`方法获取需要Insert的tuple
`Init`：
- 初始化子执行器
`Next`：
- 调用TableHeap的`InsertTuple()`接口
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407181531024.png)
- 通过catalog获取所有索引（`GetTableIndexes()`），维护每一条索引（`InsertEntry()`）
- insert的过程需要统计循环次数，最后用来构造tuple，作为输出型参数，表示成功insert的记录数量

接收`InsertTuple()`接口的返回值RID，rid不是通过子执行器的`Next()`返回
### DeleteExecutor
具有子执行器，其将返回需要删除的rid，tuple没有什么意义
DeleteExecutor需要将所有指定tuple删除完再返回。其tuple只有一个INTEGER属性，需要将成功删除的tuple数量返回
这里似乎有点问题，我没有判断是否成功删除。但是子执行器获取的rid，一定是存在于表中的，所以这里无关紧要？
更新MetaTuple的is_delete_即可
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407181916912.png)
Init：
- 初始化子执行器
Next：
- 通过`TableHeap`以及rid获取其Meta（`TupleMeta()`）
- 删除tuple：将修改后的Meta更新回去（`UpdateTupleMeta()`）
- 维护索引（`GetTableIndexes()`, `DeleteEntry()`）

delete删除索引时，需要传入索引的key，此时需要通过tuple的KeyFromTuple函数得到keys，问题是调用谁的KeyFrom函数，这里应该通过子执行器获取需要删除的rid，再通过rid获取其tuple，调用该tuple的KeyFrom函数，而不是子执行器返回的tuple，文档没有规定子执行器需要返回tuple
### UpdateExecutor
具有子执行器，其将返回需要update的rid，tuple没有什么意义
UpdateExecutor需要将所有指定的tuple更新完再返回。其tuple只有一个INTEGER属性，需要将成功更新的tuple数量返回
Init：
- 初始化子执行器
Next：
- 通过`TableHeap`以及rid获取旧tuple（`GetTuple()`）
- 删除tuple：将修改后的Meta更新回去（`UpdateTupleMeta()`）
- 维护索引（`GetTableIndexes()`, `DeleteEntry()`）
- 不断调用执行计划的`target_expressions_`，保存更新后的tuple
- 插入新的tuple并维护索引`InsertTuple()`, `InsertEntry()`

执行计划的target_expressions_包含了所有tuple的列的expression，列的顺序和原tuple相同，我们需要Evaluate每个expression，保存每个列的值，最后用这些列值构建tuple
需要注意：子执行器Next()返回的tuple没有意义，不要使用它。有些函数调用需要传入schema，不要使用子执行器返回的tuple的schema

### IndexScanExecutor
通过hash索引获取rid，通过rid获取tuple并返回，每次只返回一条tuple
如果table存在索引，可以执行以下代码，将其转换成P2实现的hash索引：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407190851135.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407190901208.png)
执行计划告诉了我们需要查询的表id以及索引id
调用hash索引的`ScanKey()`即可
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407190915322.png)

优化规则：
- 只有一个谓词条件
- 谓词条件所在列具有索引
- 谓词是等值条件

调试单点查询，看单点查询的where将调用哪个表达式，如果表达式的子节点数量为2，且第一/二个子节点的子节点数量为0？，说明有一个节点是常量
优化器实现需要参考已经实现的其他优化器代码
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407191154423.png)
我们需要先将执行计划进行动态类型转换，得到具体的执行计划
由于扫描算子的执行计划一定没有子节点，所以我们不需要对子执行计划进行优化。只需要优化当前执行计划
什么样的执行计划有孩子，如Join算子的执行计划，对超过两张表进行Join，生成的逻辑执行计划至少有两个Join算子
### OptimizeSeqScanAsIndexScan
根据seq->index的优化规则，我们需要判断扫描算子的执行计划相应表达式
首先表达式类型肯定是ComparisonExpression
其次表达式应该只有一个谓词，所以表达式只有两个孩子，分别表示用来比较的两个对象
这两个对象也是表达式，其中一个表达式的值应该为常量
比如where colA = 1，但是where colA = 0+1，是否应该判断呢？
这里就不判断了当作一个可优化的点
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407211418249.png)
（官方都没优化，那没事了）

根据C++的dynamic_cast转换，如果转换失败，dynamic_cast将返回nullptr
也就是父类指针指向子类A，可以将父类指针dynamic_cast成子类A的指针
但是dynamic_cast成子类B的指针将导致失败转换（nullptr）
**这是dynamic_cast的一个很重要的使用：多态类型之间的安全转换**
dynamic_cast将在运行时进行安全检查
如果是引用类型之间的转换，则需要通过抛异常，判断是否能转换成功

那么我们可以如何检查表达式的一侧是否为常量，通过将对应表达式进行转换，判断是否能dynamic_cast成功
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407191235156.png)
测试结果如下：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407191236066.png)
同理，另外一侧可以转换为ColumnValueExpression

以tuple的方式获取table的key
这个函数的参数为：
- schema：表结构
- key_schema：只留下索引的表结构
- key_attrs：在schema中，索引列的下标
需要注意：schema应该根据table_info获取，而不是通过this的GetOutputSchema获取
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407191434210.png)

ColumnValueExpression存了列号
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407191445122.png)
只需要获取表的所有索引，通过index_info->key_attrs判断是否命中索引

bug：还需要判断表达式类型为ComparisonExpression的Equal
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407211341275.png)

## Task2
### AggregationExecutor
`AggregationPlanNode`只有一个孩子

输出schema包含了group-by的列和聚合列？为什么还包含了group-by列？
常用的聚合算法为hash表，以group-by的列作为key值。我们可以假设用来聚合的hash表可以存储在内存中。我们可以直接使用hash表，而不经过buffer pool
为什么以group-by的列作为key值？group-by需要将相同列值的tuple分到一组
官方提供了一个in-memory的hash表，该结构还暴露出计算聚合的接口，我们需要实现`CombineAggregateValues`
aggregate算子不支持处理having子句，所以相应的执行计划具有`FilterPlanNode`，用来过滤tuple
那么首先需要实现的就是hash表的`CombineAggregateValues`
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407200839255.png)
表示正在进行聚合的值
五种agg类型，只有CountStarAggregate类型才会返回0，其他类型的agg返回null值

SimpleAggregationHashTable：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407200841884.png)
ht：存储聚合结果
agg_expres_：aggregate的表达式，可能是having？
agg_types_：
`aggregation`的类型：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407191938336.png)

将k-v插入hash table中，调用了`CombineAggregateValues`，该函数由我们实现
解释下逻辑，如果hash table中，没有key值，那么插入key与默认值

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407201023023.png)

需要注意的是，key为group-by的列值，而value为aggregate的列值
Combin则需要根据input，计算当前聚合结果到result中，由于可能有多个聚合列，所以这里需要遍历每个聚合列，计算不同聚合列的结果

aggregate算子是`pipeling breaker`，为什么这么说？聚合运算需要获取所有的tuple才能聚合结果输出，如`max`, `min`聚合函数。所以在执行agg算子时，数据流将被其阻塞
我们需要在Init函数中，获取所有tuple并保存聚合结果
在Next函数中，没有使用到rid参数，我们需要将聚合结果进行输出。schema为group-by与aggregate的列结合
agg的子执行器将进行filter过滤，所以agg算子只需要聚合数据即可

group-by其实是一些列值表达式，我们通过对tuple执行列值表达式获取列值，然后以获取的列值作为key，以tuple或者RID作为value，进行hash映射。这样的话，相同列值的tuple一定被分到了相同的桶中，
在agg中，依然以group-by的列值作为key，但是以agg的列值（可能有多个）作为value。也就是说聚合结果为value
如果没有group-by，所有的tuple的key值都会为空，聚合结果将根据这些tuple产生
问题是为什么输出结果需要使用key-value的拼接？
key和value都是多个Value的组合，为什么？因为可能有多个用来group-by的列，也有多个需要aggregate的列
MakeAggregateKey和MakeAggregateValue只是根据列值表达式，获取tuple的对应列值
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407201329040.png)
输出的schema需要包含group-by与aggregate列

所以Next函数需要返回一条tuple，包含了group-by列与aggregate列的值。我们需要使用it遍历hash表，将获取的key值（多个group-by列值）与value（多个agg列值）组合在一起（`std::vectot<Value>`），返回构造Tuple返回

**bug：**
CheckInteger这个函数用来检查value是否为整数，但如果是整数的null，也能通过检测
所以需要判断value是否为null，应该通过IsNull检查
**需要基于空表进行特判**，输出聚合默认值
Init函数没有初始化子执行器，导致无法进行嵌套agg
### NestedLoopJoinExecutor
pipeline breaker：在Init函数中执行Join
- 先保存右表
- 遍历左表的同时，遍历已经保存的右表结果
- 调用执行计划的predicate的EvaluateJoin，判断是否能进行Join

对sql_scan做了修改，可能有问题
我实现的Join操作将阻塞数据流动

bug：sql_scan实现中，第一次scan时未保存tuple到tuples中
有两个地方需要保存tuple，而我只在一个地方保存了

巨大bug：NLJ中，内循环执行了很多次，为什么不会执行右执行器的Next？？？
解决：如果算子多次调用sql_scan，需要自己保存结果。只能尽量避免这个问题了

### HashJoinExecutor
output schema为左表的所有列和右表所有列
我们可以假设hash join的输出结果能在内存中存储
该任务的自由度较高，我们需要仿照`SimpleAggregationHashTable`实现一个自己的hash table，用来存储左表的hash结果。hash过程需要在Init函数中完成
接着需要获取右表的tuple，用其join attribution查询已经构建好的hash table
如果hash table存在相同key值，我们需要连接两个tuple，连接过程和NLJ相同
但是我们要如何获取左表的tuple？所以这里应该将左表的tuple作为key存在hash桶中
此外，由于存在hash冲突的问题，具有相同key值的join attribution可能并不相同，我们需要从value中取出join attribution，进一步判断是否发生了hash冲突
这里以链地址法解决hash 冲突，将相同hash值的tuple存储在vector中，所以hash table的value为`std::vector<tuple>`。
那么已经实现的`SimpleAggregationHashTable`，是如何解决hash冲突的？
`unordered_map`使用链地址法解决hash冲突，每个hash桶都是一张链表，存储的结构是`pair<key, value>`，因为存储了key所以才能进一步进行匹配
但是我们不需要存储key，因为key在value，也就是tuple中

我需要在plan中实现提取，左右tuple的key
`HashJoinPlanNode`中`left_key_expressions_`，`right_key_expressions_`应该是计算左tuple的key值表达式，所以应该是列值表达式
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407211004317.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407211004715.png)
其返回值为指定列上的Value
这么说来，Join算子中，可能有多个Join条件，所以这里的有多个表达式，使用了`std::vector`存储
并且左右两个expressions的数量应该是相同的，两者都表示了Join Column

但是存在一个问题：如果存在两个Join Col，这两个Join Col在两表中的相对顺序不同，那么这两个列值在left_key_expressions_和right_key_expressions_中是否会以正确顺序保存
根据优化器的优化结果：似乎不存在这样的问题
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407211010697.png)

所以根据这两个expressions，我们可以在plan中实现`GetLeftJoinKey`与`GetRightJoinKey`
语法问题：重写unordered_map的hash函数后，还需要重载其比较方法`operator==`

left join没有实现左连接，哈希冲突由ht_解决，上层不需要解决

当nlj有多个等值表达式时，我们需要将其优化成hash join（似乎只有一个等值表达式时，也会进行优化，那为啥要实现nlj啊）
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407211337827.png)
其实：hash join只能在多个**等值**表达式时才能进行优化，非等值表达式则不能优化。这里还发现了之前的一个bug
所以我们需要判断nlj的表达式类型是否为逻辑/等值
如果是逻辑表达式，那么它两的两个孩子也必须是逻辑/等值
直到所有的孩子类型为等值，这是一个递归的过程，估计要封装一个递归函数
递归退出条件：当前表达式类型为等值，此时将两个列值表达式插入到left_expressions和right_expressions中即可
如果表达式类型为逻辑，则继续递归判断，一旦出现表达式类型非逻辑/等值直接返回false
此时优化不能再进行

现在已经实现了递归分解表达式，我们只需要将nlj的表达式传给该函数，如果函数返回true，说明这是一个等值比较表达式，我们可以进行优化，同时left_expr和right_expr也会被修改
分解等值表达式时，需要注意列值表达式属于哪张表？
通过表达式的tuple_idx，我们可以直到表达式属于左表还是右表
这里需要注意两者的顺序是否相同？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407211450768.png)
上面这句话的意思是：我们先要对nlj进行merge，进行filter的过滤

bug：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407211524695.png)
在unique_ptr转换成shared_ptr时，应该使用move，而不是使用get()
被注释掉的是错误的
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407211525433.png)
## Task4
### SortExecutor
我们就要假设内存能存储下表
同时排序不应该修改原来的表，我们需要将表存储在中间结果，对中间结果进行排序
通过order_bys提取出需要排序的key
使用`std::sort`排序即可
如果没有order by子句（升序/降序），默认升序

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407211621776.png)
表达式应该是列值表达式或者逻辑表达式（几个列值相加），调用Evaluate得到最终值即可

- 保存所有tuple到`std::<Tuple>`中
- 重载Tuple的比较器
如何重载：利用plan_的Evaluate获取Tuple的sort_key，由于sort_key的类型为Value，我们需要调用Value的比较`CompEqual`，根据结果返回true/false即可
需要特判两者相同，应该往后继续比较。如果所有的key都相同，那么返回true，false都无所谓

### TopNExecutor
需要将sort+limit优化成topn
重载priority_queue的比较器，这里要修改比较器规则，根据order-by的类型，如果是asc（最小的），需要建立大堆。如果是desc（最大的），需要建立小堆
这里可以在比较器内部进行asc和desc的判断
asc时，若满足小于关系，返回true（建大堆，找最小数）
desc时，若满足小于关系，返回false（建立小堆，找最大数）

优化器实现：如果一个执行计划是limit，且子执行计划是sort，那么将其转换成topn

### WindowFunctionExecutor
分组，排序，帧
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407212011926.png)
如果帧被忽略， 那么聚合函数将从分组的第一行聚合到当前行
如果帧和排序被忽略，那么聚合函数将从分组第一个行聚合到最后一行
如果都被忽略，那么将从第一行聚合到最后一行

我们只需要实现具有order by和partition
- 先进行order by，复用之前的排序算子

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407212113887.png)
这个function_表达式是什么？？？
应该是over之前的表达式，那么表达式类型很可能就是列值表达式，也有可能是逻辑表达式。但是表达式最终都会算出一个值
当窗口函数的类型为聚合函数类型，可以将其视为聚合函数，调用agg算子处理即可
partition_by_等于group-by
order_by_等于sort，因为sort attrbution可能有多个，所以sort的expression使用了vector存储。并且还需要存储order-by类型（asc，desc）

看agg和sort算子，如何构造并调用？
最终连接成结果
如果窗口函数的类型为RANK，则保证了order-by子句一定不为空，也就是进行了排序。为什么要这么保证

首先需要明白窗口函数是什么？
与普通聚合函数不同的是：聚合函数计算一组tuple，并输出单行结果
而窗口函数虽然也是计算一组结果，但是输出tuple的数量和原表tuple数量相同。在不改变原表schema的基础上，在每一行添加聚合结果。就想给房子开窗一样，在原有的基础上进行添加

所以，如果查询语句中有窗口函数，那么相应执行计划的column将使用占位符替代
理解官方给出的注释
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407220914428.png)
也就是说，注释符的数量等于窗口函数的数量
等同于parition_bys，order_bys，functions的数量

我们需要为每个窗口函数生成agg算子，获取并保存聚合结果，用来代替占位符
值得注意的是，窗口函数中存在order_by子句，排序将改变临时表的顺序，每个窗口函数的order_by key可以不一样，那么我们在agg之前就需要重新排序
但是官方明确指出：每个窗口函数的order by key都相同。所以无论有几个order by子句，我们都只需要进行一次排序

回到窗口函数的具体实现中：
- 如果存在order by，我们先对临时表进行排序
- 对于每个窗口函数，我们都需要创建agg算子，获取并保存聚合结果
- 生成results表，对于其中的窗口列，我们使用empty value填充
- 在Next函数中，用聚合结果替代empty value

那么就能解释：为什么要用`unordered_map`存储window_functions_，不用vector？
这个诡异的key：uint32_t是什么？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407220928288.png)

首先一条SQL中可能有多个窗口函数，要保存多个window_function就要使用vector, unordered_map这样的容器
uint32_t表示一条查询语句中，窗口函数所在的col idx，这些col可能是分散，不连续的。我们无法预知用户输入的sql中，窗口函数所在位置，所以使用unordered_map存储不同列的窗口函数信息

通过这个key，我们也能得知窗口函数所在的col idx
那么在构造最初results时，我们就能根据col idx，构造empty value

如果存在order-by，我们需要将当前子执行作为sort算子的子执行器，并将算子算子作为我们的子执行器
现在的问题是：如何从原tuple中获取要输出的tuple？
首先columns_表示的是当前SQL的列表达式，也就是包含了非窗口函数列的表达式与窗口函数列的表达式
通过执行计划的columns_，对于窗口函数列，column和WindowFunction中的function_相同
对于非窗口函数列，column为子执行器相应列上的expression，也就是说我们需要调用其Evaluate方法，将原表的schema和原tuple传入，提取原tuple的Value

做个测试：对于非窗口函数列，columns中非窗口函数列的列表达式的col_idx为该列在原表的col_idx
columns_存储的列值表达式，其中col idx不是该列在当前schema的idx，而是在原schema的idx
实验如下：
```cpp
auto col = dynamic_cast<const ColumnValueExpression*>(this->plan_->columns_[3].get());
std::cout << col->GetColIdx() << '\n';
```
我在Init函数中打印了当前第4个列值表达式的col_idx，如是为3，则是指向该列在当前表的位置。但结果如下图：col_idx为0，指向了该列在原表中的位置
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407221205470.png)
显然，对于非窗口函数列，我们需要通过column_获取其列值表达式，并执行Evaluate，通过原schema与原tuple获取应当存储在非窗口函数列的Value
根据该函数插入非法value
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407221224905.png)

对于窗口函数列，我们需要先进行agg，也就是将tuple插入到`WindowFunctionHashTable`中
由于`WindowFunctionHashTable`的key为group-by key，我们可以遍历窗口函数列，根据该列的WindowFunction.partition_by_.Evaluate()获取group-by key，然后构造AggregateKey，用这个key去hash table中查询AggregateValue
- 窗口函数的聚合对象只有一个，所以AggregetaValue只有一个Value
- 而在普通聚合函数中，聚合对象可以有多个，所以AggregateValue可以有多个Value
以上，我们获取了AggregateValue，将其唯一的Value填入之前的empty_value处即可

SimpleAggregationHashTable的agg_expres参数没用，我删掉了
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407221548370.png)
之前写成了`columns[func_idx]`，这是当前Schema的列值表达式，具体是窗口函数的表达式，也不是列值表达式。应该获取agg表达式，这才是一个列值表达式

特判Rank，一开始为0，仅有当前key等于上一key时，same+1，continue跳过最后的same=0。

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407222015881.png)

只遍历了一个窗口列，剩下4个没有遍历
马勒戈壁，之前改代码时，括号改错了，本来构造结果在agg外面，改到里面去了，淦