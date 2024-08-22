```toc
```
## Task1：PageGuard实现
```cpp
class BufferPoolManager;
class ReadPageGuard;
class WritePageGuard;

class BasicPageGuard {
 public:
  BasicPageGuard() = default;
  BasicPageGuard(BufferPoolManager *bpm, Page *page) : bpm_(bpm), page_(page) {}
  BasicPageGuard(const BasicPageGuard &) = delete;
  auto operator=(const BasicPageGuard &) -> BasicPageGuard & = delete;
  BasicPageGuard(BasicPageGuard &&that) noexcept;
  void Drop();
  auto operator=(BasicPageGuard &&that) noexcept -> BasicPageGuard &;
  ~BasicPageGuard();
  auto UpgradeRead() -> ReadPageGuard;
  auto UpgradeWrite() -> WritePageGuard;
  auto PageId() -> page_id_t { return page_->GetPageId(); }
  auto GetData() -> const char * { return page_->GetData(); }
  template <class T>
  auto As() -> const T * {
    return reinterpret_cast<const T *>(GetData());
  }
  auto GetDataMut() -> char * {
    is_dirty_ = true;
    return page_->GetData();
  }
  template <class T>
  auto AsMut() -> T * {
    return reinterpret_cast<T *>(GetDataMut());
  }
 private:
  friend class ReadPageGuard;
  friend class WritePageGuard;

  [[maybe_unused]] BufferPoolManager *bpm_{nullptr};
  Page *page_{nullptr};
  bool is_dirty_{false};
};

class ReadPageGuard {
 public:
  ReadPageGuard() = default;
  ReadPageGuard(BufferPoolManager *bpm, Page *page) : guard_(bpm, page) {}
  ReadPageGuard(const ReadPageGuard &) = delete;
  auto operator=(const ReadPageGuard &) -> ReadPageGuard & = delete;
  ReadPageGuard(ReadPageGuard &&that) noexcept;
  auto operator=(ReadPageGuard &&that) noexcept -> ReadPageGuard &;
  void Drop();
  ~ReadPageGuard();
  auto PageId() -> page_id_t { return guard_.PageId(); }
  auto GetData() -> const char * { return guard_.GetData(); }
  template <class T>
  auto As() -> const T * {
    return guard_.As<T>();
  }
 private:
  BasicPageGuard guard_;
};

class WritePageGuard {
 public:
  WritePageGuard() = default;
  WritePageGuard(BufferPoolManager *bpm, Page *page) : guard_(bpm, page) {}
  WritePageGuard(const WritePageGuard &) = delete;
  auto operator=(const WritePageGuard &) -> WritePageGuard & = delete;
  WritePageGuard(WritePageGuard &&that) noexcept;
  auto operator=(WritePageGuard &&that) noexcept -> WritePageGuard &;
  void Drop();
  ~WritePageGuard();
  auto PageId() -> page_id_t { return guard_.PageId(); }
  auto GetData() -> const char * { return guard_.GetData(); }
  template <class T>
  auto As() -> const T * {
    return guard_.As<T>();
  }
  auto GetDataMut() -> char * { return guard_.GetDataMut(); }
  template <class T>
  auto AsMut() -> T * {
    return guard_.AsMut<T>();
  }
 private:
  BasicPageGuard guard_;
};
```
BasicPageGuard
- 删除了拷贝构造和拷贝赋值函数，所以这是一个禁止拷贝的类
- 提供了默认构造，接收Page\*和BufferPoolManager\*的构造函数
- 提供了移动构造和移动赋值函数

Drop：
- 手动unpin/unlatch的API
- 调用bpm的UnpinPage即可

BasicPageGuard与ReadPageGuard、WritePageGuard之间是has-a的关系，这里没有使用继承
BasicPageGuard提供了基础的RAII机制，protect BufferPoolManager与Page，防止线程未调用BufferPoolManager的UnpinPage后退出，使得缓存池资源泄漏
特殊的是，BasciPageGuard的构造函数没有调用PinPage，实际上PinPage这个工作由BufferPoolManager的NewPage和FerchPage完成。所以BasicPageGuard只需要在构造函数UnpinPage即可，除此之外，BPG也提供手动释放资源（UnpinPage）的方式，调用Drop即可
大部分RAII的实现，都会禁用Guard类的拷贝功能（拷贝构造与拷贝赋值），BasicPageGuard也不例外，但是BPG保留了移动功能，使用时需要特别小心，一定不能使用被移动的BPG

UpgradeRead/UpgradeWrite：将BasicPageGuard对象升级为ReadPageGuard/WritePageGuard
- 这两个类只有BasicPageGuard一个成员
- 升级：为Page上读/写锁
- UpgradeRead/UpgradeWrite提供了构造函数，可以修改内部BPG，调用该构造函数，将当前BPG的资源转移给UpgradeRead/UpgradeWrite对象的BPG即可

接着根据以上类实现BufferPoolManager的以下成员函数
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407081913236.png)
三个Fetch函数的参数都是page_id，首先需要根据page_id获取Page \*（在page_table_中查找frame_id，根据pages_ + frame_id得到Page \*）
根据this以及Page \*构造BasicPageGuard对象
对于FetchPageRead/FetchPageWrite，调用BPG对象的UpgradeRead/UpgradeWrite即可

NewPageGuarded：获取新的页
- 与NewPage功能相同
- 只不过NewPage返回Page \*，而NewPageGuarded应该返回BasicPageGuard对象
- 所以复用NewPage接口即可

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407101626600.png)
operator bool应该添加显式标记，防止无意的隐式转换
如：
```cpp
Resource res;

if (res) {
    // 资源有效
}

// 无意中使用了隐式转换，编译器会将res转换为布尔值
bool isValid = res;
```

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407111102834.png)
为什么会有脏页？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407111114377.png)
在NewPage后，将Page强转成header page，就设置了脏位，因为header page一定会修改page的数据
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407111113488.png)

operator bool的实现逻辑存在问题：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121241144.png)
之前写成了this->bmp_ == nullptr，存在两个问题
一个是应该对page_进行判断，因为Guard对象是page，这个bool表示的是page是否有效
二是需要判断指针**不为空**，不为空才是有效page
找出原因：错误信息为内存访问错误（空指针），因为调用了GetData()引起的空指针错误，所以自然地想到获取的page无效，因为代码只增加了Page Guard，所以检查buffer_pool_manager.cpp，其中对于无效page将进行异常抛出，但是错误信息却没有提到异常。反复检查后，发现operator bool实现错误

测试用例来源：[CMU15-445 2023 Spring ＜Project 1 Task 3＞ “Read/WritePageGuards“ 死锁测试样例-CSDN博客](https://blog.csdn.net/NekoTom/article/details/131737806)（感谢这位热心网友的分享）
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121307769.png)
放弃Guard对象的所有权时，只调用BasicPageGuard的Drop，并不会释放写锁，这将导致死锁，应该调用this->Drop()
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121308472.png)
先释放自己的写锁资源（不再写入数据，此时可以释放），再设置脏位，并调用gurad的Drop。此时才正确的释放了写锁并刷脏

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121605159.png)
WritePageGuard的Drop函数中，如果page为空，则不应该抛出异常，因为可能有的Guard被移动，page已经为空
（终于过了T1！）
修改：由于可能Fetch无效page id，所以返回nullptr page即可，无需抛异常，调用层进行合法性检查即可
## Task2：Hash Table Page
以下是需要实现的哈希表示意图：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407081936430.png)

该结构被称为extendible hash table，显然这种哈希有三种表结构，表名如下图所示
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407082002006.png)
### Header Page
每个Hash Table仅有一张Header Page
用directory_page_ids_数组存储Directory Page的ids，directory page有key，这个key需要进行hash，hash得到的值作为directory_page_ids_的下标，用来存储page id
虽然directory_page_ids_存在max_size_，用max_depth_表示数组长度
```cpp
static constexpr uint64_t HTABLE_HEADER_PAGE_METADATA_SIZE = sizeof(uint32_t);
static constexpr uint64_t HTABLE_HEADER_MAX_DEPTH = 9;
static constexpr uint64_t HTABLE_HEADER_ARRAY_SIZE = 1 << HTABLE_HEADER_MAX_DEPTH;
class ExtendibleHTableHeaderPage {
 public:
  ExtendibleHTableHeaderPage() = delete;
  DISALLOW_COPY_AND_MOVE(ExtendibleHTableHeaderPage);
  void Init(uint32_t max_depth = HTABLE_HEADER_MAX_DEPTH);
  auto HashToDirectoryIndex(uint32_t hash) const -> uint32_t;
  auto GetDirectoryPageId(uint32_t directory_idx) const -> uint32_t;
  void SetDirectoryPageId(uint32_t directory_idx, page_id_t directory_page_id);
  auto MaxSize() const -> uint32_t;
  void PrintHeader() const;
 private:
  page_id_t directory_page_ids_[HTABLE_HEADER_ARRAY_SIZE];
  uint32_t max_depth_;
};
static_assert(sizeof(page_id_t) == 4);
static_assert(sizeof(ExtendibleHTableHeaderPage) ==
              sizeof(page_id_t) * HTABLE_HEADER_ARRAY_SIZE + HTABLE_HEADER_PAGE_METADATA_SIZE);
static_assert(sizeof(ExtendibleHTableHeaderPage) <= BUSTUB_PAGE_SIZE);
```

删除了构造函数，再调用宏：禁止了拷贝与移动功能
同时暴露出Init接口，用来初始化HeaderPage
header page指向多个directory page，这些page的id使用directory_page_ids数组存储
需要实现hash函数：
get/set方法：根据directory_page_ids_的下标get/set对应的page id
Hash方法：取对应数据的最高depth位
### Directory Page
```cpp
static constexpr uint64_t HTABLE_DIRECTORY_PAGE_METADATA_SIZE = sizeof(uint32_t) * 2;
static constexpr uint64_t HTABLE_DIRECTORY_MAX_DEPTH = 9;
static constexpr uint64_t HTABLE_DIRECTORY_ARRAY_SIZE = 1 << HTABLE_DIRECTORY_MAX_DEPTH;
class ExtendibleHTableDirectoryPage {
 public:
  ExtendibleHTableDirectoryPage() = delete;
  DISALLOW_COPY_AND_MOVE(ExtendibleHTableDirectoryPage);
  void Init(uint32_t max_depth = HTABLE_DIRECTORY_MAX_DEPTH);
  auto HashToBucketIndex(uint32_t hash) const -> uint32_t;
  auto GetBucketPageId(uint32_t bucket_idx) const -> page_id_t;
  void SetBucketPageId(uint32_t bucket_idx, page_id_t bucket_page_id);
  auto GetSplitImageIndex(uint32_t bucket_idx) const -> uint32_t;
  auto CanShrink() -> bool;
  auto GetLocalDepth(uint32_t bucket_idx) const -> uint32_t;
  void SetLocalDepth(uint32_t bucket_idx, uint8_t local_depth);

  void IncrLocalDepth(uint32_t bucket_idx);
  void DecrLocalDepth(uint32_t bucket_idx);
  auto Size() const -> uint32_t;
  auto MaxSize() const -> uint32_t;
  auto GetGlobalDepth() const -> uint32_t;
  auto GetMaxDepth() const -> uint32_t;
  void IncrGlobalDepth();
  void DecrGlobalDepth();
  auto GetGlobalDepthMask() const -> uint32_t;
  auto GetLocalDepthMask(uint32_t bucket_idx) const -> uint32_t;
  void VerifyIntegrity() const;
  void PrintDirectory() const;
 private:
  uint32_t max_depth_;
  uint32_t global_depth_;
  uint8_t local_depths_[HTABLE_DIRECTORY_ARRAY_SIZE];
  page_id_t bucket_page_ids_[HTABLE_DIRECTORY_ARRAY_SIZE];
};
static_assert(sizeof(page_id_t) == 4);
static_assert(sizeof(ExtendibleHTableDirectoryPage) == HTABLE_DIRECTORY_PAGE_METADATA_SIZE + HTABLE_DIRECTORY_ARRAY_SIZE + sizeof(page_id_t) * HTABLE_DIRECTORY_ARRAY_SIZE);
static_assert(sizeof(ExtendibleHTableDirectoryPage) <= BUSTUB_PAGE_SIZE);
```
每个BucketPage有depth，存储在数组local_depths_中
而DirectoryPage自己也有depth，存储在变量global_depth_中
1. 每个bucket page有$2^{GD-LD}$个指针指向
2. 在bucket_page_ids数组中，如果有多个指针指向相同bucket page，那么在local_depths_数组中，对应位置的元素应该相同。也就是bucket page的depth应该同意
3. LD<=GD

而max_depth_表示global_depth_的最大值
bucket_page_ids_的最大长度为512，每个元素大小为4，所以数组大小2048
加上local_depths_的大小：512，一共是2560，再加上两个变量，结构体大小为2568

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121446047.png)
设置局部深度时，如果局部深度超过了全局深度，应该不允许设置，我直接抛出异常，但是却无法通过测试用例。测试用例在global_depth_为0的情况下，设置了0位置的local_depth_为1
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121450653.png)
但是之后就增加了global_depth_，故猜测这是测试用例的问题
### Bucket Page
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407091933024.png)
size，capacity，array，分别表示大小，容量，实际存储数据的数组，该数组大小为4088（bucket page的元数据大小为8）
文档里明确说明了，数组存储的是key-value

```cpp
static constexpr uint64_t HTABLE_BUCKET_PAGE_METADATA_SIZE = sizeof(uint32_t) * 2;
constexpr auto HTableBucketArraySize(uint64_t mapping_type_size) -> uint64_t {
  return (BUSTUB_PAGE_SIZE - HTABLE_BUCKET_PAGE_METADATA_SIZE) / mapping_type_size;
};
template <typename KeyType, typename ValueType, typename KeyComparator>
class ExtendibleHTableBucketPage {
 public:
  ExtendibleHTableBucketPage() = delete;
  DISALLOW_COPY_AND_MOVE(ExtendibleHTableBucketPage);
  void Init(uint32_t max_size = HTableBucketArraySize(sizeof(MappingType)));
  auto Lookup(const KeyType &key, ValueType &value, const KeyComparator &cmp) const -> bool;
  auto Insert(const KeyType &key, const ValueType &value, const KeyComparator &cmp) -> bool;
  auto Remove(const KeyType &key, const KeyComparator &cmp) -> bool;
  void RemoveAt(uint32_t bucket_idx);
  auto KeyAt(uint32_t bucket_idx) const -> KeyType;
  auto ValueAt(uint32_t bucket_idx) const -> ValueType;
  auto EntryAt(uint32_t bucket_idx) const -> const std::pair<KeyType, ValueType> &;
  auto Size() const -> uint32_t;
  auto IsFull() const -> bool;
  auto IsEmpty() const -> bool;
  void PrintBucket() const;
 private:
  uint32_t size_;
  uint32_t max_size_;
  MappingType array_[HTableBucketArraySize(sizeof(MappingType))];
};
```

可以看到：实际数组的大小为(page size - bucket page元数据size) / 元素大小
MappingType就是一个`pair<KeyType, ValueType>`

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407091947482.png)
除了key，value的类型，还需要一个比较器，用来对key进行比较
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407091948197.png)

Lookup，根据cmp对key进行比较，如果cmp(key1, key2)为true，说明找到了相同的key，此时返回value的引用，所以这个结构即有查找也有修改功能
Remove，根据key删除key-value，RemoveAt，根据数组下标删除key-value
Insert，在数组末尾插入key-value，在此之前需要判断是否存在相同key
KeyAt，ValueAt，EntryAt，根据数组下标返回相应元素

验证三个条件成立：遍历bucket_page_ids_，得到每个entry指向的bucket page id，同时遍历local_depths_，可能存在多个entry指向同一个bucket page id的情况，此时这些相同的bucket page应该相同的local_depth
创建map表，存储bucket id与其local depth，如果在map表中存在bucket id，那么其存储的local depth应该和当前bucket的local depth相同
需要验证local depth小于global depth
遍历bucket_page_ids时，相同bucket_page对应的local depth应该相同（local_depths_的成立）
记录每个bucket page分别有几个指针指向。数量应该等于2的global depth - local depth次方

directory的hash函数，需要根据LSB，hash原数据，也就是取原数据的低位
而header的hash函数，需要根据MSB，hash原数据，也就是取原数据的高位

只要涉及到“参数是数组下标”就要检查是否越界
运算时，也要注意0的特判。比如global_depth_为0
## Task3
```cpp
template <typename K, typename V, typename KC>
class DiskExtendibleHashTable {
 public:
  explicit DiskExtendibleHashTable(const std::string &name, BufferPoolManager *bpm, const KC &cmp, const HashFunction<K> &hash_fn, uint32_t header_max_depth = HTABLE_HEADER_MAX_DEPTH, uint32_t directory_max_depth = HTABLE_DIRECTORY_MAX_DEPTH, uint32_t bucket_max_size = HTableBucketArraySize(sizeof(std::pair<K, V>)));
  auto Insert(const K &key, const V &value, Transaction *transaction = nullptr) -> bool;
  auto Remove(const K &key, Transaction *transaction = nullptr) -> bool;
  auto GetValue(const K &key, std::vector<V> *result, Transaction *transaction = nullptr) const -> bool;
  void VerifyIntegrity() const;
  auto GetHeaderPageId() const -> page_id_t;
  void PrintHT() const;
 private:
  auto Hash(K key) const -> uint32_t;
  auto InsertToNewDirectory(ExtendibleHTableHeaderPage *header, uint32_t directory_idx, uint32_t hash, const K &key, const V &value) -> bool;
  auto InsertToNewBucket(ExtendibleHTableDirectoryPage *directory, uint32_t bucket_idx, const K &key, const V &value) -> bool;
  void UpdateDirectoryMapping(ExtendibleHTableDirectoryPage *directory, uint32_t new_bucket_idx, page_id_t new_bucket_page_id, uint32_t new_local_depth, uint32_t local_depth_mask);
  void MigrateEntries(ExtendibleHTableBucketPage<K, V, KC> *old_bucket, ExtendibleHTableBucketPage<K, V, KC> *new_bucket, uint32_t new_bucket_idx, uint32_t local_depth_mask);

  std::string index_name_;
  BufferPoolManager *bpm_;
  KC cmp_;
  HashFunction<K> hash_fn_;
  uint32_t header_max_depth_;
  uint32_t directory_max_depth_;
  uint32_t bucket_max_size_;
  page_id_t header_page_id_;
};
```

index_name_：索引名（构建索引后将创建一个hash结构）
bpm_：获取page必须经过缓存池
cmp_：用来进行key之间的比较
hash_fn_：

helper：VerifyIntegrity、GetHeaderPageId、PrintHT、Hash
需要实现的：Insert插入，InsertToNewDirectory、InsertToNewBucket
Remove删除
MigrateEntries元素移动
UpdateDirectoryMapping更新目录
GetValue获取值（查找）

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121525881.png)
将key-value插入到新的bucket，提供directory page指针，桶号，key-value

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121525983.png)
将key-value插入到新的directory，提供header page指针，目录号，hash？key-value

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121814253.png)
获取key的hash值

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407081936430.png)

首先存储的数据类型为K，K可能不是整数类型，无法调用header，directory，bucket中的hash函数，所以需要调用DiskExtendibleHashTable中的Hash，得到一个uint32_t整数

构造函数：

由于Insert会用到GetValue，所以先实现GetValue
GetValue流程([cmu15445 2023fall project2 详细过程（下）Extendible Hash Table-CSDN博客](https://blog.csdn.net/qq_40878302/article/details/137634323?spm=1001.2014.3001.5502))：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121549374.png)

阅读测试用例后得知：获取page后，可以将其强转成T2实现的header page类型，再进行使用
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121608617.png)

以下是BasicPageGuard的As和AsMut函数，ReadPageGuard和WritePageGuard的As和AsMut函数对其进行了复用（ReadPageGuard只有As函数）。这两个函数会对page的data指针进行强转
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121611941.png)
注意As和AsMut的区别：是否设置脏位

Insert：
要注意，大部分情况下，我们只需要修改bucket_page（上写锁），而不需要修改header_page和directory_page（只需要上读锁）

首先调用GetValue，判断是否存在key
给header_page加读锁，hash directory_idx，获取directory_id，判断directory_page是否存在，若不存在则调用InsertToNewDirectory
若存在则解读锁，给directory_page加读锁，在directory_page上hash bucket_idx，获取bucket_id，判断bucket_page是否存在，若不存在则调用InsertToNewBucket
若存在则解读锁，给buket_page加写锁，写入对应值
这里要注意bucket_page是否为满，如果满了，需要分裂bucket_page：调用IncrGlobalDepth，并创建新的bucket_page，对于被分裂的bucket_page需要重新hash数据。如果分裂后还是无法存储数据，则需要重新分裂（重新hash完再调用Inser即可）
[cmu15445 2023fall project2 详细过程（下）Extendible Hash Table-CSDN博客](https://blog.csdn.net/qq_40878302/article/details/137634323?spm=1001.2014.3001.5502)：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407121851277.png)

分裂桶的实现：bucket有RemoveAt操作？这个操作将导致数组不再连续，size_也不再指向数组的下一个可用位置
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407122112021.png)

文字描述下获取page，并转换成header_page，directory_page，bucket_page的过程
首先DiskExtendibleHashTable想要获取页，就必须经过buffer pool，所以有一个指针成员bpm_
- 调用bpm_的FetchPage系列函数，获取page资源并使用guard来unpin page
- 调用guard的As，AsMut接口，将page资源的data指针转换成相应类型的指针，这样就能将page资源视为某种结构使用
- 假设page资源被转换成了ExtendibleHTableHeaderPage，第一件需要做的事就是Init，初始化page资源，因为不知道获取的资源是否有脏数据
- 之后就能通过指针调用相关接口

用代码表示：
```cpp
ReadPageGuard guard = this->bpm_->FetchPageRead(page_id);
/** 检查guard是否有效 */
auto page_ptr = guard.As<ExtendibleHTableHeaderPage>();
/** 将page资源转换成指针后就能调用相关接口了 */
```

现在在写桶的分裂过程：
首先调用IsFull()判断桶是否为满，满了就获取directory_page的write_guard，上写锁
增加相应local_depth_，判断global_depth_和local_depth_是否相等，如果相等，也需要自增global_depth_
增加相应local_depth_后，获取分裂桶的桶下标，需要获取新页作为其桶号
设置完新桶的桶号后，给新页上写锁，准备rehash
rehash：
遍历旧桶，获取每个位置的key和value，对key进行rehash得到下标，如果桶下标为新桶，则删除（RemoveAt）该位置的key-value，将key-value插入到新桶。如果桶下标既不是旧桶也不是新桶，那么抛出异常，出现了意外的数据

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407130852238.png)
分裂完成后，应该继续调用自己（Insert），如果IsFull总是成立，那么就会一直调用自己。直到桶满，Insert返回false，这里要注意桶满并不意味着页满（即hash table满），只是无法继续插入相同hash值的数据

如果不需要分裂，则调用旧桶的Insert接口即可

大多数情况下，底层调用都不应该抛异常？直接LOG_ERROR
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407130934634.png)
GetSplitImageIndex：这个函数应该根据local_depth_获取分裂桶号
而IncrGlobalDepth也将导致桶的分裂，此时需要根据global_depth_获取分裂桶号，或者`this->Size() / 2`也可以

比较在意的是：插入新directory和新bucket时，directory_page的两个depth需要如何维护？
一开始的depth为0，此时可以认为不对key进行hash映射，因为掩码为0，返回的hash也为0，无论任何数都会被hash为0
直到bucket满，此时将分裂一个新bucket，对bucket中的数据进行rehash，根据最后一位bit进行rehash
所以插入新directory和bucket时，不需要在意/维护depth

可以预见的是：多个bucket_page_idx指向相同的bucket_page_id，当这个bucket_page满，分裂后，这些bucket_page_idx就需要重新映射。重新映射则需要根据分裂后的local_depth

分裂桶的过程：
先增加local_depth，如果local_depth大于global_depth，那么也需要增加global_depth
（调用IncrGlobalDepth将自动维护directory的local_depths_与bucket_page_ids_数组）
但是IncrLocalDepth需要调用UpdateDirectoryMapping，以维护相同数组
比如原来的bucket_idx为00，depth为2，有8个指针指向，分别是
00000，00100，01000，01100，10000，10100，11000，11100
现在bucket_idx的depth增加到3，分裂为000与100，那么00000，01000，10000，11000的指向不变，而00100，01100，10100，11100将指向分裂桶
从100开始，不断地增加1000，这个值就是1 << (local_depth)，而且是增加后的local_depth
所以原本的一个参数是没用的（local_depth_mask）

Remove：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407131553256.png)
问题是删除后的桶合并：

接着判断global_depth是否能减小
不断重复以上过程

如何合并？在directory_page的bucket_ids数组中，找存储了empty bucket id的entry，修改entry为non empty bucket
那么如何找这样的entry？根据empty bucket idx，不断加上1 << local_depth获取idx，直到idx超出数组范围。这里的local_depth是减少之前的
不断累加获取idx，在bucket_ids数组中修改entry，再修改local_depths，将depths减一
再根据non empty bucket idx，不断累加1 << local_depth获取idx，修改local_depths，将depths减一

问题是：如果depth为0，获取的split_bucket_id是什么？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407131816119.png)
由于0是一个特殊情况，这里需要特判，直接返回非法id即可，所以depth为0时，不会进行合并

测试Insert和GetValue时，报错：发生了死锁
测试用例Insert 8个k-v到哈希表中，Insert完成再调用GetValue相应key，应该返回true
在Insert第3个k-v时发生了死锁
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407131916894.png)
GetValue函数使用了三把读锁，分别读取了header_page，directory_page，bucket_page
猜测是之前Insert未释放锁
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407132003522.png)
递归调用自己，结果发生了死锁
在调用自己之前，释放资源即可

所以要注意，使用锁时，尽量不要调用自己
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407132032500.png)
应该在return语句中调用自己，否则递归完成后，会继续执行后续代码，这是不可预测的行为

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407132035076.png)
在底层函数中添加ERROR信息，虽然上层运行正常，但是底层打印出了异常信息
上层调用做了合法性检查，理论上这些信息不会出现，但是却出现了，结合上下文的DEBUG信息就能快速定位到bug

Remove的Drop和Delete还未完成，先阅读Remove函数中锁的使用情况，再根据锁资源判断是否能直接Drop
依次获取了header_page，directory_page，bucket_page的写锁，删除了bucket_page中的k-v
接着要进行合并操作，合并操作中，获取了split_bucket_page的读锁（只是读取split_bucket_page是否为满的信息），之后的修改针对于directory_page，所以获取读锁就够

合并之后，directory_page的Shrink操作，不会访问split_bucket_page的资源，所以在Shrink之前就释放资源是可行的

第一个测试用例，为什么最多只能存储8个k-v？directory_depth为2，大小为4，bucket大小为2
为什么第三个测试用例，插入5个k-v后，local_depth能达到3？因为bucket大小为2，插入5个k-v导致3次分裂是可能的
阅读每个函数的锁资源使用情况，尽可能及时地释放锁资源

在Remove函数中，是否需要获取header_page的写锁？首先header_page没有删除接口，也就是header_page总是存储着directory_page，而且header_page没有size_，只有max_size
那么directory_page是否会出现为空的情况？directory_page用global_depth表示数组大小，也存在Size()接口，而directory_page的Size接口返回的是1 << global_depth，所以directory_page一定不为空，它至少有一个entry，所以header_page在删除操作中一定不会被修改，所以Remove只要获取header_page的读锁即可
只有在Insert函数中，可能涉及对header_page的修改操作，所以只有Insert函数会加写锁

需要注意：Remove只有合并操作会修改directory的数据，所以只需要给directory加读锁，需要合并时再加写锁？这样是否会存在问题？
首先是：如果线程获取了读锁，其他线程也能读，此时将阻塞其他线程的写，如果释放了读锁，其他线程获取写锁，自己再获取写锁。由于其他线程获取了写锁可能修改了数据导致之前读到的数据不准确，而获取的写锁又是基于之前读到的数据进行修改，所以这里存在数据不一致性

**如果接下来的操作可能涉及到资源的修改，那么直接获取写锁，不要先读再写，这可能导致数据不一致**
在Insert函数中，似乎就是先获取header_page的读锁，再获取写锁，这里需要判断是否存在数据不一致的问题
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407140900776.png)
获取了directory_page_id后，试着上锁，接着发现page_id无法Fetch，这就表示相应的directory_page不存在，需要先NewPage，再修改header_page的directory_page_ids
所以这里应该给header_page上写锁，不应该上读锁，可能存在资源泄漏的问题（释放读锁后，其他线程先获取写锁，NewPage后，当前线程再获取写锁，NewPage后，覆盖之前线程的page，程序将无法再次使用被覆盖的page，这是资源泄漏）
所以正确做法是Insert函数应该获取header_page的写锁，发现directory_page有效时，再释放写锁。若directory_page无效，那么将header_page的指针传给InsertToNewBucket，难怪官方给的函数参数是指针

似乎是扩缩容依然存在问题，这两个操作分别存在于Insert和Remove函数中，先阅读下自己的实现，找找问题吧
首先InsertToNewDirectory和InsertToNewBucket不会导致扩容，只向新桶插入一条k-v，最多只会导致bucket满
直觉是分裂后递归调用自己的资源没有释放干净
插入bucket后，可能导致bucket的分裂，此时需要维护directory，所以依然需要持有directory的写锁
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407141023363.png)
扩容需要根据split_idx，修改directory的local_depths和bucket_ids数组，我好像只修改了split_idx相关的entry，原idx相关的entry的local_depths也需要增加

Remove释放资源时有些啰嗦
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407141230205.png)
如果不满足条件，应该跳出死循环，不进行桶的shrink，而不是直接返回
直接返回将跳过之后directory的shrink

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407141313199.png)
合并条件：如果curr_bucket或split_bucket为空且local_depth相同，不为0，则进行合并

合并其实是修改directory的local_depths_和bucket_ids_数组的过程，当然也需要释放empty_bucket所使用的page资源
值得注意的是：合并过程中bucket_idx不会发生变化，但entry存储的bucket_id则会发生变化，也就是对应的page资源会发生变化，所以我们需要不断地根据bucket_id获取page资源，得到bucket_page后，再进行合并条件的判断


![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407141840209.png)
插入0~999后，删除0~499：存在数据丢失的情况
分裂桶时，想着使用原地算法迁移数据：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407141834547.png)
bucket的size在不断减小，因为调用了Remove，反而将bucket的size作为循环结束条件