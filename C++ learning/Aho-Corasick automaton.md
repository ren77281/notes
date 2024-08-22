```toc
```

## 写在前面
原作者的视频讲解链接：[[算法]轻松掌握ac自动机_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1uJ411Y7Eg/?spm_id_from=333.337.search-card.all.click)
原作者的代码实现：[data-structure-and-algorithm/aho_corasick.cpp at master · xiaoyazi333/data-structure-and-algorithm · GitHub](https://github.com/xiaoyazi333/data-structure-and-algorithm/blob/master/aho_corasick.cpp)
我的代码实现：[data_structure/Aho-Corasick automaton/Aho-Corasick automaton · sacajawea/code_store2023 - 码云 - 开源中国 (gitee.com)](https://gitee.com/sacajawea/code_store2023/tree/master/data_structure/Aho-Corasick%20automaton/Aho-Corasick%20automaton)
（*文章中的截图也来自原作者的视频*）

这篇文章是对上面视频的总结，原作者讲解的不错，看完一遍就能明白该算法的原理。只不过有些细节需要自己下来推敲一遍，为透彻的理解这个算法，我用C++重写了原作者的代码，并将思路总结成这篇文章。

## 算法概述
首先是两个概念的说明：
- P串：pattern string，用来搜索的字符串
- T串：text string，被搜索的文本串

AC自动机，这是一种多模式匹配算法，根据多个P串构建trie树，和KMP不同的是，只需要用T串遍历一次trie树，就可以知道哪些P串是在T串中出现过。
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230507111959.png)
 
视频中有这样一个画面，这是ac自动机最重要的部分：fail指针。
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230507110907.png)

fail指针：i->fail->j
word\[j\]是word\[i\]的最长后缀，什么是word\[i\]呢？

其表示从根节点出发到i节点构成的字符串。若fail指针指向根节点，说明该字符串没有最长后缀。若节点i的fail指针指向根节点以外的节点，假设是节点j，则说明word\[i\]的最长后缀是word\[j\]。

以上内容将在后续详细讲解。将视角拉高，回到ac自动机的整体结构上。

ac自动机可分为三个部分：
1. 将所有P串插入到trie树中
2. 为所有节点构建fail指针
3. 根据T串遍历trie树，查找匹配的P串

## trie树的构建
### trie树的节点结构

由于我们要做的是字符串匹配，字符包括字母、特殊字母、数字等，字符的数量无法确定。要如何表示一个节点的子节点？这里用映射表unordered_map存储子节点表示字符与子节点地址的映射关系，映射表的成员个数就是该节点的子节点个数。

除了基本的指针域，trie树还要保存一个fail指针，以表示当前串的最长后缀。

同时，我们需要标记该节点是否是某一P串的结束，这里用vector\<int\>表示这一信息，数组存储的是某一P串的长度，只要vector\<int\>的长度不等于0，就说明从根节点到该节点表示一个P串。当然，从中间节点开始到该节点也可能表示一个P串，所以这里不能只存储一个int，而是要用vector存储多个int。

至于该节点表示哪个字符，这里用一个char变量保存。以下是节点的结构：
```cpp
struct TrieNode
{
	std::vector<size_t> _exist;
	std::unordered_map<char, TrieNode*> _childs;
	TrieNode* _fail;
	char _name; 

	TrieNode(char name)
	{
		_fail = nullptr;
		_name = name;
	}
};
```
### 插入P串到trie树中
P串的第n个字符位于trie树的第n层，根节点位于树的第0层。
- 每次插入前先检查这一字符对应的子节点是否已经构建
  - 若构建，则向该子节点遍历
  - 若没构建，则new一个新的节点并将其作为该节点的子节点
- 最后将该串的长度填入最后一个节点的exist数组中

```cpp
void TrieTree::_insert_pstr(const std::string& p_str)
{
	node* cur = _root;
	for (size_t i = 0; i < p_str.size(); ++i)
	{
		if (cur->_childs.find(p_str[i]) == cur->_childs.end())
			cur->_childs[p_str[i]] = new node(p_str[i]);

		cur = cur->_childs[p_str[i]];
	}
	// 字符对应的最后节点需要维护存在信息
	cur->_exist.push_back(p_str.size());
}
```
### fail指针的创建
将所有P串插入到trie树中，此时trie树基本构建完成，但是还差最关键的一步：fail指针的创建。

先说明一些繁琐概念的别称：
- X节点：当前遍历的节点
- P节点：X节点的父节点
- father_fail指针：P节点的fail指针
- Y节点：X的fail指针指向的节点
- PY节点：father_fail指针指向的节点
- word\[i\]：从根节点到i节点构成的字符串

对于Y节点，其指向trie树中，从“根节点到X节点构成的字符串”的“最长后缀字符串的最后一个节点”。如何构建X的fail指针？可以通过father_fail指针，找到trie树中，从“根节点到P节点构成的字符串”的“最长后缀的最后一个节点PY”。只说概念太抽象了，举个例子：

比如this这个字符串
- 假设X节点为字符s，当word\[X\] = "this"时
- 那么P节点就是字符i，word\[P\] = "thi"
- father_fail指针指向PY节点，word\[PY\]可能为"hi"，也可能为"i"
  - 也可能word\[P\]没有最长后缀，此时PY节点指向根节点

当我们要找"this"的最长后缀（*也就是找Y节点*）时，先通过father_fail指针找到"thi"的后缀（*注意，这里没有最长*），如"hi","i",或者根节点。然后判断"hi","i"往下是否能构成"his","is"。
- 如果能构成，Y节点优先指向“较长后缀的最后一个节点”
- 如果不能构成，Y节点指向根节点

如何判断"hi","i"往下是否能构成"his","is"？具体做法是：通过判断PY节点的子节点中，是否存在表示字符和X节点表示字符相同的节点，在上面的例子中，就是判断PY节点是否存在表示字符s的子节点。

- 若不存在，则尝试word\[P\]剩下的后缀字符串
  - 直到所有后缀尝试完（*注意：不要忘记了空后缀，我们需要判断根节点的子节点中，是否存在表示字符和X节点表示字符相同的节点*），说明word\[P\]没有一个后缀有对应子节点。此时X的fail指针将指向根节点，表示trie树没有word\[X\]的最长后缀
- 若存在，X的fail指针指向该节点

如何知道trie树中word\[P\]的所有后缀字符串？很简单，一直遍历word\[P\]的最后一个节点的fail指针，直到该指针指向根节点，每个fail指针都表示一个后缀字符串，当fail指针指向根节点，此时表示word\[P\]的后缀字符串为空，后续没有word\[P\]的后缀了。

```cpp
void TrieTree::_build_tree(const std::vector<std::string>& p_strs)
{
	// 将p串插入到trie树中
	for (size_t i = 0; i < p_strs.size(); ++i)
	{
		_insert_pstr(p_strs[i]);
	}

	// 层序遍历，先将第一层（根节点为第0层）的节点入队，第一层节点的fail指针指向根节点
	std::queue<node*> level;
	for (const auto& kv : _root->_childs)
	{
		node* child = kv.second;
		level.push(child);
		child->_fail = _root;
	}
		

	while (!level.empty())
	{
		node* cur_node = level.front();
		level.pop();
		
		// 构建node所有子节点的fail指针
		for (const auto& kv : cur_node->_childs)
		{
			char name = kv.first;
			node* child_node = kv.second;

			node* father_fail = cur_node->_fail;

			// 找word[P]的后缀，需要PY节点含有对应字符的子节点
			// 所以当PY节点没有对应字符的子节点时，需要更新PY节
			// PY节点也就是father_fail指向的节点
			// 当PY节点为nullptr
			while (father_fail != nullptr && father_fail->_childs.find(name) == father_fail->_childs.end())
				father_fail = father_fail->_fail;

			if (father_fail == nullptr)
				child_node->_fail = _root;
			else
				child_node->_fail = father_fail->_childs[name];

			// 若最长后缀也是一个P串，此时要维护exits数组
			for (size_t j = 0; j < child_node->_fail->_exist.size(); ++j)
				child_node->_exist.push_back(child_node->_fail->_exist[j]);

			level.push(child_node);
		}
	}
}
```
## 搜索过程
- 从T串的第一个字符开始，从根节点遍历trie树。即当前节点为根节点，当前字符为T串的第一个字符
- 因为根节点不表示任何字符，所以我们需要判断当前节点的“子节点表示的字符“是否和当前字符匹配
- 若匹配，则向其遍历。判断该节点表示的字符是否是某一P串的最后字符
  - 若是，则根据exits数组存储的P串长度，在T串中截取P串，并保存
  - 若不是，什么都不做
- 若不匹配，则更新当前节点为fail指针指向的节点，重复上面的判断

需要注意的是：根节点的fail指针指向nullptr，此时我们不能更新当前节点为nullptr。因为后续要解引用当前节点，获取其子节点信息，解引用nullptr是非法的。

若当前节点的fail指针指向nullptr，则说明该节点是根节点。若当前字符与根节点的子节点匹配失败，则说明trie树中没有以该字符作为起始字符的P串，此时更新当前字符为下一字符即可。

 直到所有字符都经过匹配，搜索完成。

```cpp
void TrieTree::_query_tstr(std::unordered_set<std::string>& res, const std::string& t_str)
{
	node* cur_node = _root;
	for (size_t i = 0; i < t_str.size(); ++i)
	{
		char cur_char = t_str[i];
		// 若遇到空格，从根节点开始重新遍历trie树
		if (cur_char == ' ')
		{
			cur_node = _root;
			continue;
		}

		// 找到当前字符对应的节点
		// 若当前节点没有对应子节点且当前节点不是根节点，向fail指针指向的节点遍历
		// 注意：fail指针是否为nullptr是区分根节点与其他节点的关键
		while (cur_node->_fail != nullptr && cur_node->_childs.find(cur_char) == cur_node->_childs.end())
			cur_node = cur_node->_fail;
		
		// 找不到字符对应的节点，跳过该字符
		if (cur_node->_childs.find(cur_char) == cur_node->_childs.end())
			continue;
		// 找到字符对应的节点，向其遍历
		else
			cur_node = cur_node->_childs[cur_char];

		// 若当前节点表示某些P串的最后一个字符，返回P串
		for (size_t j = 0; j < cur_node->_exist.size(); ++j)
		{
			size_t length = cur_node->_exist[j];
			res.insert(t_str.substr(i - length + 1, length));
		}
	}
}
```
为什么要使用unordered_set\<string>存储P串结果，不能用vector\<string>吗？其实以上算法可能导致同一P串被多次匹配，使得vector\<string>保存重复的结果，所以我使用unordered_set进行去重。
## 测试程序

进行一些常规的测试，以检测代码是否能正常运行：
```cpp
#include "Aho-Corasick.hpp"

void UtilTest(const std::vector<std::string> p_strs, const std::string& t_str)
{
	TrieTree tree;
	std::unordered_set<std::string> res;
	
	std::cout << "trie树中的P串:" << std::endl;
	for (const auto& p_str : p_strs)
	{
		std::cout << p_str << ' ';
	}
	std::cout << std::endl;

	tree._build_tree(p_strs);
	tree._query_tstr(res, t_str);
	std::cout << t_str << "中，存在的P串:" << std::endl;
	for (auto& str : res)
	{
		std::cout << str << ' ';
	}
	std::cout << std::endl << "-------------------------------------------------------------" << std::endl;
}


int main()
{
	std::vector<std::string> p_strs = { "apple", "banana", "pear" };
	UtilTest(p_strs, "An apple a day keeps the doctor away");
	UtilTest(p_strs, "An apple a day keeps the doctor away. I love bananas and pears too!");

	p_strs = { "at", "cat", "hat" };
	UtilTest(p_strs, "the cat in the hat sat on the mat");
	
	return 0;
}
```
测试结果：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230515224818.png)
