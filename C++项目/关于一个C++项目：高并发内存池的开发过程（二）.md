```toc
```
上篇文章梳理了内存申请操作的流程，大概测试了一下，没有发现什么问题。这篇文章将梳理内存释放操作的流程，若申请操作中，有些细节没有把控好，那么释放操作将bug不断。有些bug我至今还在调试...所以，这篇文章的梳理，侧重点依然是逻辑结构。代码的细节可能存在问题，并且解决这些问题的过程与心得，也能够单独梳理成一篇文章，好好总结一番了。
***
## 内存释放操作的总述
当某些条件满足时，thread cache将闲置的内存块归还给central cache，central cache将span归还给page cache，page cache将小块的span合并成大块的span，从而减少外碎片的问题。内存释放的过程是内存块的不断整合，由少到多，由小到大，将零散的内存块重新整合成新的span。

（注意：一些参数大小固定，这里先展示出来）
```cpp
static const size_t NFREELIST = 208;
static const size_t MAX_SIZE = 256 * 1024;
static const size_t PAGE_SHIFT = 12;
static const size_t NPAGES = 129;
```
***
## thread cache
首先是thread cache的内存块归还，当FreeList的长度大于“该链表一次能申请的最大数量”时需要进行归还操作（*当然，还可以添加更多的条件，使空闲内存块尽可能合理的归还*），这些内存块被归还给central cache。

“该链表一次能申请的最大数量”由接口size_t SizeClass::\_adapt_count(size_t block_bytes)获取。

可以注意到，我们无法很快的得知FreeList的长度。所以这里再为FreeList创建一个变量以保存其长度，并且修改相关push和pop接口，因为进行这些操作会改变链表的长度，以下给出FreeList到目前为止的实现：
```cpp
class FreeList
{
private:
	void* _head = nullptr;
	size_t _fetch_count = 1;
	size_t _length = 0;
public:
	void _push_front(void* obj);
	void* _pop_front();
	bool _empty();
	// 将start到finish之间的内存块，头插到链表中
	void _range_push(void* start, void* finish, size_t count);
	// 将链表头部往后count块内存块删除，以输出型参数的方式返回区间中的头尾两块内存块
	void* _range_pop(size_t count);
	// 关于_fetch_count与_length的操作
	inline size_t _get_fetch_count() { return _fetch_count; }
	inline void _add_fetch_count() { ++_fetch_count; }
	inline size_t _get_length() { return _length; }
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
		_length += 1;
	}
}

void* FreeList::_pop_front()
{
	if (_head != nullptr)
	{
		void* ret = _head;
		_head = next(_head);
		_length -= 1;
		return ret;
	}

	return nullptr;
}

void FreeList::_range_push(void* start, void* finish, size_t count)
{
	if (start && finish)
	{
		next(finish) = _head;
		_head = start;
		_length += count;
	}
}

void* FreeList::_range_pop(size_t count)
{
	void* start = _head;
	void* finish = start;
	for (size_t i = 0; i < count - 1; ++i)
	{
		finish = next(finish);
	}

	_head = next(finish);
	next(finish) = nullptr;

	_length -= count;
	return start;
}
```
***
通过_get_length()接口获取单链表长度后，与_get_fetch_count()接口返回的值进行比较。如果链表长度大于等于一次能获取的最大内存块数量，说明当前链表中有很多闲置的内存块没有使用，此时可以将一些内存块还给central cache。

FreeLlist的归还逻辑：调用链表的\_range\_pop接口（*该接口会将count个内存块从链表中删除，并返回区间中第一块内存块的起始地址*），获取被删除内存块区间的起始块地址后，以start为参数调用central cache的接收接口。该接口会将所有内存块插入到对应span中，该接口将在后续实现。

（*\_range\_pop只返回内存块区间中第一块内存块的地址，为什么不返回最后一块内存块的地址？由于区间的最后一块内存块的指针域指向空，所以判断指针域是否指向空就能得知内存块区间是否遍历结束。*）

修改thread cache的\_deallocate接口：
```cpp
void ThreadCache::_deallocate(void* obj, size_t block_size)
{
	if (obj && block_size <= MAX_SIZE)
	{
		size_t bucket_index = SizeClass::_get_index(block_size);
		FreeList& aim_list = _free_lists[bucket_index];
		aim_list._push_front(obj);

		// 链表长度大于等于一次能获取的最大内存块数量，需要归还
		if (aim_list._get_length() >= aim_list._get_fetch_count())
		{
			void* start = aim_list._range_pop(aim_list._get_fetch_count());

			// TODO调用central的接收接口
		}
	}
}
```
***
## central cache
接着是实现central cache的接收接口：除了内存块区间的起始块地址，thread cache还需要将FreeList所处的桶号告知central cache，因为thread cache和central cache的哈希映射规则相同，所以FreeList的桶号不需要做转化。

内存区间中的每一块内存块可能属于不同的span，central cache需要将其归还到正确的span中。而我们只知道内存块的地址，如何通过内存块地址得知其位于的span？所以这里要设计算法实现内存块与span之间的映射，这个算法之后再实现。假设已经得知内存块位于的span，central cache要将其插入到span的FreeList中，当span的所有内存块都被归还时，central cache就要将该span向上交付，还给page cache。

如何得知span的内存块都被归还？这里为span添加一个成员变量：\_used_count，表示该span下，被使用的内存块数量。取走span的内存块，该变量的值将增加。收回span的内存块，该变量的值将减少。当_used_count的值为0，说明没有线程使用该span的内存块。此时将central cache要把该span归还给page cache。

以下实现central cache的接收接口：
```cpp
void CentralCache::_return_blocks_to_spans(void* start, size_t bucket_index)
{
	_span_lists[bucket_index]._mtx.lock();
	while (start)
	{
		Span* aim_span = TODO();// 将内存块地址映射为span的地址
		if (aim_span == nullptr)
		{
			std::cerr << "内存块没有从属的span" << std::endl;
		}
		else
		{
			next(start) = aim_span->_free_list;
			aim_span->_free_list = start;
			--(aim_span->_used_count);

			if (aim_span->_used_count == 0)
			{
				// TODO::向page归还span
				_span_lists[bucket_index]._erase(aim_span);
			}
		}
		start = next(start);
	}
	_span_lists[bucket_index]._mtx.unlock();
}
```
注意：高并发的场景下，线程需要加锁访问central cache。
***
## page cache

接着实现TODO接口“page cache接收central cache归还的span”：这个接口需要接收一个span的地址，并尽可能地将该span和未使用的span进行合并，得到一个更大的span。如何合并出更大的span？这里要设计一个算法：查找与该span相邻的且未使用的span。

- 相邻的span：这个相邻指的是物理内存上的相邻，也就是页号的相邻，不是在SpanList链表中的逻辑位置相邻。我们可以通过span的页号推测与其相邻的页号，**将这些页号转换成span的地址**，进而判断这些span是否在使用
- 是否在使用：为Span添加一个bool类型的成员变量，true表示该span正在使用，false则相反
- 页号和地址的转换：Span结构存储了页号，因此我们可以通过span的地址获取该span的起始页号。即span\*->页号，但是我们不能通过页号获取span的地址，即页号->span\*。因此我们要维护页号到span\*的映射关系，可以用映射表unordered_map保存映射关系，只要page cache申请了span，就要为这些span建立页号和地址的映射关系

因此，我们可以根据页号查找映射表，从而得知该页是否被申请（*表中是否存在该key值*），以及被申请了，但是否被central cache使用（*key值存在，根据value值进行下一步判断*）？

先添加映射表以及相关操作：
```cpp
class PageCache
{
private:
	PageCache() {}
	PageCache(const PageCache& x) = delete;
public:
	static PageCache* _get_instance() { return &_instance; }
	// 获取一个跨越k页的span
	Span* _fetch_kspan(size_t k);
	// 接收来自central归还的span
	void _return_span_to_spans(Span* span);
	// 将k个页和span建立映射关系
	void _insert_map(page_t page_id, size_t k, Span* span);
	// 删除一对映射关系
	void _erase_map(page_t page_id);
private:
	static PageCache _instance;
	SpanList _span_lists[NPAGES];
	std::recursive_mutex _rmtx;;
	std::unordered_map<page_t, Span*> _pageid_to_span;
};

// 这个操作大多是被其他函数调用，因为其他函数都加锁了，为防止死锁，这个函数不用加锁
void PageCache::_insert_map(page_t page_id, size_t k, Span* span)
{
	if (span)
	{
		for (size_t i = 0; i < k; ++i)
		{
			_pageid_to_span[page_id + i] = span;
		}
	}
	else
	{
		assert(false);
		std::cerr << "PageCache::_insert_map::span不正确" << std::endl;
	}
}

void PageCache::_erase_map(page_t page_id, size_t k)
{
	for (size_t i = 0; i < k; ++i)
	{
		auto it = _pageid_to_span.find(page_id);
		if (it != _pageid_to_span.end())
		{
			//std::cout << "删除了" << it->first << "，页数为:" << it->second->_n << std::endl;
			_pageid_to_span.erase(page_id);
		}
		else
		{
			assert(false);
			std::cerr << "PageCache::_erase_map::page_id不存在" << std::endl;
		}
	}
}
```
其中关于映射表的成员：
- \_pageid\_to\_span就是所谓的映射表
- \_map\_span可以在映射表中为“从page\_id往后的k个页”与“span”建立映射关系
- \_erase_map可以在映射表中删除“从page\_id往后的k个页”与“span”的映射关系

然后为Span添加成员变量\_is_used，其初始值为flase，表示该span未被使用：
```cpp
struct Span
{
	Span* _prev = nullptr;
	Span* _next = nullptr;

	// 存储的内存页其实id与数量 
	page_t _id = 0;
	size_t _n = 0;

	// Span切好或合并后的单链表，以及分配出去的内存块数量
	void* _free_list = nullptr;
	size_t _used_count = 0;

	// 是否被使用
	bool _is_used = false;
};
```
接着实现page cache接收span的操作_return_span_to_spans：
- 先判断是否能合并出更大的span
  - 往前合并：判断该span的前一页是否被申请且未被使用，同时合并后的span大小不超过128页，若满足以上条件，合并两个span。然后继续往前合并
  - 往后合并：同样的也是满足以上三个条件就可以进行后续的合并
- 最后将合并好的span插入到SpanLlist中

需要注意的是，每次的合并都要删除映射表中，被合并span的映射关系：
```cpp
void PageCache::_return_span_to_spans(Span* span)
{
	// 合并的过程需要加锁
	std::unique_lock<std::recursive_mutex> guard(_rmtx);
	if (span->_n > NPAGES - 1)
	{
		system_dealloc((void*)(span->_page_id << PAGE_SHIFT), span->_n << PAGE_SHIFT);
		return;
	}

	// 往前合并
	while (true)
	{
		page_t prev_page_id = span->_page_id - 1;

		auto it = _pageid_to_span.find(prev_page_id);
		if (it == _pageid_to_span.end()) break;

		Span* prev_span = it->second;
		
		if (prev_span->_is_used == true) break;
		if (prev_span->_n + span->_n > 128) break;

		span->_page_id = prev_span->_page_id;
		span->_n += prev_span->_n;

		// 删除被合并的span的映射关系
		_erase_map(prev_span->_page_id, 1);
		if (prev_span->_n != 1)
			_erase_map(prev_span->_page_id + prev_span->_n - 1, 1);
		
		// 删除SpanList中被合并的span
		_span_lists[prev_span->_n]._erase(prev_span);
		delete prev_span;
	}

	// 往后合并
	while (true)
	{
		page_t next_page_id = span->_page_id + span->_n;

		auto it = _pageid_to_span.find(next_page_id);
		if (it == _pageid_to_span.end()) break;

		Span* next_span = it->second;

		if (next_span->_is_used == true) break;
		if (next_span->_n + span->_n > 128) break;

		span->_n += next_span->_n;
		
		// 删除被合并的span的映射关系
		_erase_map(next_span->_page_id, 1);
		if (next_span->_n != 1)
			_erase_map(next_span->_page_id + next_span->_n - 1, 1);
		
		// 删除被合并的span
		_span_lists[next_span->_n]._erase(next_span);
		delete next_span;
	}

	span->_is_used = false;
	_insert_map(span->_page_id, 1, span);
	if (span->_n != 1)
		_insert_map(span->_page_id + span->_n - 1, 1, span);

	_span_lists[span->_n]._push_front(span);
}
```
***
### central cache的TODO实现
回到central cache的接收接口_return_blocks_to_spans：
![](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230518175837.png)

其中有一个TODO没有实现，该接口需要将内存块地址转换成span的地址。有了映射表，这个接口就可以实现了：
```cpp
// 根据obj判断该内存块所属的span
Span* PageCache::_map_obj_to_span(void* obj)
{
	// 映射obj内存块时也需要加锁
	std::unique_lock<std::recursive_mutex> guard(_rmtx);

	page_t id = (page_t)obj >> PAGE_SHIFT;
	auto it = _pageid_to_span.find(id);
	if (it == _pageid_to_span.end())
	{
		return nullptr;
	}
	else
	{
		return it->second;
	}
}
```
page cache通过系统调用获取内存，而系统调用获取的内存以页为单位。一般情况下，页的大小为4kB或8kB，这里以4kB为例。系统调用返回的内存地址中，低12位是全0（*因为4kB是基本单位*），剩余位则用来表示页号。无论该页切分的内存块大小是多少，我们都能通过“将内存块地址右移12位”获取其属于的页号。有了页号，就能通过映射表找到其对应span的地址。
***
### 何时维护这张映射表？
什么时候要为页号和span\*建立映射关系？当page的哈希桶申请128页的span时，需要建立映射吗？答案是需要，只要page cache申请了内存块，就要建立映射关系。然而是否要维护所有页的映射关系，则需要根据情况决定：
- 当span在page cache中，未被使用central cache使用时，不需要维护所有页号与span\*的映射关系，只要维护起始页和终止页与span\*的映射关系即可。这两对映射关系将在合并span时被使用
- 当span从page cache分配出去时，需要建立所有页号与span\*的映射关系。这是因为cenctral cache在接收thread cache归还的内存块时，需要将内存块地址转换成span的地址
- 如果某个span的跨越了大量的页，而映射表值只维护其起始页和终止页的映射关系，这将导致中间页的内存块无法找到其对应span

综上，未被分配的span只要将首尾页号与span建立映射关系，被分配出去的span需要将所有页号与span建立映射关系。

以下是\_fetch_kspan接口的修改：
```cpp
Span* PageCache::_fetch_kspan(size_t k)
{
	// 获取PageCache的span时需要加锁
	std::unique_lock<std::recursive_mutex> guard(_rmtx);
	if (!_span_lists[k]._empty())
	{
		Span* ret_span = _span_lists[k]._pop_front();
		_insert_map(ret_span->_page_id, k, ret_span);
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
				
				_insert_map(old_span->_page_id, 1, old_span);
				_insert_map(ret_span->_page_id, k, ret_span);

				ret_span->_is_used = true;
				return ret_span;
			}
		}

		//  没有更大的Span可以用，此时向堆区申请一块128page的空间
		void* ptr = system_alloc(128);
		
		Span* max_span = new Span;
		max_span->_id = (page_t)ptr >> PAGE_SHIFT;
		max_span->_n = 128;
		_span_lists[128]._push_front(max_span);

		// 128页的span映射关系的建立（首尾）
		_insert_map(max_span->_page_id, 1, max_span);
		_insert_map(max_span->_page_id + 127, 1, max_span);
		
		return _fetch_kspan(k);
	}
}
```
没有合适的span而进行切分更大的span时，由于切分方向是从前往后，所以对于被切分的span，只需要为其起始页添加映射关系即可。对于被分配出去的span，需要为其所有页添加映射关系。
***
## tc_dealloc的修改

修改释放接口：使之不接收内存块大小，只接收内存块的起始地址。有了page cache的映射表，我们可以通过内存块地址获取其span地址。现在的问题是：我们需要知道该span切分的内存块大小是多少？只有知道内存块大小，我们才能找到thread cache中对应的桶，进而归还该内存块。所以我们需要为Span添加一个成员_block_size，表示该span切分的内存块大小：
```cpp
struct Span
{
	// .....
	// 该span切分的内存块大小
	size_t _block_size = 0;
};
```
每次切分span成内存块(*也就是这个接口：CentralCache::_get_span*)时，都要维护该变量的值。

以下是利用PageCache::\_map_obj_to_span接口改造的tc_deallocate：
```cpp
static void tc_deallocate(void* obj)
{
	if (pTLSThreadCache)
	{
		Span* span = PageCache::_get_instance()->_map_obj_to_span(obj);
		size_t block_size = span->_block_size;
		pTLSThreadCache->_deallocate(obj, block_size);
	}
}
```
***
## 申请大内存的适配

由于thread cache的哈希桶中，最大的内存块大小为256kB。当线程申请的内存大于256kB时，需要重新设计内存申请和释放的过程，以满足此类需求。

而问题的本质在于：是否要贯穿我们设计的三层结构申请内存？一般情况下，申请小于等于256kB的内存块将贯穿三层结构。但我们设计的page cache的哈希桶能存储128页的span，当页的基本单位为4kB时，256kB也只使用了哈希桶的前64页，还有一半的哈希桶没有被使用。当然了，因为哈希桶的结构限制，大于256kB的内存申请不能贯穿central cache和thread cache，只能经过page cache层。我们知道page cache的合并span同样也是可以减少内存碎片的，所以我们应该尽可能的利用page cache，将大于256kB但小于等于128页的内存申请操作贯穿page cache层。至于大于128页的内存申请操作，这里直接调用系统接口即可，不使用page cache的哈希桶，只使用其映射表以完成内存的归还操作。

总结下，针对内存块大小，我们可以分成三种情况：
- 小于等于256kB的内存块，申请操作将贯穿三层结构
- 大于256kB但小于等于128页的内存块，申请操作只涉及page cahe，不涉及其他两层结构
- 大于128页的内存块，直接调用系统接口

因此，我们需要对tc_allocate和tc_deallocate进行再设计：
```cpp
static void* tc_allocate(size_t need_bytes)
{
	if (pTLSThreadCache == nullptr)
		pTLSThreadCache = new ThreadCache;

	void* obj = nullptr;
	if (need_bytes > MAX_SIZE)
	{
		size_t block_size = SizeClass::_round_up(need_bytes);
		Span* span = PageCache::_get_instance()->_fetch_kspan(block_size >> PAGE_SHIFT);
		obj = (void*)(span->_page_id << PAGE_SHIFT);
	}
	else
	{
		obj = pTLSThreadCache->_allocate(need_bytes);
	}
	
	return obj;
}

static void tc_deallocate(void* obj)
{
	if (pTLSThreadCache)
	{
		Span* span = PageCache::_get_instance()->_map_obj_to_span(obj);
		size_t block_size = span->_block_size;
		if (block_size > MAX_SIZE)
			PageCache::_get_instance()->_return_span_to_spans(span);
		else
			pTLSThreadCache->_deallocate(obj, block_size);
	}
}
```
接着是page cache的两个接口的修改。

关于page cache的\_fetch\_kspan接口：
- 当申请的span页数超过128页时，直接调用系统接口，获取内存后向映射表添加映射关系（*只添加起始页和span\*的映射关系*），最后返回
- 当申请的span大小超过256kB时，操作需要贯穿page cache的哈希桶，但是需要添加起始页、终止页和span\*的映射关系，因为合并span时需要这两对映射关系
- 当申请的span大小小于等于256kB时，操作需要贯穿page cache的哈希桶，并添加所有页和span的映射关系
```cpp
// 系统调用的封装
// Common.hpp
inline static void* system_alloc(size_t kpage)
{
#ifdef _WIN32
	void* p = VirtualAlloc(0, kpage << PAGE_SHIFT, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

#else
	// 其他系统申请内存的函数
#endif
	if (p == nullptr)
		throw std::bad_alloc();

	return p;
}

inline static void system_dealloc(void* ptr, size_t total_size)
{
#ifdef _WIN32
	VirtualFree(ptr, total_size, MEM_DECOMMIT);
#else
	// 其他系统
#endif 

}

Span* PageCache::_fetch_kspan(size_t k)
{
	// 获取PageCache的span时需要加锁
	std::unique_lock<std::recursive_mutex> guard(_rmtx);
	// 申请大于的span大于128页
	if (k > NPAGES - 1)
	{
		void* obj = system_alloc(k);
		Span* ret_span = new Span;
		ret_span->_page_id = (page_t)obj >> PAGE_SHIFT;
		ret_span->_block_size = k << PAGE_SHIFT;
		ret_span->_is_used = true;
		ret_span->_n = k;
		// 添加起始页的映射关系
		_insert_map(ret_span->_page_id, 1, ret_span);
		return ret_span;
	}
	
	if (!_span_lists[k]._empty())
	{
		Span* ret_span = _span_lists[k]._pop_front();
		// 注意区分不同的页
		if (k > (MAX_SIZE >> PAGE_SHIFT))
		{
			_insert_map(ret_span->_page_id, 1, ret_span);
			_insert_map(ret_span->_page_id + ret_span->_n - 1, 1, ret_span);
		}
		else
			_insert_map(ret_span->_page_id, k, ret_span);

		ret_span->_is_used = true;
		return ret_span;
	}
	// ...
}
```

关于\_return_span_to_spans接口：
- 当归还的内存块页数大于128页，调用系统接口，直接将其返回给系统
- 当归还的内存块页数小于等于128页，需要进行合并操作
```cpp
void PageCache::_return_span_to_spans(Span* span)
{
	// 合并的过程需要加锁
	std::unique_lock<std::recursive_mutex> guard(_rmtx);
	if (span->_n > NPAGES - 1)
	{
		system_dealloc((void*)(span->_page_id << PAGE_SHIFT), span->_n << PAGE_SHIFT);
		return;
	}
	// ...
}
```

当时我在想：大于128页的内存申请直接在tc_alloc调用系统接口不就行了，这样脱离page cache不是更好吗？但是我忽略了一个重要的问题：tc_dealloc时，我只知道一个内存块的起始地址，怎么知道它的大小？小于等于128页的内存块可以通过page cache的映射表得知，而大于128页的内存块大小就不得而知了。所以这里还是需要记录内存块起始地址与内存块大小间的关系啊，因为已经实现了page cache的映射表，这里就不用再实现新的结构了。
***
## 写在最后
将代码转换成文字，并尽可能简洁准确地表述整体的样貌，解剖其中的关键。这使得我需要对文章进行不断的修改，然后再阅读，再修改，直到句子通顺，流畅。虽然这个过程枯燥乏味，但是它能帮助你理清思路与头绪。梳理完这两篇文章，对这个系统也有了更深刻的认识，现在我只希望能尽快找出其中的bug，让它尽量不出错，同时也要对其可读性和可维护性进行更多的考虑。

（*文章日后还会修改，若有错别字请见谅*）




