现代C++中，可以对类的成员函数单独使用模板，而不需要对整个类声明模板
```cpp
class TrieNode {
 public:
  TrieNode() = default;
  explicit TrieNode(std::map<char, std::shared_ptr<const TrieNode>> children) : children_(std::move(children)) {}
  virtual ~TrieNode() = default;
  virtual auto Clone() const -> std::unique_ptr<TrieNode> { return std::make_unique<TrieNode>(children_); }
  std::map<char, std::shared_ptr<const TrieNode>> children_;
  bool is_value_node_{false};
};
// 两个成员函数：连接子节点的children_哈希表，表示当前节点是否存储value的bool值
// 构造函数实现了基于shared_ptr的浅拷贝：复制children_哈希表
// Clone()：复用拷贝构造，对当前TrieNode进行浅拷贝，用unique_ptr管理TrieNode对象

template <class T>
class TrieNodeWithValue : public TrieNode {
 public:
  explicit TrieNodeWithValue(std::shared_ptr<T> value) : value_(std::move(value)) { this->is_value_node_ = true; }
  TrieNodeWithValue(std::map<char, std::shared_ptr<const TrieNode>> children, std::shared_ptr<T> value)
      : TrieNode(std::move(children)), value_(std::move(value)) {
    this->is_value_node_ = true;
  }
  auto Clone() const -> std::unique_ptr<TrieNode> override {
    return std::make_unique<TrieNodeWithValue<T>>(children_, value_);
  }
  std::shared_ptr<T> value_;
};

class Trie {
 private:
  std::shared_ptr<const TrieNode> root_{nullptr};
  explicit Trie(std::shared_ptr<const TrieNode> root) : root_(std::move(root)) {}
 public:
  Trie() = default;
  template <class T>
  auto Get(std::string_view key) const -> const T *;
  template <class T>
  auto Put(std::string_view key, T value) const -> Trie;
  auto Remove(std::string_view key) const -> Trie;
  auto GetRoot() const -> std::shared_ptr<const TrieNode> { return root_; }
};

```
auto接收引用类型时，必须使用&
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406091102317.png)
因为cur指向const TrieNode，所有返回的迭代器具有const属性
`dynamic_cast`不是函数，不需要使用`std::`！！！，改了半天
`dynamic_pointer_cast`是函数，在`<memory>`文件中，用来进行智能指针的动态类型转换，通常用于将父类指针转换成子类指针
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406101437642.png)
T类型可能禁用了拷贝构造函数，所以我们需要使用move调用其移动构造
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406101438206.png)
shared_ptr的拷贝构造函数的参数为unique_ptr的右值引用，所以能用unique_ptr构造shared_ptr
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406110723929.png)
如果unique_ptr作为函数返回值，那么shared_ptr可以直接接收
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406110930609.png)

```cpp
template <class T>
class ValueGuard {
 public:
  ValueGuard(Trie root, const T &value) : root_(std::move(root)), value_(value) {}
  auto operator*() const -> const T & { return value_; }

 private:
  Trie root_;
  const T &value_;
};

class TrieStore {
 public:
  template <class T>
  auto Get(std::string_view key) -> std::optional<ValueGuard<T>>;

  template <class T>
  void Put(std::string_view key, T value);
  void Remove(std::string_view key);

 private:
  std::mutex root_lock_;
  std::mutex write_lock_;
  Trie root_;
};
```
std::optional的使用
std::lock_guard

如果key已经存在，并且TrieNode有value，将TrieNode转换成`TrieNodeWithValue<T>`时可能失败，以为value类型可能不相同

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406201114137.png)
线程并发时出现释放空指针错误
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406201115969.png)
一个线程T6调用ValueGuard的析构函数，释放Trie树使用的内存，而这块内存已经被释放
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406201116846.png)
这块内存由另一个线程T1调用Put函数时获得
显然这块内存被T1释放又被T6释放
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406201119863.png)
而Put函数只有在最后才会释放原Trie树资源
最后发现Get使用引用获取Trie树资源，而引用指向的资源被Put释放
这里应该去掉Get函数中的引用
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406201121149.png)


GDB调试
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406201558960.png)
可以访问类的成员，但是无法调用成员函数
如果类有一个map成员，则不能调用其`[]`，`find()`，`at()`
但是能看到其具体数据，可以直接看到想要的key对应的value
如果value是一个指针，可以对该指针进行强转再解引用，进而获取指针指向的具体数据
如果强转涉及到模板参数，模板参数的错误将导致强转失败
可以通过`ptype`查看变量对应的类型
如果该模板参数具有子类，可以直接打印变量，通过信息里的虚函数表确定变量类型

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202406221619686.png)