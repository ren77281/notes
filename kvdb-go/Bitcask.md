```toc
```
## 数据结构介绍
Bitcask实例目录就是一个活跃文件与多个旧的数据文件的集合。
同一时刻只有一个活跃文件用于数据的写入，当文件的大小达到阈值时，将关闭活跃文件，并创建一个新的活跃文件。
注意：不管是文件被写满还是进程退出，它就成为了不活跃文件。
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408090829068.png)

以追加的方式写入文件，数据的格式为：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408090829654.png)

前4个字段长度固定，通过ksz和value_sz可以解释出最后两个字段key和value. 
删除时，不会原地删除，而是追加写入墓碑值，merge时再执行物理删除。

这些文件存储在磁盘中，在内存中我们需要存储目录文件keydir. 
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408090833453.png)
- file_id: 标识数据文件(新旧数据文件)
- value_pos: 整条记录在数据文件中的偏移量

keydir只会保存最新数据的信息，如果你插入了相同key, 将被视为update操作。
get操作如下：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408090837538.png)

为何要重复记录value_sz? 减少一次IO次数。用于数据文件中，记录的前四个字段已知，并且也已知key_sz, 所以能够通过一次寻址找到磁盘中value的起始与结束位置，并将其读取。

merge操作时，将遍历所有旧数据文件，保留有效记录(没有被删除的key)的最新版本到新的数据文件merged data file中，丢弃/删除无效数据。
merge在后台操作，不会影响读写操作。
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408090843509.png)

同时merge将生成hint file. 该文件保存了记录的元信息，用于加快引擎启动时，keydir的加载。

以上，Bitcask将key-value分离，在内存中对key建立索引，在磁盘文件中存储value。
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408090847661.png)

只要数据得落盘，那么**顺序写**的吞吐量是最高的。
关于崩溃恢复，有点不能理解。

value的大小要远大于key的大小，Bitcask才能发挥出最佳性能。

具体实现分为两个部分：一是内存中的数据如何存储，而是磁盘中的数据如何组织。
## 代码实现
### 内存，磁盘设计
对于内存中的数据存储，我们使用btree作为索引结构。
创建index目录，btree.go对btree的操作进行封装，index.go对索引操作进行封装

需要在index.go中，封装Indexer接口，该接口提供Get，Put，Delete的函数签名

结构体声明，字段声明，返回其指针，此时对象将在堆区创建

将接口作为函数参数时，需要传入实现该接口的类型指针，与rust不同，不能传入对象
在btree.go中，封装btree，封装btree的Get，Put，Delete函数。

创建fio目录，封装文件操作
定义IOManager接口，封装Read，Write，Sync，Close方法
### DB读写流程
需要封装两个接口：Get与Put，给用户使用

写流程：先写磁盘，再写内存
这里有一个问题：为什么需要Sync接口？文件的写入估计具有缓冲区，Sync负责刷新缓冲区，真正地写入磁盘？

Put：根据k-v, 构造LogRecord结构体，调用append方法，将该记录追加到磁盘的文件上，最后更新索引
appendLogRecored：将LogRecord追加到磁盘上，并返回记录的位置信息：LogRecordPos
先判断是否存在活跃文件，若不存在则新建
将LogRecord序列化成byts，EncodeLogRecord，该函数返回byte与size。预判此时写入是否达到阈值，若达到，保存当前活跃文件Sync，维护旧数据文件的map，再新建活跃文件
此时，向活跃文件Write数据，根据用户配置决定是否持久化Sync
最后，构造LogRecordPos并返回
### DataFile读写流程
写的话，直接向文件末尾追加bytes即可。在这之前需要调用一个Encode函数，将结构体序列化再写
读的话，keysz与valuesz都是varint编码，变长。所以header的大小可变（只有前两个字段大小固定）
需要调用binary.PutVarint将int64整数以varint编码的方式写入到bytes中


生成随机数源，通过随机数源生成随机数生成器

为数据库封装迭代器：首先对index封装了迭代器，定义了iterator特征。在btree.go中定义BTreeIterator结构体，使其实现iterator特征。同时对Indexer特征添加方法：NewIterator，这样实现了Indexer特征的结构体也需要实现NewIterator方法，进一步地就是需要实现Iterator特征。所以我们就能通过 具有Indexer特征的结构体获取 具有Iterator特征的结构体。

所以我们实现了BTree的迭代器，通过Indexer就能获取。
当然，这个迭代器是针对内存中的索引建立的。我们的最终目标是建立数据库的迭代器，用来遍历k-v。
为什么要实现BTree的迭代器呢？为了在db的迭代器中，封装BTree的迭代器。如果db的迭代器要获取k-v中的value，就需要访问磁盘。因为value存储在磁盘中，无法通过index迭代器访问。

需要实现写操作的原子性，Put接口有锁，已经保证了原子。我们要实现多个写操作的原子，即原子写。
思路是：将事务写入的数据暂存在事务内部，Commit的最后put一条表示批量操作完成的record
实现WriteBatch结构体，描述引擎中的批量写操作
WriteBatch配置：运行一次写入的最大数据量，Commit时是否序列化

Commit的逻辑：之前已经保存了需要写入了Record，所以Commit时只需要调用db.append... 往数据文件中追加记录即可。每次追加记录都会阐述logRecordLogPos，此时需要用Pos更新内存中的index信息
需要判断record是否是删除标记，如果是删除标记，更新index信息时需要调用index.Delete，而不是index.Put
最后，追加一条“完成”的Record即可

当然，Commit时需要获取一个序列号（类似于事务ID），这是一个递增序列。引入了批量写后，磁盘中的record的key似乎为序列号+原来的key

先写序列化与反序列化函数
WriteBatch完成后，引擎中存储key的方式发生了改变：序列号+key。
所以之前涉及到record的地方，都要修改key，这里的序列号为0，说明这不是一个批量写

（构建index时，需要加载data file，调用其ReadLogRecord接口读取record。该接口将返回record的长度与record，我们需要用record的长度维护record在data file中的偏移量，依次构建pos。当引入了writebatch后，我们需要读到类型为finish的record才能该wb的数据加载到index中。否则将无法保证wb的原子性。由于只需要更新/删除index，所以我们只需要暂存相同key，logrecord类型，logrecordpos即可）

对batch进行测试：主要有三个方法Put，Delete，Commit以及序列化与解析函数
先测序列化与解析函数

没有搞清楚index中的key与磁盘中的key不同，磁盘中的key需要有id
梳理一下：只有磁盘中的key为：id+key，而内存中index的key没有id。所以db.Put中key需要有id，而Delete中的key也需要有id。
总之，Delete与Put都需要向磁盘写入record，其中的key需要加上id，因为它们的位置在磁盘中。而Get只需要向内存中的index获取pos，index不需要存储id，这是一个无效操作。在磁盘中存储id只是为了表明批量写操作由谁进行，以及最终是否完成。只有完成之后才会向index中添加索引信息。
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408130921389.png)

写入磁盘的key应该为id+realkey，这里只有realkey没有id

merge思路：检查act文件是否存在，存在才merge
检查是否正在merge，同一时间只能有一个线程在merge，当然，检查前需要给db上锁。这里会稍微影响到正在使用db的线程
将当前活跃文件关闭，保存为不活跃文件。并保存所有的不活跃文件，最后释放db锁。
for循环加载所有数据文件，读取数据文件中的record，将record在index中查找。这里要注意，读取record的时候，还需要维护其off，以便在查找时与index保存的pos做比较。如果比较结果相同，说明这是一条最新记录，不应该被清理，所以我们要重写它。
关于重写，如果直接调用db.Put，又会影响用户的使用，降低用户体验。

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408131512785.png)

解决方案是：再开一个db实例，以将数据文件存储在不同目录。这个db实例和merge前的db实例是两个线程，操作着不同的db实例，彼此之间的数据不会冲突。

综上，遇到最新record后，我们需要将其重写到新db实例中。调用其append接口，同时旧record的key可能含有wb标记，你需要将其清除。因为merge不会重写finish record到新db中，在新db重启时，这些含有旧wb标记的record不会被加载，这将导致数据丢失。
重写record后，你还应该维护hint文件，以便加速db的启动过程。hint文件需要存储key以及其pos，也就是index需要用到的信息（这个后面再细看）。
重写后，新的db以及hint文件都需要持久化：Sync()。
最后，你需要在新db中添加一个finish文件，以表示这次merge的完成。
有点不明白的是：为什么需要往finish文件中，写入数据。通过finish的文件名不就能知道merge成功的信息吗？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408131527629.png)
打开了finish文件，并向其写入了一条k-v。key是"merge.finished"，value则是nonMergeFileId，这个变量表示了未被merge的file id。
哦知道了，所以finish文件中，保存了这次merge的范围，nonMergeFileId及其往上的文件都没有被merge过，这样db在重启时就知道hint文件所表示的范围了。

把文字流程截图钉在桌面上，这是一种提高效率的编程方式。这样的翻译式编程似乎更能让人沉浸

实现了merge流程后，考虑重启。重启时需要依据hint文件构建index，那么如何判断hint文件存在？
你需要通过os.Stat判断hint文件是否存在，如果存在则将其打开（加载data file）。维护offset，读取其中的record，parse value获取其pos信息，以维护index

以及，我们需要获取上一次merge的最大fid，这个可以通过打开finish file完成。其中只有一条record，key为mergeFinishedKey，value为byte格式的maxfid

替换merge目录与原数据目录：获取merge目录的路径getMergePath，判断merge目录是否存在。存在则读取该目录，os.ReadPir。不要忘记defer 删除目录，因为最后一定完成了替换，那么merge目录留着就没有用了。
读取merge目录下的所有文件，若存在finished文件，则标记此次merge的成功，可以进行替换。对于其他文件，则保存它们的文件名。
读取完成后，若merge不成功，则退出merge。获取最大merge的file id，根据file id删除原目录中的数据文件。然后将merge中的数据文件移动到原目录中，替换时使用os.Rename，用之前保存的文件名，将merge目录与文件名拼接，获取完整的文件名。

现在考虑db.Open的修改，首先，如果存在hint文件，应该先读取hint文件，以快速构建index。之前读取datafile文件的目的是：为了在内存中构建index。而现在hint文件已经保存了构建index所需的元信息，就没有必要再去读取datafile，因为读取datafile是一个耗时的操作。所以，我们需要修改loadIndexFromDataFile，先判断是否存在hintfile，如果存在则保存maxMergeFileId

关于merge的测试，现在似乎只能手动merge？或者添加策略：容量每增长多少就merge一次？无所谓了，先测试merge是否能成功吧。

先验证merge后的目录是否存在：写入多条不重复的数据，然后调用merge，此时将生成一个目录
应该是在原db的index中查找
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408150910188.png)
应该在最后更新off，因为中间会使用到off

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408150914545.png)

淦！
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408151008812.png)

如果创建迭代器，将产生冗余数据：迭代器将在冗余数据上备份，尽管art tree可以直接在树上遍历，但是b tree已经这样实现了，还是统一一下吧。
由于B+树索引会将数据持久化到磁盘上，所以在使用B+树索引时不需要加载磁盘的datafile文件，再构建索引。因为索引已经存在于磁盘，我们只需要读取磁盘即可。

因为B+树索引结构的加入，我们还需要保存wb序列号在文件中。为此需要特地创建一个文件以存储wbid，但是这就产生了冗余，因为非B+树不需要这个文件。但是再次启动数据库时，内存索引可能从非B+树变成了B+树，就又需要这个文件了。总之，无论什么样的内存索引，在关闭数据库时，都需要保存这个wbid文件。但是容灾呢？每次wb之后都写一次这个文件？

首先需要添加的就是wbid文件的读取方法，先设定该文件的名字。为其添加打开方法，接着就是其加载方法，首先要明确的是：其中的数据也是k-v的，因此，我们需要特别地设置一个key。之后：拼接文件名，存在则打开，打开后读取record，strconv.ParseUint（由于binary.Vaint过于麻烦，这里只有一个int，直接strconv即可）
然后是Close db时，需要保存wbid file。这里添加逻辑：先关闭index（使B+ tree持久化），打开wbid文件，构造record，将wbid strconv成字符串，作为value写入

最后需要修改的是Open的逻辑：如果当前index为B+ tree，则只需要load wbidFile即可
如果非B+ tree，则是之前写的逻辑。

做了个修改，index不允许插入空的key，因为底层的数据结构对空的key判断不统一（ARTree竟然能插入空key无法Get！）

db是单进程的设计，为了使db只有一个进程在使用，需要使用文件锁。
先新建文件锁flock.New
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408161600498.png)


添加：统计失效数据bytes的功能，当失效数据达到一定阈值，将进行merge。这里需要改写index的Put，Delete接口：返回old record的pos
需要注意的是：delete时，追加的record也需要被统计为无效数据

无效数据量的字段已经添加，还有：无效数据量达到阈值时，自动触发merge
以及Stat的实现：数据库占用磁盘空间的情况。获取数据库所使用的内存，再判断是否能够进行merge

merge时，需要判断无效数据量是否达到阈值，没有达到不merge。
```bash
go test -bench=.
go test -bench=. -benchtime=5s
go test -bench=. -benchtime=1000000x
```

需要完成对redis数据结构的支持，首先redis的数据结构有String，Hash，Set，List，Zset。可以用byte表示这些结构常量。而RedisDataStructure其实就是bitcask的封装，这些数据结构的方法相同，都是：Set，Get，Delete。而Delete是一个通用的方法，我们不需要针对每个结构设置不同的Delete方法。

这些结构的Set方法有相同的参数ttl，也就是生存时间。我们需要根据不同结构的存储格式，对value进行encode，再使用bitcask存储这些value。
String的格式是：type + expire + payload。由于time.Duration是int64的别名，而type是byte，所以buf需要变长64字节+1字节。
有些数据类型比较特殊：如hash，list，与string不同，一个key对应一个value。这些数据类型的一个key对应了多个value，可以这么说。比如hash类型，一个key对应了多个字段与value的映射。也可以理解这里的字段为key，我们需要通过一个key去索引多个key。那么如何表示这个字段在哪个key下？当然是不仅仅存储字段名，还存储key值。
如果key被删除了，我们不能遍历key的所有字段，删除所有字段，只需要给key标记一个唯一值，让字段也具有这个唯一值，移除这个唯一值后，其字段在存储引擎中也是无效的。

TODO：测试！list与redis协议支持
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408202039322.png)
这个`:=`，前面的测试是怎么通过的？
```bash
root@racknerd-73c23b:/home/kv-go/benchmark# go test -bench=. -benchtime=150000x
goos: linux
goarch: 386
pkg: kv-go/benchmark
cpu: Intel(R) Xeon(R) CPU E5-2683 v4 @ 2.10GHz
Benchmark_PutValue_BaskDB         150000             11653 ns/op             708 B/op         10 allocs/op
Benchmark_GetValue_BaskDB         150000              1235 ns/op              68 B/op          4 allocs/op
Benchmark_PutValue_BoltDB         150000             84053 ns/op           15937 B/op        113 allocs/op
Benchmark_GetValue_BoltDB         150000              5008 ns/op             559 B/op         25 allocs/op
Benchmark_PutValue_GoLevelDB      150000             20534 ns/op             548 B/op          9 allocs/op
Benchmark_GetValue_GoLevelDB      150000              3620 ns/op             109 B/op          6 allocs/op
PASS
ok      kv-go/benchmark 23.641s
```

```bash
root@racknerd-73c23b:/home/kv-go/benchmark# go test -bench=. -benchtime=100000x
goos: linux
goarch: 386
pkg: kv-go/benchmark
cpu: Intel(R) Xeon(R) CPU E5-2683 v4 @ 2.10GHz
Benchmark_PutValue_Badger         100000             28551 ns/op            1449 B/op         44 allocs/op
Benchmark_GetValue_Badger         100000              5756 ns/op             476 B/op         12 allocs/op
Benchmark_PutValue_BaskDB         100000             10246 ns/op             700 B/op         10 allocs/op
Benchmark_GetValue_BaskDB         100000              1346 ns/op              68 B/op          4 allocs/op
Benchmark_PutValue_BoltDB         100000             71808 ns/op           15483 B/op        108 allocs/op
Benchmark_GetValue_BoltDB         100000              4118 ns/op             559 B/op         25 allocs/op
Benchmark_PutValue_GoLevelDB      100000             18204 ns/op             553 B/op          9 allocs/op
Benchmark_GetValue_GoLevelDB      100000              2457 ns/op             109 B/op          6 allocs/op
Benchmark_PutValue_RoseDB         100000             12916 ns/op             827 B/op         14 allocs/op
Benchmark_GetValue_RoseDB         100000             10512 ns/op             501 B/op         10 allocs/op
PASS
ok      kv-go/benchmark 29.371s
```

```bash
root@racknerd-73c23b:/home/kv-go/benchmark# go test -bench=. -benchtime=10s
goos: linux
goarch: 386
pkg: kv-go/benchmark
cpu: Intel(R) Xeon(R) CPU E5-2683 v4 @ 2.10GHz
Benchmark_PutValue_BaskDB         322922             42781 ns/op            4606 B/op         10 allocs/op
Benchmark_GetValue_BaskDB        7554565              1540 ns/op              68 B/op          4 allocs/op
Benchmark_PutValue_BoltDB          74385            150521 ns/op           23716 B/op        115 allocs/op
Benchmark_GetValue_BoltDB        2277883              4765 ns/op             559 B/op         25 allocs/op
Benchmark_PutValue_GoLevelDB      176368             72313 ns/op            3693 B/op         10 allocs/op
Benchmark_GetValue_GoLevelDB     4681560              2637 ns/op             108 B/op          6 allocs/op
Benchmark_PutValue_RoseDB         228446             59318 ns/op            3718 B/op         14 allocs/op
Benchmark_GetValue_RoseDB        4527708              2475 ns/op             190 B/op          4 allocs/op
PASS
ok      kv-go/benchmark 205.564s
```

写爆磁盘，ez
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202408211844483.png)

keyLen:         14 B
valLen:          128 B
largeValLen: 1024 * 1024 B
