原项目地址：[高并发内存池项目: 高并发内存池项目的课堂板书+代码 (gitee.com)](https://gitee.com/bithange/project)
```toc
```
## 写在前面
本打算利用五一假期的时间将这个项目一口气开发完成，但由于本人的懈怠，这个项目最终只完成了80%。于是利用长假后的一天假期，将这个项目的框架搭建完成。本以为这个项目就此结束，但是调试项目所花费的时间远远超出了预期，直到现在这个项目还是有着未知的bug。修好了一个bug，测试用例终于能跑通时。以为终于结束，换个测试用例，下个bug又马上到来。比起上个项目的顺利，这个项目中的无数问题（*大多是运行时内存崩溃*）也算得上是C++的一种“魅力”吧。学了这么久C++，我也总会体会到了它的“魅力”。

之所以提前写这篇文章，是因为我不知道这个项目的调试何时能完成。故文章中列出的代码可能是有问题的，因此这篇文章的重点不在代码，重点在于**项目的整体框架**。并且在开发项目的过程中，记录下的类似草稿的文字（*包括代码*）已经超过了40000个字符，当项目完成时，我想我应该没有勇气再将这些笔记整理成文章了。因此这里就先整理一部分的笔记，后续应该还有调试与总结的文章。

刚才说到项目还在调试，而调试项目需要对整体的框架有着清晰的认识，大到类的结构，小到变量的名称，这些都要熟悉。所以总结这篇文章也是为了更好的调试。

最后，再次强调：这篇文章是一个未完成项目的逻辑梳理，文中的代码可能含有bug，若读者能够指出错误，还望与我理性讨论，我将感激不尽。

## 我的命名规则
- 类名：大驼峰
- 函数名与变量名：unix风格的下划线
- 为区分构造函数的形参与成员变量，类成员统一以下划线开头
***
malloc是语言提供的一个内存分配机制，其拥有自己的缓存机制以提高分配内存的效率。但malloc毕竟是一个通用的接口，为了通用性，其效率必然不高。因此，STL为频繁申请小块内存的操作定制了一个内存池，该内存池解决了多次申请小块内存导致的内存碎片问题，并且提高了小块内存的分配效率。而高并发内存池则是针对多线程场景下的优化，其目的在于解决内存碎片问题与提高多线程申请内存时的效率。

（注意：一些参数大小固定，这里先展示出来）
```cpp
static const size_t NFREELIST = 208;
static const size_t MAX_SIZE = 256 * 1024;
static const size_t PAGE_SHIFT = 12;
static const size_t NPAGES = 129;
```
## 定长内存池的实现
为什么要实现定长内存池？它是高并发内存池的子结构，同时通过它我们可以了解内存池的简单机制。定长内存池不考虑内存碎片的问题，即假设用户每次申请的内存长度固定。

与STL的空间配置器对比，为了追求极致的效率与减少内存碎片的产生，空间配置器的内存池不是定长的，用户可以申请的内存长度为8B、16B...、128B，它们都是8的倍数，当然了，配置器需要将用户申请的内存向8的倍数对齐。如果不使用配置器，用户直接调用malloc申请没有对齐过的小内存块，很容易导致内存碎片的问题。所以空间配置器不仅要保证极致的效率还要减少内存碎片的产生。这也是空间配置器和定长内存池的自由链表类型不同的缘由，这个结构将在之后看到。空间配置器的自由链表是一个哈希桶，存储的是二级指针。而定长内存池的自由链表是一个单链表，存储的是一级指针。从结构与原理的角度看，定长内存池的实现更为简单。

### 定长内存池的结构
首先，定长内存池以一个类模板的形式呈现。关于它的模板参数，可以是表示内存块大小的非类型模板参数。也可以是一个类型模板参数，此时内存块的大小就是该类型的大小。我用模板参数实现内存池，每次调用内存申请接口获取的内存块大小，等于该参数类型的大小。

接着是内存池的结构：与空间配置器一样，定长内存池用两个指针start和finish维护内存池。为什么不能只是用start指针呢？若只使用start指针，则无法确定内存池的使用情况：是否使用完了，还剩多少内存没有使用？这将导致内存池内存耗尽而我们却不知情，进而导致内存的越界访问。

由于内存池是连续的，用户申请与归还内存块的顺序大概率是不同的，所以我们无法只用内存池维护用户归还的内存块。因此，这里使用自由链表维护用户归还的内存块，当用户申请内存块时，优先使用自由链表中的内存块。因为从内存池拿走的内存块被归还到自由链表中了。因此除了一个内存池，我们还需要一个自由链表维护内存块的申请与归还。

### 定长内存池的内存操作
定长内存池主要的接口有两个：new_obj和delete_obj，分别负责分配内存块给用户与归还用户的内存块。

- new_obj：优先检查自由链表是否为空
  - 若不为空，则将链表的第一个内存块返回（*头删*）
  - 若为空，则填充内存池，并将第一块内存块返回
- delete_obj：将用户归还的内存块头插到自由链表

在这两个接口中需要用到一个操作：获取节点的next指针时，需要将内存块强转为void\*\*再解引用得到next指针。我们知道32位系统与64位系统的指针大小是不相同的。在自由链表中，内存块需要用4/8字节的空间存储下一内存块的地址，类似单链表中的next指针。一种简单的解决方式是：使用**条件编译**判断当前系统的位数，从而确定指针的字节数。

另一种方法是使用\*(void\*\*)，void\*作为一个指针变量，在32位系统下大小为4字节，在64位系统下大小为8字节。将内存块的地址强转成void\*\*，再通过解引用该地址访问void\*长度字节的数据，这个操作在32位系统下可以访问4字节空间，64位系统下可以访问8字节空间，很好的规避了条件编译的繁琐。当然了，\*(void\*\*)中的void可以被替换成任意类型，如int，char。（*ps：SGI版本的STL中，使用了union联合体代替这种强转操作，这也是一种不错的解决方法*）

以下是定长内存池的所有实现：
```cpp
#pragma once
#include "Common.hpp"

#ifdef _WIN32
	#include <Windows.h>
#else
// 其他系统的头文件
#endif

static void* SystemAlloc(size_t kpage)
{
#ifdef _WIN32
	void* p = VirtualAlloc(0, kpage << 12, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

#else
	// 其他系统申请内存的函数
#endif

	if (p == nullptr)
		throw std::bad_alloc();

	return p;
}


template <class T>
class ObjectPool
{
private:
	char* _start_free = nullptr;
	char* _finish_free = nullptr;     // 维护内存池的两个指针
	void* _free_list = nullptr;  // 自由链表
public:
	// 内存块的申请与归还
	T* new_obj();
	void delete_obj(T* obj);
};

template <class T>
T* ObjectPool<T>::new_obj()
{
	T* obj = nullptr;

	// 优先向自由链表申请内存块
	if (_free_list)
	{
		obj = (T*)_free_list;
		_free_list = *(void**)_free_list;
	}
	// 自由链表无内存块，向内存池索要
	else
	{
		// 内存池空间不足
		if ((_finish_free - _start_free) < sizeof(T))
		{
			size_t bytes_to_get = sizeof(T) * 1024; // 默认每次申请1024个T的空间
			_start_free = (char*)SystemAlloc(bytes_to_get >> 12);
			// _start_free = (char*)malloc(bytes_to_get);
			if (_start_free == nullptr)
			{
				throw std::bad_alloc();
			}

			_finish_free = _start_free + bytes_to_get;
		}

		size_t need_bytes = sizeof(T) > sizeof(void*) ? sizeof(T) : sizeof(void*);
		obj = (T*)_start_free;
		_start_free += need_bytes;
	}

	// 定位new，调用T的构造函数以进行初始化
	new(obj)T;

	return obj;
}

template <class T>
void ObjectPool<T>::delete_obj(T* obj)
{
	obj->~T();

	*(void**)obj = _free_list;
	_free_list = (void*)obj;
}
```

其中对于申请内存的操作使用了系统调用，不再使用malloc。Windows下malloc封装了VirtualAlloc，Linux下malloc封装了brk，利用条件编译使不同的系统调用相应的系统调用，从而使我们的定长内存池完全脱离malloc。并且，定长内存池的new_obj和delete_obj操作不仅要分配空间，还要进行初始化与销毁工作，所以new_obj的最后需要使用定位new，delete_obj一开始要调用对象的析构。

最后进行效率的测试，测试demo：
```cpp
struct TreeNode
{
	int _val;
	TreeNode* _left;
	TreeNode* _right;

	TreeNode()
		:_val(0)
		, _left(nullptr)
		, _right(nullptr)
	{}
};

void TestObjectPool()
{
	// 申请释放的轮次
	const size_t Rounds = 5;

	// 每轮申请释放多少次
	const size_t N = 100000;

	std::vector<TreeNode*> v1;
	v1.reserve(N);

	size_t begin1 = clock();
	for (size_t j = 0; j < Rounds; ++j)
	{
		for (int i = 0; i < N; ++i)
		{
			v1.push_back(new TreeNode);
		}
		for (int i = 0; i < N; ++i)
		{
			delete v1[i];
		}
		v1.clear();
	}

	size_t end1 = clock();

	std::vector<TreeNode*> v2;
	v2.reserve(N);

	ObjectPool<TreeNode> TNPool;
	size_t begin2 = clock();
	for (size_t j = 0; j < Rounds; ++j)
	{
		for (int i = 0; i < N; ++i)
		{
			v2.push_back(TNPool.new_obj());
		}
		for (int i = 0; i < N; ++i)
		{
			TNPool.delete_obj(v2[i]);
		}
		v2.clear();
	}
	size_t end2 = clock();

	std::cout << "测试数量n = " << N << std::endl;
	std::cout << "使用new花费的时间:" << end1 - begin1 << std::endl;
	std::cout << "使用定长内存池花费的时间:" WWWWWW<< end2 - begin2 << std::endl;
}
```
测试结果：
![](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230430104752.png)

![](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230430104709.png)

![](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230430104820.png)

## 高并发内存池的整体框架

![](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230429152308.png)
接着进入主题，介绍高并发内存池的整体框架。

一般内存池都需要考虑两个问题：
1. 内存碎片问题
2. 性能问题

高并发内存池还需要考虑：线程加锁导致的竞争问题。高并发内存池使用三层结构以解决这三个问题

- thread cache：每个线程独享一个thread cache，可以简单的理解为thread cache是每个线程的小内存池，由于被线程独享，所以不需要加锁访问，这样能减少锁的竞争
- central cache：当thread cache无可用内存时，需要向central cache索取内存。当thread cache闲置内存过多时，需要向central cache归还内存，以实现每个thread cache的均衡。由于central cache是多线程共享的，所以需要加锁访问
- page cache：当central cache无可用内存时，需要从page cache索取内存，page cache会通过系统调用获取堆资源。当central cache的闲置内存过多时，需要向page cache归还内存，以合并小块内存得到大块内存

所以，线程获取的内存资源总是通过调用系统接口获得，只不过我们在线程和系统接口间建立了三层缓存结构，以解决其他问题。

## thread cache

thread cache是一个线程独享的内存池，当线程需要内存资源时，优先向thread cache索取。和文章一开始实现的定长内存池不同，我们只能向定长内存池索取相同大小的内存块，但是线程的情况很复杂，需要向thread cache申请不同大小的内存块，每次申请的内存块大小极有可能不相同。

因此thread cache是不定长的，其自由链表也不再是简单的单链表，而是悬挂单链表的**哈希桶**。哈希桶其实是一个指针数组，其成员指向了和定长内存池一样的单链表。

至于thread cache的操作，主要有两个：\_allocate和\_deallocate。

对于\_allocate，我们假设线程能够申请的最大内存块大小为256kB，以8B作为桶的基本单位，每个桶存储的内存块大小为8B，16B，24B，...，256kB，但是这样的自由链表未免有些笨重（*注意，最大内存块为256kB*），竟然有32k个桶，显然是不合理的。所以我们需要设置对齐规则，使桶的数量尽可能的合理：
```cpp
// [1, 128]                         8B对齐           _free_lists[0, 16)
// [128 + 1, 1024]                  16B对齐          _free_lists[16, 72)
// [1024 + 1, 8 * 1024]             128B对齐         _free_lists[72, 128)
// [8 * 1024 + 1, 64 * 1024]        1024对齐         _free_lists[128, 184)
// [64 * 1024 + 1, 256 * 1024]      8 * 1024对齐     _free_lists[184, 208)
```
对于不同的区间设置不同的对齐数，若线程申请的内存块大小不是对齐数的整数倍，那么我们应该向上对齐，多分配给线程一些空间，就算线程压根不会使用，这是为了结构的简单考虑。根据上面的对齐规则，thread cache的自由链表只有208个桶，同时线程能够申请的最大内存块大小为256kB，在这样的规则下，空间的浪费只有10%左右，这是相当不错的了。

刚才提到了由向上取整导致的空间浪费问题，这样的空间浪费有一个名字：内碎片。通常我们说的内存碎片是指外碎片，外碎片存在于两个内存块之间，当我们需要连续的大块空间时，由于过多的外碎片，可能导致内存不足的情况，但这些外碎片大小的总和却大于我们需要的空间，所以此时的内存不足并不是真正的内存不足。总之，外碎片具有不连续，可利用性低的特点，如果内存中存在大量外碎片，系统的性能将受到极大的影响。而向上取整导致的内碎片是无法避免的，这是为了追求极致的效率而付出的代价。但是，用可控的内碎片减少不可控的外碎片的产生，也是一种问题解决办法。

综上，实现thread cache的allocate之前要实现两个主要操作：内存的向上取整和桶号映射。这里我们需要对自由链表的子结构进行封装：
```cpp
// 桶的最多数量与内存块的最大字节数
static const size_t NFREELIST = 208;
static const size_t MAX_SIZE = 256 * 1024;

// 提取出obj的指针字段，以引用的形式返回
static void*& next(void* obj)
{
	return *(void**)obj;
}

// 自由链表的子结构：单链表
class FreeList
{
private:
	void* _head = nullptr;
public:
	void _push_front(void* obj);
	void* _pop_front();
	bool _empty();
};

bool FreeList::_empty()
{
	return _head == nullptr;
}

void FreeList::_push_front(void* obj)
{
	if (obj)
	{
		next(obj) = _head;
		_head = (void*)obj;
	}
}

void* FreeList::_pop_front()
{
	if (_head != nullptr)
	{
		void* ret = _head;
		_head = next(_head);
		return ret;
	}

	return nullptr;
}
```
刚才说过thread cache的自由链表的本质是一个哈希桶，哈希桶也是一个单链表，其成员也悬挂着单链表，也就是FreeList结构。这个结构对外暴露三个接口：判空、元素的插入与删除。

然后是封装一个类SizeClass，以实现向上取整与桶号映射，这个类对外暴露两个接口，分别是：
- 字节数的向上取整
- 字节数和桶号映射

```cpp
class SizeClass
{
private:
	static inline size_t __round_up(size_t bytes, size_t align_num) { return (bytes + (align_num - 1)) & ~(align_num - 1); }
	static inline size_t __get_index(size_t bytes, size_t align) { return ((bytes + (1 << align) - 1) >> align) - 1; }
public:
	static inline size_t _round_up(size_t bytes);
	static inline size_t _get_index(size_t bytes);
};

size_t SizeClass::_round_up(size_t bytes)
{
	int ret = 0;

	if (bytes > 0)
	{
		if (bytes <= 128)
			ret = __round_up(bytes, 8);
		else if (bytes <= 1024)
			ret = __round_up(bytes, 16);
		else if (bytes <= 8 * 1204)
			ret = __round_up(bytes, 128);
		else if (bytes <= 64 * 1024)
			ret = __round_up(bytes, 1024);
		else if (bytes <= 256 * 1024)
			ret = __round_up(bytes, 8 * 1024);
		else
			std::cerr << "__round_up::bytes illegal" << std::endl;
	}
	return ret;
}

size_t SizeClass::_get_index(size_t bytes)
{
	int ret = 0;
	static int bucket_count[4] = { 16, 72, 128, 184 };
	if (bytes > 0)
	{
		if (bytes <= 128)
			ret = __get_index(bytes, 3);
		else if (bytes <= 1024)
			ret = __get_index(bytes - 128, 4) + bucket_count[0];
		else if (bytes <= 8 * 1204)
			ret = __get_index(bytes - 1024, 7) + bucket_count[1];
		else if (bytes <= 64 * 1024)
			ret = __get_index(bytes - 8 * 1024, 10) + bucket_count[2];
		else if (bytes <= 256 * 1024)
			ret = __get_index(bytes - 64 * 1024, 13) + bucket_count[3];
		else
			std::cerr << "__get_index::bytes illegal" << std::endl;
	}
	return ret;
}
```
以上两个类都定义在Common.hpp中。其中，向上取整和桶号映射使用了位运算，这样做可以让算法变得简洁与高效，这也是经常会用到的两个位运算操作。

回到thread cache的\_allocate：
- 我们需要将线程索取的内存块大小映射为桶号
- 根据桶号在哈希桶中找到这个桶
- 判断这个桶是否为空
  - 如果为空，那么thread cache就要向central cache索取内存（*这个操作之后实现*）
  - 如果不为空，那么thread cache就从桶中取出一块内存块（*FreeList的头删操作*）

至于thread cache的\_deallocate：
- 我们也需要将线程索取的内存块大小映射为桶号
- 根据桶号在哈希桶中找到这个桶
- 将内存块放回桶中（*FreeList的头插操作*）

ThreadCache的实现：
```cpp
#include "Common.hpp"

class ThreadCache
{
private:
	FreeList _free_lists[NFREELIST];
public:
	// 内存块的分配与归还
	void* _allocate(size_t bytes);
	void _deallocate(void* obj, size_t bytes);
};

// TLS
#ifdef _WIN32
static _declspec(thread) ThreadCache* pTLSThreadCache = nullptr;
#else
// 其他系统
#endif

void* ThreadCache::_allocate(size_t bytes)
{
	void* obj = nullptr;
	if (bytes <= MAX_SIZE)
	{
		size_t need_bytes = SizeClass::_round_up(bytes);
		size_t bucket_index = SizeClass::_get_index(bytes);
		if (_free_lists[bucket_index]._empty())
		{
			// TODO内存不足，向CentralCache索取内存
		}
		else
		{
			obj = _free_lists[bucket_index]._pop_front();
		}
	}

	return obj;
}

void ThreadCache::_deallocate(void* obj, size_t bytes)
{
	if (obj && bytes <= MAX_SIZE)
	{
		size_t bucket_index = SizeClass::_get_index(block_size);
		_free_lists[bucket_index]._push_front(obj);
	}
}
```
其中使用到了：TLS，thread local storage，线程本地存储。因为每个线程需要独享thread cache，所以每个线程都需要有一个自己的ThreadCache对象。我们用一个指针指向ThreadCache对象的地址，并且这个指针是线程独享的，当线程需要申请内存时，首先检查这个指针是否为空，若为空则需要new一个ThreadCache对象并保存其指针，然后通过这个指针调用ThreadCache的\_allocate和\_deallocate。

所以我们再封装两个函数，这两个函数对标malloc和free。对外只暴露这两个函数，它们会调用thread cache的\_allocate和\_deallocate，但在那之前会先获取TLS对象，也就是ThreadCache对象的指针，通过该指针调用ThreadCache的两个成员函数。
```cpp
// ConcurrentAlloc.hpp
#pragma once

#include "Common.hpp"
#include "ThreadCache.hpp"

static void* tc_allocate(size_t bytes)
{
	if (pTLSThreadCache == nullptr)
		pTLSThreadCache = new ThreadCache;
	
	return pTLSThreadCache->_allocate(bytes);
}

static void tc_deallocate(void* obj, size_t bytes)
{
	if (pTLSThreadCache)
		pTLSThreadCache->_deallocate(obj, bytes);
}
```
这样的话，对于线程的动态资源获取与释放的操作，只要调用tc_allocate和tc_deallocate就行了。
***
对于thread cache的\_allocate实现中，还有一个操作没有实现：某一链表中没有内存块时，ThreadCache应该向CentralCache索取内存，要实现这个操作就要先实现central cache的整体结构。

## central cache
对于central cache，我们需要完成其节点Span，桶（*存储Span的双向链表SpanList*），以及哈希桶的实现。也就是说，哈希桶作为指针数组，其成员指向类型为SpanList的双向链表，双向链表的节点类型是Span。

对于Span：
- Span通过保存起始页号以及拥有的页的数量存储连续的内存页，这里需要用到两个变量
- 其次它是一个节点，需要有prev和next两个指针
- 再者，因为Span跨越了多个内存页，而一个内存页的大小通常是4kB，因此central cache需要切分内存页，得到合适的连续的内存块，再分配给thread cache。这里用一个单链表保存切分好的内存块，但该链表不复用FreeList结构，Span只是保存链表的头指针。至于为什么，之后就知道了

对于SpanList：Span只是一个节点，我们需要用它来构成双链表SpanList。central cache的哈希桶需要包含208个SpanList（*和thread cache的FreeList数量一样*）。这些双链表在哈希桶中的映射规则和thread cache的哈希桶一样，当thread cache的某一桶号下的FreeList没有内存块可用时，我们只需要到central cache的相同桶号下的SpanList中，获取一个可用的Span，将其切分好的内存块给thread cache使用即可。

总结一下：
- Span作为双链表的节点，存储了多个连续的页的信息，并且将这些页切成了小内存块
- SpanList是将Span作为节点的双链表，悬挂了多个Span
- central cache的本质是由208个SpanList构成的一个哈希桶，桶的映射规则与thread cache是对应的

先实现Span结构：
```cpp
// 类似于单链表中的节点
// 其跨越/存储了多个内存页
struct Span
{
	Span* _prev = nullptr;
	Span* _next = nullptr;

	// 存储的内存页其实id与数量
	page_t _id = 0;
	size_t _n = 0;

	// Span维护的多个内存块
	void* _free_list = nullptr;
};
```
因为我不想写构造函数，所以给每个参数带上了缺省值。

再来实现SpanList结构，这是一个带头节点的双链表，对外提供insert和erase操作：
```cpp
// 由Span作为节点的双向链表
// SpanList是CenctralCache的一个桶
class SpanList
{
public:
	SpanList();
	~SpanList();
	// 在pos前插入节点
	void _insert(Span* obj, Span* pos);
	// 删除pos节点
	void _erase(Span* pos);
	// 为什么需要一把锁，这个等等就解释
	std::mutex _mtx;
private:
	Span* _head;
};

SpanList::SpanList()
{
	_head = new Span;
	_head->_prev = _head;
	_head->_next = _head;
}

SpanList::~SpanList()
{
	delete _head;
}

void SpanList::_insert(Span* obj, Span* pos)
{
	Span* prev = pos->_prev;
	
	prev->_next = obj;
	obj->_prev = prev;

	pos->_prev = obj;
	obj->_next = pos;
}

void SpanList::_erase(Span* pos)
{
	Span* prev = pos->_prev;
	Span* next = pos->_next;

	prev->_next = next;
	next->_prev = prev;
}
```

这是很简单的数据结构的实现，没啥好说的。

最后是CentralCache的单例：因为central cache只需要有一个，所以被设计成单例。又因为多个thread cache可以同时访问central cache，所以还需要对central cache加锁。但是考虑到加锁粒度的问题：由于thread cache的哈希桶映射规则和central cache一样，因此thread cache向central cache索取内存块时，只会访问其中一个SpanList。也就是说，获取内存块时，只需要对SpanList加锁，而不用对整个哈希桶加锁。所以SpanList还需要拥有一把锁，当thread cache访问SpanList前，需要先获取这把锁。

central cache的结构：
```cpp
#include "Common.hpp"

class CentralCache
{
private:
	SpanList _span_lists[MAX_SIZE];
public:
	static CentralCache* _get_instance() { return &_instance; }
private:
	CentralCache() {}
	CentralCache(const CentralCache& x) = delete;
	static CentralCache _instance;
};

CentralCache CentralCache::_instance;
```
这个单例没有必要设计成懒汉，为了程序的简单，这里就设计成饿汉。
***
聊完了central cache的具体结构，回到最开始要解决的问题：实现某一链表中没有内存块时，ThreadCache向CentralCache索取内存的操作。

首先，由于线程池中可申请的内存块大小极值相差过大，我们需要设计一个慢启动算法（*类似拥塞控制中的慢启动*）使thread cache每次索取的内存块数量是合理的。

为thread cache的FreeList结构添加一个变量\_fetch\_count，该变量存储了某一FreeList因为内存块不足而向central cache索取的次数，但这个数一开始是1不是0。每次索取内存块后，该数都会自增。修改后的FreeList结构，添加一个成员变量：
```cpp
// 自由链表的子结构：单链表
class FreeList
{
private:
	void* _head = nullptr;
	size_t _fetch_count = 1; // 添加变量
public:
	void _push(void* obj);
	void* _pop();
	bool _empty();
	// 关于_fetch_count的操作
	inline size_t _get_fetch_count() { return _fetch_count; }
	inline void _add_fetch_count() { ++_fetch_count; }
};
```
然后再设计一个算法：对于小的内存块，FreeList索取得多一些。对于大的内存块，FreeList索取得少一些，使得总体上索取的内存大小是差不多的。计算过程很简单：将内存块大小的最大值（*256kB*） / FreeList对应的内存块大小（*STL的空间配置器中，由于可申请的最大内存块大小为128B，最小为8B，两者相差不大。因此直接默认给20块内存块，但是在我们的内存池中，可申请的最大内存块大小为256kB，对于不同大小的内存块，肯定不能统一对待，多给几块256kB内存块可不是开玩笑的*）。但是该算式得到的结果在极端情况下，对于小内存块来说很大，对于大内存块来说很小，所以这里需要控制一下上下限：
```cpp
size_t SizeClass::_adapt_counts(size_t rounded_bytes)
{
	size_t nums = MAX_SIZE / rounded_bytes;

	if (nums < 2) nums = 2;
	else if (nums > 512) nums = 512;

	return nums;
}
```
拉低上限，提高下限，将结果控制在一个合理的范围。

回到一开始说的慢启动：我们无法保证thread cache可以使用完得到的内存块，可能得到了10块内存块，但thread cache只使用了1块。我们可以先给thread cache分配少量的内存块，如果thread cache还要内存块，我们再给它多一些。所以central cache给thread cache的内存块数量要从\_fetch\_count和\_adapt\_nums函数返回的结果中取最小值。一开始thread cache只能获取一块内存块，之后获取的内存块数量将线性增长。直到\_fetch\_count的值超过了\_adapt\_nums返回的结果，之后thread cache可以得到的内存块数量就等于\_adapt\_nums返回的结果。

我们将central cache分配内存块给thread cache的行为封装成函数\_fetch\_blocks，该函数将访问对应SpanList下的可用Span，然后将其维护的内存块尽可能多的返回：
```cpp	
// 将Span维护的内存块返回，以两个指针指向内存块的首尾
void CentralCache::_fetch_blocks(void*& begin, void*& end, size_t& njobs, size_t block_size)
{
	size_t bucket_index = SizeClass::_get_index(block_size);
	
	std::unique_lock<std::mutex> guard(_span_lists[bucket_index]._mtx);
	// TODO
	Span* span = TODO();
	if (span)
	{
		begin = span->_free_list;
		end = begin;

		int real_jobs = 1;
		// 找到自由链表的尾，并且修改njobs的值
		for (size_t i = 0; next(end) && i < njobs - 1; ++i)
		{
			end = next(end);
			++real_jobs;
		}
		njobs = real_jobs;

		// 将begin和end间的内存块从Span中删除
		span->_free_list = next(end);
		next(end) = nullptr;
	}
}
```
由于需要使thread cache互斥访问的SpanList，所以需要加锁。对于TODO函数：该函数会获取SpanList中的一个Span，这个之后再来实现。其中需要注意的是：begin和end分别指向多块内存块中的第一块与最后一块，一开始end等于begin，所以至少能返回一块，即real_jobs的值为1。之后的循环结束条件不是i < njobs而是i < njobs - 1。

总结下，对于\_fetch\_blocks：该接口会修改begin和end的值，表示其获取的内存块区间，同时也会修改njobs的值表示真正获取到的内存块数量。

thread cache的某一链表中没有内存块时，会向central cache索取内存。central cache提供给thread cache的接口已经实现，现在要实现thread cache的接口：FreeList需要将从\_fetch\_blocks得到的多个内存块中的第一块返回，并将剩下的悬挂到自己的FreeList中，其实涉及到一开始说的慢启动算法。该接口的实现：
```cpp
void* ThreadCache::_fetch_from_central_cache(size_t bucket_index, size_t block_size)
{
	// 慢启动，取两者的较小值
	size_t adapt_count = SizeClass::_adapt_count(block_size);
	size_t fetch_count = _free_lists[bucket_index]._get_fetch_count();
	size_t njobs = fetch_count < adapt_num ? fetch_count : adapt_num;
	// 自增_fetch_count
	if (njobs == fetch_count)
		_free_lists[bucket_index]._add_fetch_count();

	void* begin = nullptr;
	void* end = nullptr;
	CentralCache::_get_instance()->_fetch_blocks(begin, end, njobs, block_size);

	if (njobs > 1)
		_free_lists[bucket_index]._range_push(next(begin), end);
	return begin;
}
```
可以看到，只有最后得到的值和\_fetch\_count相同时，该变量才会自增，所以准确的说这个变量表示的不是该链表申请内存块的次数，因为当该变量的值超过adapt_count后，自增就会停止。

其中涉及到“\_range\_push：将多块内存块悬挂到自己的FreeList中”的操作，这里再为FreeList封装一个操作以实现该操作：
```cpp
void FreeList::_range_push(void* start, void* finish, size_t count)
{
	if (start && finish)
	{
		next(finish) = _head;
		_head = start;
	}
}
```
至此，thread cache的接口基本实现完成，以下是到目前为止它的所有实现：
```cpp
#pragma once

#include "Common.hpp"
#include "CentralCache.hpp"
class ThreadCache
{
private:
	FreeList _free_lists[NFREELIST];
	// 内存不足时，向CentralCache索取
	void* _fetch_from_central_cache(size_t bucket_index, size_t block_size);
public:
	// 内存块的分配与归还
	void* _allocate(size_t bytes);
	void _deallocate(void* obj, size_t bytes);
};

// TLS
#ifdef _WIN32
static _declspec(thread) ThreadCache* pTLSThreadCache = nullptr;
#else
// 其他系统
#endif

void* ThreadCache::_allocate(size_t bytes)
{
	void* obj = nullptr;
	if (bytes <= MAX_SIZE)
	{
		size_t need_bytes = SizeClass::_round_up(bytes);
		size_t bucket_index = SizeClass::_get_index(bytes);
		if (_free_lists[bucket_index]._empty())
		{
			obj = _fetch_from_central_cache(bucket_index, need_bytes);
		}
		else
		{
			obj = _free_lists[bucket_index]._pop_front();
		}
	}

	return obj;
}

void ThreadCache::_deallocate(void* obj, size_t bytes)
{
	if (obj && bytes <= MAX_SIZE)
	{
		size_t bucket_index = SizeClass::_get_index(bytes);
		_free_lists[bucket_index]._push_front(obj);
	}
}

void* ThreadCache::_fetch_from_central_cache(size_t bucket_index, size_t block_size)
{
	// 慢启动，取两者的较小值
	size_t adapt_num = SizeClass::_adapt_counts(block_size);
	size_t fetch_count = _free_lists[bucket_index]._get_fetch_count();
	size_t njobs = fetch_count < adapt_num ? fetch_count : adapt_num;
	// 自增_fetch_count
	if (njobs == fetch_count)
		_free_lists[bucket_index]._add_fetch_count();

	void* begin = nullptr;
	void* end = nullptr;
	CentralCache::_get_instance()->_fetch_blocks(begin, end, njobs, block_size);

	if (njobs > 1)
		_free_lists[bucket_index]._range_push(next(begin), end);
	return begin;
}
```
***
而刚才实现的接口，central cache中的\_fetch\_blocks中还有一个接口没有实现：从SpanList中获取Span。

当SpanList下没有一个可用Span时，就要向page cache申请一个span（*该接口也是之后实现*），而SpanList得到的新的span需要被切分成对应大小的内存块后才可用使用。内存块的切分由这个接口负责：
```cpp
Span* CentralCache::_get_span(SpanList& span_list, size_t block_size)
{
	Span* cur = span_list._begin();
	Span* end = span_list._end();

	// 遍历SpanList，查找是否有可用的Span
	while (cur != end)
	{
		if (cur->_free_list)
		{
			// 注意不要erase(cur)
			return cur;
		}
		cur = cur->_next;
	}

	// 先解桶锁，使后续的thread cache可用访问（可能归还，可能索取）
	span_list._mtx.unlock();
	// TODO::无可用Span，向page cache求助
	Span* ret_span = TODO();

	if (ret_span)
	{
		// 获取到span后，切分span
		void* start = (void*)(ret_span->_id << PAGE_SHIFT);
		void* finish = (void*)((ret_span->_id + ret_span->_n) << PAGE_SHIFT);

		ret_span->_free_list = start;

		page_t pending = (ret_span->_id << PAGE_SHIFT) + block_size;

		while (pending <= (page_t)finish - block_size)
		{
			next(start) = (void*)pending;
			pending += block_size;
			start = next(start);
		}
		next(start) = nullptr;
	}
	// 此时要返回到_fetch_blocks函数，需要加锁
	span_list._mtx.lock();
	return ret_span;
}
```
\_get\_span首先会查找SpanList下是否有可用span，若一个可用span都没有则会向page cache求助，得到新的span，将其切分后返回。

关于切分算法的实现：
- 先定义两个指向头尾的void\*指针，此时span的内存地址为\[start, finish)
- 然后定义一个整数pending，切分内存块时，该数表示下一个内存块的地址
- 将start作为\_free\_list的值，然后开始将start和pending进行链接
- pending每次增加一个内存块的大小，当pending的值在\(finish - block_size, finish)之间时，以pending开始的内存块将越过finish，使用这样内存块将导致非法访问，所以此时将停止循环
- 最后，将span的最后一块内存块的下一指针置空，表示链表的结束

其中要注意的是：void\*的加减法和普通整数的加减法不同，所以pending使用page_t类型定义。

至此，将目前为止实现的CentralCache展示出来：
```cpp
#pragma once

#include "Common.hpp"
#include "PageCache.hpp"

class CentralCache
{
private:
	SpanList _span_lists[MAX_SIZE];
public:
	static CentralCache* _get_instance() { return &_instance; }
	// 将Span维护的内存块返回，以两个指针指向内存块的首尾
	void _fetch_blocks(void*& begin, void*& end, size_t& njobs, size_t block_size);
	// 获取一个Span
	Span* _get_span(SpanList& span_list, size_t block_size);
private:
	CentralCache() {}
	CentralCache(const CentralCache& x) = delete;
	static CentralCache _instance;
};

CentralCache CentralCache::_instance;

void CentralCache::_fetch_blocks(void*& begin, void*& end, size_t& njobs, size_t block_size)
{
	size_t bucket_index = SizeClass::_get_index(block_size);

	_span_lists[bucket_index]._mtx.lock();
	Span* span = _get_span(_span_lists[bucket_index], block_size);
	if (span && span->_free_list)
	{
		begin = span->_free_list;
		end = begin;

		int real_jobs = 1;
		// 找到自由链表的尾，并且修改njobs的值
		for (size_t i = 0; next(end) && i < njobs - 1; ++i)
		{
			end = next(end);
			++real_jobs;
		}
		njobs = real_jobs;
		span->_used_count += njobs;

		// 将begin和end间的内存块从Span中删除
		span->_free_list = next(end);
		next(end) = nullptr;
	}
	_span_lists[bucket_index]._mtx.unlock();
}

Span* CentralCache::_get_span(SpanList& span_list, size_t block_size)
{
	Span* cur = span_list._begin();
	Span* end = span_list._end();

	// 遍历SpanList，查找是否有可用的Span
	while (cur != end)
	{
		if (cur->_free_list)
		{
			// 注意不要erase(cur)
			return cur;
		}
		cur = cur->_next;
	}

	// 先解桶锁，使后续的thread cache可用访问（可能归还，可能索取）
	span_list._mtx.unlock();
	// TODO::无可用Span，向page cache求助
	Span* ret_span = TODO();

	if (ret_span)
	{
		// 获取到span后，切分span
		void* start = (void*)(ret_span->_id << PAGE_SHIFT);
		void* finish = (void*)((ret_span->_id + ret_span->_n) << PAGE_SHIFT);

		ret_span->_free_list = start;

		page_t pending = (ret_span->_id << PAGE_SHIFT) + block_size;

		while (pending <= (page_t)finish - block_size)
		{
			next(start) = (void*)pending;
			pending += block_size;
			start = next(start);
		}
		next(start) = nullptr;
	}
	// 此时要返回到_fetch_blocks函数，需要加锁
	span_list._mtx.lock();
	return ret_span;
}
```
和thread cache一样，central cache也有一个等待实现的接口，因为这个接口也涉及page cache的实现，所以我们需要先实现page cache。

## page cache

page cache用来做什么？当central cache无内存可用时，就会向page cache索取内存。对于thread cache，当其需要内存块时，central cache会将内存块返回，而central cache向page cache申请的却是span，所以central cache自己承担了将span切分为内存块的工作。这个接口刚刚也实现过了，可以看出central cache是一个承上启下的结构。

关于page cache，其结构也是一个哈希桶，也就是一个数组，数组成员指向SpanList，和central cache的结构一样。但是映射规则和central不一样，它有129个桶，第1桶不使用，从第二个桶开始使用，也就是从下标为1的位置使用数组。每个下标对应的是span的页数，比如下标为5的位置，该位置的SpanList悬挂的都是大小为5页的span。并且page cache也是一个单例，central cache也需要互斥访问它，总之，先搭建个大概的结构：
```cpp
class PageCache
{
private:
	PageCache() {}
	PageCache(const PageCache& x) = delete;
public:
	static PageCache* _get_instance() { return &_instance; }
private:
	static PageCache _instance;
	SpanList _span_lists[NPAGES];
	std::recursive_mutex _rmtx;
};
PageCache PageCache::_instance;
```

当central向page索取span时，需要告知page自己要索取的span的页数，然后page就会找到对应的桶，检查该SpanList下是否有span可用。
- 若没有span，page cache不会直接向堆区申请内存，而是往后查找是否有更大的span可以使用
- 若找到了更大的span就进行切分，将切好的span返回，剩下的span重新放入哈希桶中
- 若没找到更大span，才会向堆区申请内存
```cpp
Span* PageCache::_fetch_kspan(size_t k)
{
	// 获取PageCache的span时需要加锁
	std::unique_lock<std::recursive_mutex> guard(_rmtx);
	if (!_span_lists[k]._empty())
	{
		Span* ret_span = _span_lists[k]._pop_front();
		return ret_span;
	}
	else
	{
		for (int i = k + 1; i < NPAGES; ++i)
		{
			// 往后找更大的span，进行切分
			if (!_span_lists[i]._empty())
			{
				// 找到不为空的SpanList，获取第一个Span，该Span的大小为i个page
				Span* old_span = _span_lists[i]._pop_front();
				Span* ret_span = new Span;
				ret_span->_n = k;
				ret_span->_id = old_span->_id;

				old_span->_n -= k;
				old_span->_id += k;
				_span_lists[old_span->_n]._push_front(old_span);

				return ret_span;
			}
		}

		//  没有更大的Span可以用，此时向堆区申请一块128page的空间
		void* ptr = SystemAlloc(128);
		
		Span* max_span = new Span;
		max_span->_id = (page_t)ptr >> PAGE_SHIFT;
		max_span->_n = 128;
		_span_lists[128]._push_front(max_span);

		return _fetch_kspan(k);
	}
}
```
关于切分算法实现：向后找到更大的未使用的span后，需要new一个新的span保存切分后得到的合适大小的span。切分需要修改新旧span的信息，比如起始页号、跨越的页数。最后将修改后的旧span重新插入到SpanList中，将切好的span返回。

当page cache山穷水尽（*没有更大内存块可用时*），就会调用VirtualAlloc向堆区申请资源（*其他系统的调用接口不同*），申请到一个128页的span，将其插入到哈希桶中。本着代码重用的原则，我们可以使函数递归调用自己。这个递归会继续申请k页的span，也就是会将128页的span进行切分，一个span被重新插入，一个span被返回。

所以，到现在这个内存池的基本框架已经搭建起来，总结一下：
- 内存池分为三层缓存，从堆区到线程依次是：page cache、central cache、thread cache
- 线程获取和释放内存不再调用malloc和free，而是调用我们封装的接口：tc_allocate和tc_deallocate
- tc_allocate的逻辑已经实现：
  - 调用thread cache的_allocate申请内存，该接口将返回一块内存块，其中会产生内碎片
  - 若thread cache内存块不足，将调用_fetch_from_central_cache向central cache申请内存块
  - central cache作为一个承上启下的结构，其存储的不是内存块，而是span
  - central cache将通过_fetch_blocks接口响应thread cache的_fetch_from_central_cache请求，将内存块返回
  - 当central cache无可用span，将会调用_get_span接口，向page cache申请span，该接口同时负责将申请到的span切分成内存块的操作
  - page cache通过_fetch_kspan接口响应central cache的_get_span请求，将一个k页的span返回
  - 当page cache也无可用span时，会调用系统接口，向系统申请堆区资源

当然了，以上梳理的只是一个大致的流程，其中还有很多细节没有展示。
***
这篇文章主要整理高并发内存池的内存申请逻辑，考虑到文章字数已经很多了，关于内存释放逻辑将在下一文章中展示。