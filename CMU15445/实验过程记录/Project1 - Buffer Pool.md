```toc
```
## Buffer Pool
在缓存池模块中，我们需要管理数据，具体的说，数据在磁盘和内存之间的流动
作为结果，我们需要实现缓存池管理器Buffer Pool Manager
缓存池以数组的形式实现，数组下标被称为**帧**，元素被称为页，准确来说是页的地址Page\*
读写操作以页为单元，不论是读操作还是写操作都需要经过缓存池。调用缓存器的NewPage方法获取一个页。获取指定页时，根据页ID调用FetchPage方法
## Task1：LRU-K替换策略
以下是需要实现的类：
```cpp
class LRUKNode {
 public:
  LRUKNode(int64_t current_timestamp_, size_t k, frame_id_t fid);
  std::list<int64_t> history_;
  size_t k_;
  int64_t k_distance_;
  frame_id_t fid_;
  bool is_evictable_{false};
};

class LRUKReplacer {
 public:
  explicit LRUKReplacer(size_t num_frames, size_t k);
  DISALLOW_COPY_AND_MOVE(LRUKReplacer);
  ~LRUKReplacer() = default;
  auto Evict(frame_id_t *frame_id_ptr) -> bool;
  void RecordAccess(frame_id_t frame_id, AccessType access_type = AccessType::Unknown);
  void SetEvictable(frame_id_t frame_id, bool set_evictable);
  void Remove(frame_id_t frame_id);
  auto Size() -> size_t;
 private:
  std::unordered_map<frame_id_t, std::shared_ptr<LRUKNode>> node_store_;
  std::set<std::shared_ptr<LRUKNode>, Compare> node_evict_;
  int64_t current_timestamp_{0};
  size_t replacer_size_;
  size_t k_;
  std::mutex latch_;
};
```

通常喜欢将保存相似信息的结构封装为node
LRUKNode保存了帧的相关信息，在缓存池替换策略（replacer）中，这些信息就是：
- history_，帧的访问历史（时间戳）
- fid_，帧号
- is_evictable_，是否可驱逐
- k-distance_
- k_

replacer结构拥有node属性，负责维护node的访问记录，驱逐node
- node_store_：kv结构，帧号与node映射
- node_evict_：set结构，重载了比较规则，k-instace最大的node排在最前
- current_timestamp_：当前时间，每次调用RecordAccess将自增
- replacer_size_：最大帧数量
- k_
- latch_

Evict：根据k-distance驱逐frame
-   `auto Evict(frame_id_t *frame_id_ptr) -> bool`
- 访问次数小于k次，k-distance为inf，此时驱逐最早访问的frame（FIFO）
- 驱逐具有最大k-distance的frame，且只能驱逐具有`evictable`的frame
- 成功驱逐后，减少replacer的size以及移除该frame的访问记录（node_store）

RecordAccess：记录frame的访问
- `void LRUKReplacer::RecordAccess(frame_id_t frame_id)`
- 使用current_timestamp_表示时间戳，每次访问frame都会使其自增
- 若frame_id不在node_store_中，则创建node存储该frame的访问
- 若frame_id在node_store_中，push_front访问记录即可

SetEvictable：设置帧的驱逐许可
- `void LRUKReplacer::SetEvictable(frame_id_t frame_id, bool set_evictable)`
- set_evictable表示帧的驱逐许可，true表示将其设置为可驱逐

Remove：不管k-distance，移除指定帧
- `void LRUKReplacer::Remove(frame_id_t frame_id)`
- 只能移除`evictable`的frame

find判断两元素是否相等时，将这样调用比较器（比较器返回true表示左元素应该排在右元素之前）：
```cpp
!Comp(current_element, target_element) && !Comp(target_element, current_element)
```
大小应该是5，程序运行结果为1：比较器认为fid为2~6的node与fid为1的node相同
调试时发现是用来比较的时间戳相同（比较器也有问题），时间戳以秒为单位，则1~6node的时间戳都是相同的
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406271429002.png)

bug：k-distance为inf时，根据frame的最早访问时间驱逐，而不是最近访问时间。但这和最近最少使用有什么关系？？？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407061929212.png)

以下是官方对LRU-K的描述：k-distance由当前时间戳与倒数第k次时间戳之差表示
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407070932448.png)
这个表述存在歧义：当前时间戳容易被理解为访问frame时的时间
而当前时间戳应指“绝对的时间”，也就是k-distance随着时间而不断增大
由于所有的frame的k-distance都在以同样的速度增大，我们可以忽略这部分增量，以倒数第k次时间戳表示k-distance，此时k-distance越小，表示访问frame的时间越早
所以k-distance越小的frame更可能被淘汰

k-distance示例：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407071026278.png)
[CMU15445 (Spring 2023) #Project1_cmu 15445 spring 2023 project 1-CSDN博客](https://blog.csdn.net/weixin_70354558/article/details/130497608)
怎么发现k-distance的描述歧义的？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407070955323.png)
debug以上示例一晚上，突然想到一个hack数据：以k-distance表示**frame访问**的当前时间与倒数第k次访问时间差值为前提
如果某个frame的k-distance足够小，那么大概率不会驱逐它，如果我们之后永远不会访问该frame，那么该frame就应该被驱逐

## Task2
```cpp
struct DiskRequest {
  bool is_write_;
  char *data_;
  page_id_t page_id_;
  std::promise<bool> callback_;
};
class DiskScheduler {
 public:
  explicit DiskScheduler(DiskManager *disk_manager);
  ~DiskScheduler();
  void Schedule(DiskRequest r);
  void StartWorkerThread();
  using DiskSchedulerPromise = std::promise<bool>;
  auto CreatePromise() -> DiskSchedulerPromise { return {}; };
 private:
  DiskManager *disk_manager_ __attribute__((__unused__));
  Channel<std::optional<DiskRequest>> request_queue_;
  std::optional<std::thread> background_thread_;
};
```
DiskRequest描述磁盘请求
- is_wirte_：读/写
- data_：从磁盘读取数据保存到data_/将data_中的数据写入到磁盘
- page_id_：读取/写入的页号
- callback_：`std::promise<bool>`，表示请求是否被处理

DiskScheduler
- construct：创建后台线程，并且使之执行StartWorkerThread函数
- StartWorkerThread：负责获取share queue中的请求并处理：调用Schedule将请求发送DiskManager。同时设置DiskRequest的callback_为true，使发送请求方得到“请求被处理”的信息。直到DiskScheduler生命周期结束前，该函数不应该返回
- destruct：向share queue中发送std::nullopt，告知调用StartWorkerThread的线程返回
- Schedule(DiskRequest r)：向DiskManager发送读/写请求

promise和future的使用：
用来进行简单的线程间通信，future对象依赖于promise。线程A以promise作为参数向线程B传递，线程B调用set_value修改promise的值，线程A调用相应future的get方法获取promise的值。如果promise还未被设置，将导致线程的阻塞
通常需要使用mutex和condition_variable实现以上场景，但这两个变量是全局的，不利于程序的维护
而promise和future是局部变量，使用比mutex和condition_variable简单且利于维护

```cpp
class DiskManager {
 public:
  explicit DiskManager(const std::string &db_file);
  DiskManager() = default;
  virtual ~DiskManager() = default;
  void ShutDown();
  virtual void WritePage(page_id_t page_id, const char *page_data);
  virtual void ReadPage(page_id_t page_id, char *page_data);
  void WriteLog(char *log_data, int size);
  auto ReadLog(char *log_data, int size, int offset) -> bool;
  auto GetNumFlushes() const -> int;
  auto GetFlushState() const -> bool;
  auto GetNumWrites() const -> int;
  inline void SetFlushLogFuture(std::future<void> *f) { flush_log_f_ = f; }
  inline auto HasFlushLogFuture() -> bool { return flush_log_f_ != nullptr; }
 protected:
  auto GetFileSize(const std::string &file_name) -> int;
  std::fstream log_io_;
  std::string log_name_;
  std::fstream db_io_;
  std::string file_name_;
  int num_flushes_{0};
  int num_writes_{0};
  bool flush_log_{false};
  std::future<void> *flush_log_f_{nullptr};
  std::mutex db_io_latch_;
};
```

语法问题：
```cpp
std::thread t(&DiskScheduler::Schedule, this, std::move(r.value()));
```
`&DiskScheduler::Schedule`不能写成`&this->Schedule`，`this->Schedule`会被解释成函数的调用，应该使用类名::成员函数名表示函数的地址
## Task3
```cpp
class BufferPoolManager {
 public:
  BufferPoolManager(size_t pool_size, DiskManager *disk_manager, size_t replacer_k = LRUK_REPLACER_K, LogManager *log_manager = nullptr);
  ~BufferPoolManager();
  auto NewPage(page_id_t *page_id) -> Page *;
  auto FetchPage(page_id_t page_id, AccessType access_type = AccessType::Unknown) -> Page *;
  auto UnpinPage(page_id_t page_id, bool is_dirty, AccessType access_type = AccessType::Unknown) -> bool;
  auto FlushPage(page_id_t page_id) -> bool;
  auto DeletePage(page_id_t page_id) -> bool;
 private:
  const size_t pool_size_;
  std::atomic<page_id_t> next_page_id_ = 0;
  Page *pages_;
  std::unordered_map<page_id_t, frame_id_t> page_table_;
  std::list<frame_id_t> free_list_;
  std::mutex latch_;
  
  auto AllocatePage() -> page_id_t;
  void DeallocatePage(page_id_t page_id) {}
  std::unique_ptr<LRUKReplacer> replacer_;
  std::unique_ptr<DiskScheduler> disk_scheduler_;
  LogManager *log_manager_;
};
```
- pool_size_：缓存池大小，在构造函数的初始化列表定义。表示页/帧的数量，frame_id的范围在`[0, pool_size_ - 1]`
- next_page_id_：原子变量，表示NewPage获取的页Id
- pages_：pool_size_个Page的集合，在数组中通过frame_id查找对应的page
- page_table_：页表，页号与帧号的映射
- free_list_：空闲帧的集合
- disk_scheduler_：task2实现的磁盘调度器
- replacer：LRU-K替换策略
- log_manager_：日志管理器，暂不关心

NewPage：创建新的页
- `auto NewPage(page_id_t *page_id) -> Page *`
- 从free_list_获取空闲帧，若无空闲帧
- 调用replacer的驱逐方法得到空闲帧，如果所有的帧正在被使用，返回nullptr
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406301258512.png)
- 如果驱逐的页脏了，需要先将其写回磁盘。然后重新初始化Page
- 通过replacer.SetEvicetable(page_id, false) "pin"住frame（`pin_count += 1`），使得replacer不会驱逐该frame
- 调用AllocatePage方法得到新页Id
- 最后调用replacer.RecordAccess方法，记录该frame的访问
总之就是：获取空闲帧以存储Page，若无空闲帧，驱逐具有evictable属性的Page以获得空闲帧

只要页表中存在页的映射记录，那么就能获取该页。页可能在对应的帧上，也可能在磁盘中，需要加载到page_table。这里用-1的帧，表示该页不在内存，需要从磁盘加载。所以调用lruk_replacer驱逐某个页后，将其映射的帧设置为-1

使用Page结构描述页，页存储数据
pages存储Page\*，pages是线性数组，下标被称为帧，数组大小由pool_size表示
page_table用来映射页号到帧号，页号唯一标识了Page，
由于pages是线性数组，所以用free_list表示空闲帧号
缓存池只能存储pool_size个页，远小于磁盘空间。使用LRU-K替换策略，尽可能淘汰无用的页，从磁盘加载新的页
如果页的pin_count不为0，那么对应的帧具有unevictable属性，不能驱逐这样的帧。使用UnpinPage减少pin_count，若pin_count为0，则对应的帧具有evictable属性。即根据pin_count是否为0体现evictable属性，具有evictable属性的帧数量为lruk_replacer的size
NewPage不会增加lruk_replacer的size，UnpinPage会增加lruk_replacer的size

FetchPage：获取指定页
- `auto FetchPage(page_id_t page_id, AccessType access_type = AccessType::Unknown) -> Page *;`
- 先确定页号是否有效且未被删除
- 若frame_id为-1，则从磁盘加载该页数据到bufferpool中
- 此时需要从free_list_中获取空闲帧或使调度器驱逐帧，从而获得可用帧
- 如果所有的帧都不可驱逐，返回false
- 驱逐时，如果被替换的帧存储的是脏页，和NewPage一样，需要进行刷脏
- 最后需要pin住该页
- 若frame_id不为-1，说明该页在内存中，pin住该页再返回

UnpinPage：
- `auto UnpinPage(page_id_t page_id, bool is_dirty, AccessType access_type = AccessType::Unknown) -> bool;`
- 如果page_id不存在，或pin_count_已经为0，返回false
- 减少pin_count_，如果pin_count_为0，则调用replacer_.SetEvictable，将所在帧设置为可驱逐
- 如果在使用页的期间（pin）修改了页数据，则需要设置脏位

DeletePage：删除指定的页
- `auto DeletePage(page_id_t page_id) -> bool`
- 如果该页在磁盘中，直接返回true(?)
- 如果页在缓存池中，但被pin住，直接返回false
- 否则调用replacer_.Remove()，在替换器中删除该页占用的帧
- 向this->free_list_中添加帧号，释放帧资源
- 重置page的元数据，调用DeallocatePage()模拟页在磁盘中的释放(?)

FlushPage：强制刷脏
- `auto FlushPage(page_id_t page_id) -> bool`
- 如果无法找到该页，返回false
- 无论是否为脏页，都构造DiskRequest的写请求，写入数据
- 数据写入完成将脏位设置为false

在NewPage和FetchPage时，pin_count_需要+1
NewPage和FetchPage时，需要记录新页ID
没有理解缓存池中的页也能从磁盘加载，比如：直接删除被驱逐的页号，FetchPage也能从磁盘加载页，UnpinPage需要注意page是否在磁盘

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407071615423.png)
