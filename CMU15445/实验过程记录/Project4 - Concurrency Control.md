```toc
```
## Task1 - Timestamps
每个事务有两个时间戳，读时间戳和commit时间戳
begin事务，分配read ts。这将是上一个事务的commit ts。read ts表示该事务能够读取小于等于该时间戳的数据，这意味着当前事务能够上一个事务commit的所有数据
commit事务，分配commit ts，这是一个单调递增的序列，用来表示事务的串行顺序
commit ts是一个逻辑计数器，每当事务递交，事务将被赋予commit ts，同时commit ts将会自增
txn_map_：事务状态表

Watermark用来追踪所有的read ts，watermark_用来表示所有正在运行事务中，最小的read ts
我们需要实现$O(logN)$的算法，找出最小read ts
事务提交后，应该从Watermark中移除read ts（RemoveTxn）
事务开启后，应该向Watermark中添加read ts（AddTxn）
`std::unordered_map<timestamp_t, int> current_reads_`：为什么value是int，为什么要有value？
不同事务的read ts可能相同，value表示该read ts的数量
可以用map维护最小read ts，当数量为0，或者从0到1时，需要维护map
commit_ts_表示当前调度中，最后一个提交的事务commit ts，在AddTxn时，应该保证time_ts大于等于commit ts，这是一个逻辑判断
所以事务commit后，需要UpdateCommitTs

## Task2 - Storage Format and Sequential Scan
### T2.1
bustub在三个地方存储数据：page heap，txn manager，事务内部
page heap存储了最终的数据，txn manager存储了PageVersionInfo，其指向了数据的下一个“修改”，也就是undo log
我们需要不断回溯执行undo log，来获取历史数据
这和课上讲的delta增量类似，只是我们没有将undo log存储在磁盘，而是存储在事务内部
这使得数据库更容易实现，且效率更高，但是容灾能力差
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407242001142.png)

我们需要实现`ReconstructTuple`，定义在 `execution_common.cpp`中，我们也可以将常用的函数定义在这个文件
undo log以时间降序的方式存储在系统中
不管版本的时间戳，我们直接根据函数参数应用所有的undo log
那么看下如何应用undo log，和官方描述的一样：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407250825685.png)

undo log格式比较特殊
- `vector<bool>`数组，长度和tuple attrs的数量相同
- 如果bool为true，表示该字段被修改了
- tuple，表示修改的attrs，attrs的数量和true的数量相同
- is deleted，该log是否有效，猜测这是用来回滚的？
- read ts
- 一个指向下一个log的link
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407242001310.png)

那么如何重建schema？先看schema的成员
length_：字节数
columns_：列属性，保存Column
tuple_is_inlined_：是否内联
uninlined_columns_：内联列的属性
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407250829450.png)

构造函数将获取每个列的长度，并维护每个列的偏移字段，最终维护schema长度
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407250832972.png)
只有这一个构造函数，这意味着我们需要根据modified_fields获取原schema的Column
用下面这个函数：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407250835423.png)
这样就能构造modified_schema，通过modified_schema与undo log的tuple获取修改值，并应用在res_tuple上
这里需要看tuple的构造函数：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407250844868.png)
不管它的实现细节，总之我们需要一个Value数组，我们可以先resize该数组，根据undo log的修改内容，直接通过下标修改
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407250855211.png)
Value的默认构造函数，将调用一个有参构造，最终size_.len_会被设置为`BUSTUB_VALUE_NULL`
这样就能通过IsNull判空
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407250856354.png)

存在两个问题：如果undo log被删除，我们要直接返回吗
是否要对undo log的link进行合法性判断？
1. 我们不需要对undo log进行合法性判断，这将由调用者保证。但是做一个判断总归是好习惯
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407250906400.png)
2. 我们需要根据is_delete标志，处理tuple被删除的情况
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407250912098.png)
那么回滚日志将是什么情况？是否要考虑级联回滚？
要continue还是直接ruturn？个人理解是：应该要continue，级联回滚的问题也不应该在这里考虑，上层回滚了就需要将比较版本的is_delete标记，我们只需要关心is_delete标记，不应该应用这样的修改，应该继续往后遍历
### T2.2
重写SqlScan算子，通过比较自己的read ts与tuple metadata的ts，会遇到三种情况
1. table heap中的tuple已经是最新的（时间戳相等），此时直接返回
2. table heap中的tuple（1）被另一个未提交的事务updata，或者（2）tuple的rs大于事务的read ts（这是一个未来的数据），此时需要继续往前回溯
3. table heap中的tuple被未提交的事务updata，但该事务就是自己，此时直接返回即可
那么bustub如何表示：该tuple被一个未提交的事务update？
ts是一个64位的数，而txn id也是一个64位的数
如果tuple的ts最高位为1，说明该tuple被一个未提交的事务update
`1 << 62`是一个非常巨大的数，事务id和ts不会增长到这么大。bustub将`1 << 62`设置为`TXN_START_ID`，用`TXN_START_ID + txn id`作为ts，表示该tuple被某个未提交的事务update，将ts的最高位非符号位翻转就能得到事务id

这里的三种情况是所有的情况吗？单纯ts的比较可以分为两种情况：
`<=`：可以读取该tuple，因为txn可以读取ts小于等于read ts的版本，因为这些版本已经被提交
`>`：可能是未提交的版本，对应事务的read ts可能等于自己，也可能大于自己，我们需要判断背后的事务是不是自己？如果是自己，因为是我提交的版本，所有我能肯定读到，直接返回。如果不是自己，那么我不应该看到这样的版本，应该继续往前找未被提交或已被提交且ts <= read ts的版本
所以官方给出的三种情况很好地覆盖了ts之间比较
而且比较巧妙的是，未被提交的版本其ts为:`TXN_START_ID + txn_id`，因为`TXN_START_ID`是一个很大的数，所以ts一定大于事务的read ts，这就能被归到未来事务的情况上
此时根据`TXN_START_ID`，继续判断该版本是否未提交，还是来自未来

通过txn manager的`GetUndoLink`获取指向undo log的undo link
在具体实现中，还需要考虑null与非整数类型的attrbute

梳理下txn_mgr中有关版本的成员
version_info_根据页号保存了所有page的版本信息PageVersionInfo
PageVersionInfo中，prev_version_成员根据slot便宜保存了page中所有tuple的版本信息：VersionUndoLink
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407260852344.png)

VersionUndoLink有个prev_成员，其类型为UndoLink，指向了版本链的链头（第一个版本）
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407250957327.png)
那么这里就有一个问题，为什么prev_version_的value类型不是UndoLink，而是VersionUndoLink？这个后面应该就知道了

传入RID，通过txn_mgr的GetUndoLink获取RID对应的版本链（第一个版本）
GetUndoLink将通过RID调用GetVersionLink获取对链头的封装：`VersionUndoLink`，接着返回其保存的链头：`prev_`
关于GetVersionLink：该函数将通过version_info_获取PageVersionInfo，进而获取VersionUndoLink
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407260911943.png)

我们需要基于T2.1实现的`ReconstructTuple`，重写SqlScan算子
这里就需要仔细看参数
schema：需要输出tuple的列结构
base_tuple：链头版本的tuple？（如果被删除了呢？）
base_meta：tuple的meta数据
undo_logs：这个需要注意，因为`ReconstructTuple`将直接回溯undo_logs的所有版本。所以我们必须小心地构造undo_logs
根据文档的意思：SqlScan不应该关注tuple是否被删除，或者被回滚，这将由`ReconstructTuple`关注
只需要关注版本时间戳（因为文档列举了时间戳比较的三种情况）
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407251003813.png)

那么我们只需要通过`GetUndoLink(RID rid)`函数，通过rid获取tuple的版本链，遍历版本链，比较时间戳，在适当的位置停下调用ReconstructTuple即可
如何判断ReconstructTuple返回的optional是否有效，有效则输出

然后就是需要注意null和non-lined类型
Transaction的成员：
```cpp
  std::atomic<TransactionState> state_{TransactionState::RUNNING};
  std::atomic<timestamp_t> read_ts_{0};
  std::atomic<timestamp_t> commit_ts_{INVALID_TS};
  std::mutex latch_;
  std::vector<UndoLog> undo_logs_;
  std::unordered_map<table_oid_t, std::unordered_set<RID>> write_set_;
  std::unordered_map<table_oid_t, std::vector<AbstractExpressionRef>> scan_predicates_;
  const IsolationLevel isolation_level_;
  const std::thread::id thread_id_;
  const txn_id_t txn_id_;
```

UndoLink通过prev_log_idx_连接了UndoLog，prev_log_idx_是Transaction的undo_logs_数组下标，所以每个事务维护了自己的undo logs，每个tuple的undo logs由事务的undo_logs_成员连接起来
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407260958854.png)

UndoLink保存了前一个事务的id，与undo log在其undo logs中的下标，通过这两个字段获取undo_log
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407251058846.png)

官方给的图很好：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407251042352.png)

但是_是什么意思？
我们需要获取事务的read ts，将其与tuple的ts进行比较
tuple的ts存储在tuple meta里
GetTuple会返回TupleMeta与Tuple
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407251105938.png)


但是现在的问题是：如何获取下一个UndoLink？
通过UndoLog的prev_version_成员
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407261002681.png)

还没有考虑null与non-lined情况
存在这样的测试用例：都是被删除的tuple，调用ReconstructTuple时，一个返回nullopt，一个返回有效tuple

所有版本被回滚，或者tuple被删除且没有历史版本，将返回空
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407261029696.png)

1. undo log为空，而tuple未被删除，返回有效值
2. undo log为空，且tuple被删除，返回null
3. undo log不为空，且tuple被删除，返回有效值
4. undo log不为空，而tuple未被删除，返回有效值
5. undo log不为空，但是全是被回滚的版本，而tuple未被删除

所以最后判断条件是：看最后一条undo log是否被回滚，但存在undo logs为空的情况，此时需要根据tuplemeta的is_delete判断

SeqScan需要考虑filter以及non-lined的问题
filter：ReconstructTuple后，在返回tuple之间，判断是否满足filter即可
先看下Next的逻辑

上层不需要判断tuplemeta的is_delete，因为is_delete可能被未来事务删除，当前事务依然能读取这样的tuple
continue会将循环条件的更新跳过，导致死循环
filter添加了，现在只需要考虑non-lined的问题
底层的GetValue已经考虑的non-lined的问题，所以我们应该不用考虑啊，直接测试
外层循环不断遍历table heap中的所有tuple，内层循环遍历其版本链，维护undo_logs，内存循环的最后需要用undo_logs调用ReconstructTuple，如果得到的值为有效值，则直接返回，否则外层循环继续

SqlSacn返回了空表
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407271111540.png)
gdb调试时观察到，每个版本的ts都不变，最后发现是每个版本的ts都等于tuplemeta的ts
这里应该等于undo_log的ts
（为什么tuplemeta也有一个ts？）
可以理解为该ts表示的是最新版本的ts，版本链中的版本都是历史版本，
版本链第一个版本才是最新版本？之前理解错了
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407271110116.png)

若当前事务能够直接读取当前版本，且该版本未被回滚，那么直接返回
错。我们应该读取事务的最新版本，如果无论该版本是否被删除，上层只需要找最新版本就好了，下次就需要考虑最后一个版本是否被删除了
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407271344322.png)

先实现TxnMgrDebug函数，该函数将打印tuple的版本链，参考格式如下
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407271211527.png)

undo_log的is_delete是回滚标志吗？？？
总之，正确的逻辑应该是：判断base_tuple的ts与read ts之间的关系
如果read ts >= tuple_ts，或者tuple_ts == txn_id，直接返回base_tuple
否则需要回溯版本链，回溯到第一个<=read ts的版本

最后还是忘记返回rid了
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407281933401.png)

这个等值判断似乎有问题，怎么能用base_tuple判断呢
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407281956367.png)

先更新了循环条件，但是后续的结果依然使用了该循环条件
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407282000225.png)

## Task3 - MVCC Executors

### 3.1 Insert Executor
需要使用UpdateUndoLink与UpdateLinkVersion这两个函数
先阅读下：
UpdateUndoLink主要是定义函数对象wrapper_func，再调用UpdateLinkVersion
该包装函数根据传入的link参数是否有效，调用check函数对象
link的参数类型为VersionUndoLink，其prev_成员的类型为UndoLink
- 如果有效，则以link的prev_为参数，调用check
- 无效，则以std::nullopt为参数，调用check
所以是调用check检查UndoLink类型的数据
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407271640621.png)
以上check是用户传入的，UpdateUndoLink对其做了一个封装，使其成为UpdateVersionLink的check函数
如果传入的prev_link参数无效，那么将删除该VersionLink，这似乎是用来回滚insert操作的
因为VersinoLink只有一个UndoLink。但是insert之后，不应该没有UndoLink吗？
所以这里应该是插入一个空的UndoLink，看看有没有这样的接口：默认参数就是无效的，所以直接构造就行了
用一个无效的UndoLink作为链表的结束

TupleHeap访问需要加锁

和之前的insert算子逻辑一样，不同的是我们需要维护txn的write_set，添加tuple的rid

### 3.2 Commit
先获取锁，获取commit ts，不要增加last_commit_ts，应该得到commit操作完成再增加
遍历write_set，更新tuple的ts（这里有个问题，insert后的tuple是否总是base_tuple，还是有可能成为历史版本？）
设置事务的commit状态，更新commit ts
最后再更新txn_mgr的commit ts

之前没有+1
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407282033743.png)

这里怎么就全都置为false了？如果tuple被删除了，应该为true
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407282032245.png)

### 3.3 Update and Delete Executor
如果一条tuple正在被未提交的事务修改，其他事务不能修改它。否则将发生写写冲突
如果事务删除tuple时，发现该tuple已经被read ts更大的事务删除（已经提交），那么当前事务应该被abort：试图删除被未来删除的数据
此时需要将后一个事务abort，发生冲突的事务状态应该被设置为TAINTED

检查冲突未发生后，应该生成undo log
- 对于update，只生成增量，对于delete，生成完整tuple（因为用应用过去的版本需要完整的tuple）
- 如果undo log已经存在，update/delete时应该直接update table heap与undo log（undo log中的数据不应该删除，就算多次修改使其变回了原数据/或者接近元数据，此时不要删除undo log中的Value）
- 更新VersionLink，使其指向新的undo log
- 在table heap中update tuple以及tuple meta

回顾下昨天整理的update和delete算子
首先，update和delete算子中，存在事务之间的冲突，对于tuple的ts，我们使用事务id填入，这是一个很大的数，如果tuple的ts很大，我们则认为该tuple正在被一个事务修改
那么其他事务不能修改这样的tuple，由于我们不使用悲观锁，所以只要冲突发生，我们就需要终止其中一个事务，终止哪个事务呢？引起冲突的那个事务将被终止，也就是后一个事务
而写写冲突还存在一个特例，如果当前事务想要删除一个已经被未来删除的数据，那么当前事务将被abort。如果我想删除被未来修改过的tuple呢？可以删除/修改吗？似乎这不是一种写写冲突的情况
MVCC中，写者相互阻塞，如果事务A修改了tuple，未提交，事务B也想修改同一tuple，必须要等事务A提交。似乎是将修改后的tuple保存在本地？
在具体实现中，也就是保存undo log在本地
但是首先要检测写写冲突。不管这么多，写完undo log的生成先
`writing to a tuple larger than read ts`，前面举的例子是删除，但是最后却说writing？难道是所有的修改操作都会触发写写冲突吗？

update算子是pipeline breaker，应该通过子执行器获取所有需要update的rid，再获取原tuple，根据执行计划的target_expressions_，Evaluate每个列值，与原tuple的每个列值比较
保存原tuple中与新tuple不相同的列值，维护undo log的bool数组，以及其中的tuple，这里再保存不同的Value，再创建一个新的tuple，也可以直接用不同的Value构建tuple，只是需要同时维护bool数组
暂时不维护索引了，有些麻烦
除了构建undo log，还需要更新base_tuple，考虑自我修改，写写冲突
不考虑自我修改以及写写冲突，先更新base_tuple

Update函数都没有实现check
现在的问题是：自我修改的逻辑需要分开写吗？
首先自我修改需要Update base tuple，更新txn的undo logs，write set不用更新，也不需要追加版本链，只需要update head即可。综上，逻辑差得挺多，需要分开写

似乎构建undo log在多个地方用到，这里封装成函数
将自我修改时的first_link设置为second link，接着需要维护VersionLink了
构建undo link也是相同逻辑，也封装成函数吧

UpdateUndoLink将调用UpdateVersionLink，将rid的VersionUndoLink修改为“用UndoLink构建VersionUndoLink”

现在梳理下：自我更新与首次更新的逻辑，前提是没有发生写写冲突
首次更新：根据子执行器获取需要更新的rid，根据rid获取old_tuple
执行target_experssions_，获取new_tuple的每个列值，并构建new_tuple
同时判断new_tuple的列值与old_tuple列值不同时，保存old_tuple的列值与Column，以此构建undo_log的tuple
接着维护txn的undo_logs与write_set，最后更新base_tuple及其meta，并维护版本链
自我更新：同样的，需要根据子执行器获取需要更新的rid，根据rid获取old_tuple
获取rid的版本链，修改其undo log？通过undo link获取undo log的引用？GetUndoLog其实是根据UndoLink指向txn的undo_logs，获取其undo_log，所以我们只需要更新txn的undo_logs即可，那么如何更新？还不是要创建新的undo_log？？？
计算new_tuple的每个列值，并构建new_tuple，同时构建undo log
根据undo log构建undo link
这里有个问题，txn_mgr的updateundolink接口，到底在更新什么？其实它调用了UpdateVersionLink，该函数会将rid的VersionLink更新为新的undo link，此时我们需要自己维护undo link的指向，其指向的undo log需要保存原来的undo link（就是一个链表的头插）

在update的init函数，需要保存所有的old_tuple，因为官方规定了update算子进行任何写操作之前，应该获取所有的tuple
自我修改应该在检测写写之前完成，而且只能添加undo log中的内容
那要怎么做？undo log的tuple需要根据schema生成，这里需要做到子原来的基础上添加列值？那么schema应该如何维护？
首先构建undo log需要根据old tuple，将new tuple与其做比较，如果列值不同再维护schema与vals
这里我应该使用`std::unordered_map<size_t, Column`与`std::unordered_map<size_t, Value`维护schema和tuple吗？这样做的话，最后还需要将其重新转换为`std::vector`，而unordered_map是无序的，所以这里还应该使用map？
还是说，我重新使用一个std::vector，当new_tuple和old_tuple的列值不同，或者原undo log已经对该列值做了修改（通过bool数组判断），再来维护这两个vector？但是Column很好维护，Value应该怎么从undo log的tuple中提取？
一个简单的办法是：在new_tuple和old_tuple的列值比较时，当原undo log存在修改时，先维护两个vector数组，如果new_tuple没有修改，那么原来的修改将被保存，如果有修改，那么原来的修改将被覆盖，这样就能保证在原undo log的基础上应用新的修改
所以我可以改写ConstructUndoLog函数，传入原undo log的optional，与link的optional
若为nullopt则不需要应用原来的修改（这是第一次修改），还需要将undo log的prev_设置为原undo log的prev。否则将prev_设置为传入的link参数

undo log和数量和自我修改之间是否存在关系？
一般来说，undo link指向的log都是有效的，只有在自我修改时，可能存在无log的情况（自己先插入再update，插入不会增加undo log），还有就是其他事务插入后，进行update，如果其他事务未提交，将会发生写写冲突。提交了就正常写入，所以我们只需要在自我修改时，判断log是否为空以更新undolog即可

基本上写完了delete的逻辑，完成测试完写GC吧
undo log的ts应该是base tuple的ts
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407282101211.png)


update时，应该先保存rid，而不是tuple和meta，因为后续需要用rid获取UndoLink

update的自我修改存在问题
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407282135199.png)
首先就是自我修改时，版本的ts不应该是事务id，而应该是原来的ts

两个算子自我修改时，更新undo log存在问题：
本来是需要重建tuple，应用第一个undo log，再结合第一个undo log生成new undo log
但是我没有重建tuple，直接将修改后的tuple与本次的修改后的new tuple进行比较，并结合第一个undo log，生成new undo log
首先要确定的是：这样做是否可行？我写的ConstructUndoLog先应用了原undo log的修改，再应用新的修改，所以能保证update一定是正确的
那么delete呢？我应该先判断该tuple是否已经被delete，如果已经被删除，我就不应该再删除，如果没有被删除，则以整个base tuple作为undo log即可
不对，已经被删除将触发写写冲突，此时不应该直接返回
这里应该触发写写冲突，但是我写了提前返回，触发了莫名的错误
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407291007914.png)

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407291009695.png)

不对，ConstructUndoLog有问题：我应该先计算新的增量，再应用旧的增量，因为旧的增量将使得该tuple回到base tuple。所以新旧增量冲突时，我们应该应用旧增量，保存新增量无法回到undo log

问题：自我修改后，bast tuple的ts更新错误，应该更新为事务id
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407290908845.png)

那么delete的自我更新，什么时候tuple具有版本链？显然不是insert，只有可能是delete或update。二次delete已经提前返回，所以只可能是update。显然，我们只需要设置is_delete
### Task 3.4 Stop-the-world Garbage Collection
当所有的事务停止时，将触发GC
这么看来，undo log中的is_delete是用来标记垃圾回收的
我们需要检查正在运行的事务，通过这些事务的read_ts（watermark表示这些事务的最小read ts），找到一个<= read ts的版本，将之后的版本删除is_delete

之后遍历txn_mgr的next_gc_txn_id，如果该事务的所有undo log可以被删除，我们就移除该事务，并增加next_gc_txn_id
现在的问题是：如何移除Transaction，只有txn_mgr的txn_map存储了事务吗
似乎是这样的，我们只需要将txn从txn_map中移除就行

不能删除正在运行的事务
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407291222249.png)

通过can_delete判断undo log是否能删除，如果base tuple能够被读取，那么can_delete应该为true，而不是一开始为false
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407291238191.png)

因为GC导致了悬空指针，而TxnMgrDbg又没有对undo link的合法性进行判断，所以抛出了异常
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407291253634.png)

同样的GC也存在悬空指针的问题，其实应该修改GetUndoLog函数。似乎返回值不是optional，那么只能在GetUndoLog前，判断该事务是否存在了

我们只能清理commited的事务，tainted都无法清理
## Task4 - Primary Key Index
### 4.1 - Inserts
insert时需要维护索引，这里的索引为主键索引，需要检查唯一性冲突。这就需要insert算子在插入前检查是否存在相同主键的情况，当然这是在插入具有主键索引的表时需要做的检查
tainted状态：事务被abort，但其对table heap做了修改（数据没有被清理干净）。我们不需要实现abort逻辑，我们也不需要清理tainted事务留在table heap中的数据
我们需要先设置tainted状态（`txn->SetTainted()`）再抛出ExecutionException，这一点Task3已经知道了
1. 检查
2. 在table heap中插入新的数据，使用事务id作为ts
3. 维护索引，从一开始的检查到现在的插入索引这段时间中，可能其他事务也插入相同tuple并维护索引，此时当前事务应该被abort。在table heap中留下一个没有任何指针指向的tuple

不要运行`make check-lint`！！！

之前写的hash索引，insert函数在调用完bucket page的insert后，直接返回true
而bucket page的insert也返回了bool值，当bucket满，或者存在相同key时，返回false
所以Insert应该返回bucket page的insert函数
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407291816306.png)
但是依然存在一个问题，我们假设调用底层函数之前，已经保证了没有相同key，那么在调用bucket page的insert时，就不应该存在相同key
但是由于并发问题，在检查不存在相同key之后到真正插入的这段时间中，可能发生冲突。因为检查只是获取了header page的读锁，可能两个线程都通过检查，再插入相同的值。在释放读锁，获取写锁的这段时间中两者竞争header page的写锁，但是最终双方都会向bucket page写入这个相同值，导致数据冲突。由于这个情况只有在线程竞争激烈时才会发生，为了应对这种情况，我们在bucket insert函数中，也加入对相同key的判断
### 4.2 Index Scan, Deletes and Updates
我们需要先实现index scan执行器，在此基础上为update和delete提供索引支持
文档说：index的entry应该始终指向相同rid，即使相应的tuple被删除。这将使得更早的事务能够访问该tuple的历史版本，所以，一旦具有相同key的tuple被创建，我们应该update index的entry，而不应该create新的entry

以下是官方给出的相应例子：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407291954474.png)
可以看到，似乎我们可以在被删除的base tuple上，insert tuple？
而之前实现的insert算子，都是直接插入一个空的版本链，也就是默认了insert之前，表中没有相同key的tuple，就算该tuple被删除，也不用维护该tuple的版本链
所以还需要考虑的是：在被delete的tuple上insert

此外，我们还需要实现写写冲突以及唯一性的检测

#### Index Scan
似乎找第一个可读undo log的过程和SqlScan相同，这里将这个过程封装为函数吧
封装了FindFirstUndoLog，但还没有修改index scan
IndexScan写完了，梳理下流程：
Init：获取表的索引结构，根据执行计划的常量表达式获取where子句的常量，用该常量构建key tuple。调用索引的Scan结构，将查找的结果保存
Next：遍历init保存的结果（rids），调用FindFirstUndoLog函数，该函数将在rid对应的tuple的版本链中，找到当前事务第一个可读的版本，并重建tuple
如果重建成功，就返回true，输出rid与tuple

现在需要考虑并发问题：似乎scan不需要考虑并发，update，delete再考虑吧
因为这三个算子未添加索引支持，所以似乎无法测试IndexScan
T4.1已经实现了Insert的索引支持，所以要实现的是update和delete的索引支持

VersionUndoLink，不仅是对UndoLink的封装，还有一个`in_progress`字段，表示是否有事务正在修改tuple的版本链。可以预感到，这个字段是用来做check检测写写冲突的

#### delete
为delete添加索引支持：需要注意，底层索引将存储tuple rid，这是一种非聚簇索引
当索引实际执行的tuple被删除时，我们也无需修改该索引的entry，使其依然指向被删除的rid，这将使得正在运行的事务能够通过索引读取其旧版本，不会因为未来的修改受到影响，这本质和SeqScan相同，SeqScan因为会遍历table heap中的所有tuple，这其中也包括了被删除的tuple，所以正在运行的事务不会因为未来的修改，而无法读取tuple。index scan也需要做到这一点，要怎么做？delete tuple时，依然指向被删除的tuple
而由于index的entry存储的是rid，我们只需要更新rid指向的tuple即可
综上，delete操作无需维护索引？可以在GC中添加对index entry的清理
#### update
update似乎也不需要维护index？如果更新了tuple的主键列呢？是否要删除索引中原entry，使新entry存储rid？

#### insert
由于delete操作不会删除所有的entry，所以在insert相同key的tuple时，我们需要通过索引获取已经存在的tuple（IndexScan），并检测该tuple是否被删除，如果被删除，我们更新该tuple即可，版本链为nullopt（删除之前的版本链）
此时就不需要维护索引了，因为entry存储了tuple rid，我们没有新增rid，而是在原有的基础上进行更新

check函数将传递当前VersionLink，调用方也需要传入一个VersionLink，该VersionLink是在有锁的情况下，获取到的。我们需要检查这两个VersionLink是否相等
这里调用`==`的重载即可
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407301443216.png)

这里有一个问题，TupleMeta的ts为事务id，表示该TupleMeta正在被该事务修改。由于写写冲突，我们不能修改这样的tuple。所以在update/delete之前，我们需要检查TupleMeta的ts
但是由于TOCTTOU问题，从检查到真正插入的这段时间内，数据可能被修改，导致之前的检查无效。此时进行插入操作，将导致资源泄漏与无法预期的错误
在这段时间中，要么上latch保护资源，防止事务冲突。要么依赖底层模块的逻辑与返回信息，在插入后再次检查
比如T4.1的insert算子中，我依赖了底层extendible_hash的insert函数，该函数线程安全，且插入相同key时将返回false。根据其逻辑与返回值，我再次检查：将insert失败的事务SetTainted
但是这将使得两个模块之间高度耦合，在T4.2后，官方介绍了VersionLink的in_progress_字段，该字段表示是否有事务正在操作当前tuple
而TupleMeta的ts将具体表明哪个事务正在操作当前tuple，这么看来，in_progress_的功能是ts的子集。可以说in_progress_是一个多余的设计吗？
我们需要在UpdateVersionLink时，传入check函数，让UpdateVersionLink在执行真正的更新前回调check，以检查将被更新的VersinUndoLink和一开始被检查的VersionUndoLink是否相同。这是一个CAS操作，操作系统提供了CAS指令，能够用一条语句完成寄存器上的比较，设置的操作，保证操作原子性
但是我们需要CAS的对象显然更复杂，如何保证操作原子性？只能够上latch，阅读UpdateVersionLink就会发现，比较与更新操作将在持有page latch时进行
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407301550274.png)
因为比较（检查），更新的操作具有原子性，所以我们就能通过UpdateVersionLink更新VersionUndoLink的in_progress_字段。将其设置为true后，就得到了tuple lock

总之，UpdateVersionLink将通过page latch获取tuple lock。有了tuple lock后，我们就能放心地对tuple进行修改操作

那么tuple update的逻辑为：
```
·获取tuple的VersionLink
  ·如果有事务在操作该tuple，且该事务是自己，进行自我修改（已经持有tuple lock）
  ·如果有事务在操作该tuple，且该事务不是自己，发生了写写冲突，SetTainted
  ·如果没有事务在操作该tuple，检查该tuple是否被未来事务修改过
    ·如果被未来修改过，说明发生了写写冲突，SetTainted
    ·如果没有被未来事务修改过，调用UpdateVersionLink获取tuple lock
      ·成功则进行正常修改的逻辑
      ·失败则SetTainted
```

还要注意的是：in_progress_是一个lock，什么时候释放该lock呢？和TupleMate的ts相同，在commit时释放

最后，引入主键索引后的insert的逻辑为：
```
·检查主键的唯一性冲突
  ·如果发生冲突，判断是否有事务正在操作该tuple
	·有事务在操作，判断该事务是否是自己
	  ·该事务是自己，进行自我修改
	  ·该事务不是自己，SetTainted
    ·没有事务在操作，判断tuple是否被彻底删除
      ·被彻底删除，记录其rid
        ·获取该rid的VersionLink，并调用UpdateVersionLink获取tuple lock
          ·成功则进行update逻辑
          ·失败则SetTainted
      ·没有被彻底删除，SetTainted
  ·如果没有发生冲突，执行正常的插入逻辑，最后调用UpdateVersionLink获取tuple lock
```
bug：在正常插入的执行逻辑中，存在同时插入多个具有相同key的tuple情况
此时获取的tuple lock不会阻止其他具有相同key的tuple的插入，我们需要根据索引是否插入成功，判断是否插入了相同key的tuple，进而Atinted事务

还有一个问题：是先维护索引还是先维护table heap
只有自己删除了当前tuple才能执行insert操作，所以自我修改时第一个undo log的创建者一定是自己，且tuple的is_delete_为true，这说明了该tuple现在被当前事务删除
还要注意的是：正常插入时，UpdateVersionLink的prev参数不能写nullopt，否则check将永远为true，我们需要使VersionLink在插入前后不同，而nullopt不会创建VersionLink

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407301927200.png)

lambda表达式捕获的对象必须显式声明，不能用auto

不知道是插2，3还是5，6抛的异常，明天再debug
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407302120063.png)

插入6抛的异常，正在操作的事务是自己时，需要满足什么条件才能插入？
首先tuple必须被删除，其次，如果tuple有undo log，那么第一个undo log的创建者一定为自己
这是否是一个多余的判断条件？还是要判断的，能看到潜在的bug

什么问题呢？update的自我修改时，存在有效的undo log，但是创建者不是自己
这是因为insert，有些情况只更新update base tuple导致的，什么时候需要更新undo log呢？
没有事务在操作tuple时，如果tuple已经被删除，且不是来自未来的删除，那么该tuple的第一个undo log一定是其他事务的删除undo log，或者没有undo log（同一事务插入再删除，不会产生undo log）
无论是否有undo log，我们都应该根据table heap中的tuple构造undo log
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407310842848.png)

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407310840731.png)

update的自我修改存在问题，引入主键后，我们会在被删除的tuple上再次插入，这会导致有些undo log的is_delete字段为true（之前只会因为GC的清理为true）
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407311020633.png)

update算子中，维护txn的undo log要在UpdateVersionLink之前？
不然先更新了VersionLink，其他事务select，要应用undo log时，发现不存在，淦
这就报了空指针错误
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407311354479.png)

思考一个问题：delete的自我修改，如果存在undo log，那么undo log应该被怎么更新？
首先undo log一定是当前事务update产生的，在update基础上delete，undo log将保存update前的tuple
所以这里不能传入当前tuple（delete_tuple）
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407311645505.png)
最后一个参数截不到，它表示了修改后undo log的tuple。但是我传了当前的base tuple
这是不对的

测试程序是：8个线程，每个线程都会更新0-19的主键
程序将记录每个主键成功更新的次数，以位图的方式表示哪些线程成功更新
而每个线程也是以位图的方式更新主键
程序最后将成功更新的次数位图与表结果匹配，判断两表是否相等

但是出现了表结果小于程序记录的位图结果，某个主键逻辑上要更新，但是实际却没有更新
可以将每一次地成功更新记录，输出主键值与结果值
有一个线程应该要更新tuple，但是由于什么原因没有更新。

其实更新了，但是为什么是往小地去更新？一个线程更新时，持有该rid的行锁，其他线程无法访问该rid
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407311942960.png)
该现象出现的原因：两个线程基于0（原始值）同时更新了tuple，前一个更新被后一个更新覆盖

进一步发现，冲突发生在前两次更新
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407312000403.png)
有没有一种可能，加锁确实是原子的，但是基于更新的tuple已经被更新过
也就是说，在加锁之前读取了tuple，到加锁时，该tuple已经被其他事务修改过，加锁是给更新后的tuple加锁，但是却基于更新前的tuple修改，那么当前事务所作的修改将覆盖其他事务的修改
加锁确实能保证当前tuple只有一个事务在操作，但是操作要保证“基于当前tuple”
就是一个TOCTTOU问题，解决方法是：加锁后再获取tuple，因为加锁后其他事务无法操作tuple，该tuple一定不会被修改

in_progress的原子问题：当前检查in_progress为false后，将其修改为true，并操作tuple
但是可能有多个事务通过了in_progress的检查，将其修改为true，同时修改tuple，此时将发生数据竞争
个人想法是：in_progress的临界问题根本上也是TOCTTOU问题，检查in_progress为true到修改in_progress的这段时间中，in_progress被其他事务修改为了false
我们无法保证修改（使用）in_progress_时，它一定为false，虽然之前做了检查。可能因为系统的调度，其他事务将其改为false。总之检查-使用不是一个原子操作，这将导致基于错误的检查而使用
怎么解决呢？保证检查-使用是原子操作即可。可以使用mutex，但推荐使用更轻量的atomic，因为这只是一个简单的CAS（Compare-and-Swap）操作
```cpp
if (!in_progress_) {    /** 检查 */
  in_progress_ = true;  /** 使用 */
}
```

bustub是怎么保证in_progress_是原子操作的？严格来说，既不是mutex，也不是atimoc
具体的说：in_progress被封装在VersionUndoLink中，在访问VersionUndoLink前必须获取该VersionUndoLink的写锁。有了写锁后，我们才能修改in_progress_，这保证了修改in_progress_的原子性。但是聪明的你发现：当前事务读取in_progress_~获取写锁修改in_progress_这段时间中，其他事务可能已经对in_progress_做了修改，这意味到当前事务不能修改in_progress_。又回到了一开始的TOCTTOU问题，怎么解决呢？只好在获取写锁后再次检查in_progress_为false，检查通过才能修改。这也是为什么要在UpdateVersionLink时传入check函数，check将在获取写锁后被调用以检查in_progress为false（和之前读取时相同）

是的，小心翼翼地实现了这一大段逻辑后，才能保证in_progress_的原子性

有了in_progress_的原子性，事务就能保证修改tuple时的临界性。但如果需要基于tuple进行修改，请再读取一次tuple。因为我们无法保证get tuple lock前读取的tuple不会被其他事务修改
***
依然存在一个问题：验证事务是否正在被修改时，需要使用in_progress，而不是使用txn ts
### 4.3 Primary Key Updates
update主键，需要delete原来的tuple，更新构造新的tuple，将其insert回表中
这里需要用到insert和delete的逻辑

在引入主键索引后，自我修改时，第一个undo log的创建者是否是自己？
可能是一个delete的undo log，当前事务只是insert，insert+update，insert+delete
此时第一个undo log的创建者就不是自己
在delete中，如果第一个undo log的创建者是自己，当前事务update，那么我们需要撤销之前的update，保存原来的tuple作为第一个undo log
但如果第一个undo log的创建者不是自己，就只需要更新base tuple
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408011100794.png)

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408011053134.png)

需要保存更新前以及更新后的key value，在真正的更新前判断两者是否相同
如果相同，就是和以前一样的逻辑
如果不同，delete+insert，由于被删除的tuple可能也会被更新，所以我们需要先记录所有的old tuple以及它们产生的new tuple，以及需要删除的rids
先delete所有的rids再insert所有的new tuple


完结撒花了，接下来干嘛捏？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408011510018.png)
最多一个礼拜时间整理实验过程，分享思路，无聊了就刷刷题

