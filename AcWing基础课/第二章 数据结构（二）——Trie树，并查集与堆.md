```toc
```
***
## Trie树
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202312261953532.png)
以及在特殊情况下的优化，代替unordered_map<string, int>保存数字编号，[2977. 转换字符串的最小成本 II - 力扣（LeetCode）](https://leetcode.cn/problems/minimum-cost-to-convert-string-ii/description/)这样的substr操作，使用Trie树更为高效

用来高效的存储和查找字符串集合的数据结构
Trie树的根节点不存储有效字符，假设根节点为第0层，Trie树从第1层开始存储有效数据
假设n为字符串str的长度，`k < n` 时，第k层存储字符串的第k个字符

通过Trie树可以快速地查找某个字符串是否出现在集合中
如何插入一个字符串到Trie树中？
遍历字符串的每个字符，从字符串的第一个字符与Trie树的第0层开始，判断下一层（*第一层*）是否存在存储当前字符的节点
- 若不存在，则插入该节点，并走向该节点
- 否则走向该节点
重复以上操作直到字符串的所有字符被遍历完，并对最后一个节点进行标记，表示从根节点到该节点构成的字符串数量+1

如何查询字符串在集合中出现的次数？
遍历字符串的每一个字符，从第一个字符和第0层开始，判断下一层是否存在存储当前字符的节点
- 若不存在，表示集合中没有该字符串
- 若存在，走向该节点
重复以上操作直到所有字符遍历完，对于遍历的最后一个Trie树节点，返回其标记：集合中从根节点到该节点构成的字符串数量

如何用代码实现Trie树？
```cpp
// 假设Trie树中只存储小写字符
// son[N][26]：Trie树中所有节点与其子节点，以下标的形式存储与索引
// cnt[N]：Trie树中，从根节点到某个节点，组成的字符串数量
// idx：用于索引Trie树中的节点
int son[N][26], cnt[N], idx;
char str[N];

void insert(char str[])
{
	int p = 0;
	for (int i = 0; str[i]; ++ i )
	{
		int u = str[i] - 'a';
		if (!son[p][u]) son[p][u] = ++ idx;
		p = son[p][u];
	}
	cnt[p]++;
}

int query(char str[])
{
	int p = 0;
	for (int i = 0; str[i]; ++ i )
	{
		int u = str[i] - 'a';
		if (!son[p][u]) return 0;
		p = son[p][u];
	}
	return cnt[p];
}
```
***
## 并查集
用来快速的处理
1. 将两个集合合并
2. 询问两个元素是否在一个集合当中

暴力做法：用`belong`数组存储某个元素属于的集合，`belong[x] = a`：表示x属于`a`这个集合
此时询问两个元素是否在一个集合中，这个操作是O(1)的
但将两个集合合并，假设一个集合的元素有1000个，一个集合的元素有2000个，至少需要修改1000次belong数组，才能完成这个操作
这时使用并查集完成这两个操作，能够达到近乎O(1)的时间复杂度

分别用树的形式存储每个集合，每颗树用根节点进行唯一标识（*根节点表示集合*）
树中除了根节点，其他节点存储其父节点的下标
判断某个节点在哪个集合中，我们只需要从该节点开始，不断地往父节点遍历，找到其根节点，就找到其属于的集合

如何判断树根：特殊设置根节点`p[x] == x`，只有父节点的父指针指向自己
如何求x的集合编号：`while (p[x] != x) x = p[x]`，最后的x就是集合编号
如何合并两个集合：将一棵树插入到另一棵树下。假设x和y分别是两集合的根节点，`p[x] = y`，将x的父指针指向y，不再指向自己，从而将x集合合并到y集合中，x集合中所有的元素都属于y节点

其中第二个问题可以优化：查询x节点属于哪个集合时，需要从x节点遍历到根节点，此时将这条路径上的所有点的父指针都指向根节点
之后再查找这条路径上的节点属于的集合时，只需一次查找便能找到所属集合
以上优化叫做路径压缩，其本质就是尽可能的降低树的高度

模板：
```cpp
int find(int x)
{
	if (x != p[x]) p[x] = find(p[x]);
	return p[x];
}

void merge(int x, int y)
{
	p[find(x)] = p[find(y)];
}
```

有些题目需要记录集合中的节点数量，此时额外使用一个size数组保存该值。注意：只保证数组中根节点对应的值有意义
合并数组时，需要维护size数组，若x集合合并了y集合，那么`size[x] += size[y]`

***
## 堆
需要实现的操作：
1. 插入一个数
2. 求集合中的最小值
3. 删除最小值
4. 删除任意一个元素
5. 修改任意一个元素

其中1~3操作是STL的`priority_queue`支持的操作

堆的结构：完全二叉树，小根堆，即堆顶为最小值
存储方式：完全二叉树用一维数组存储，当x是某个元素的下标时
- x的左孩子：`2 * x`
- x的右孩子：`2 * x + 1`
- x的父亲：`x / 2`

删除最小值涉及到`down`操作，`down`将某个元素向下调整，使调整后的堆满足小根堆的性质
为什么需要down，因为删除操作将堆顶元素与最后一个叶子进行了交换，并删除最后一个叶子（*原堆顶元素*），此时堆顶的值变大。不满足小根堆的性质，需要将变大的值向下调整，使堆重新满足小根堆的性质
除此之外，还有`up`操作，删除最小值不会用到`up`，只有在某个位置的值变小时，才需要对其进行`up`操作

模板：
cnt为堆中元素的数量，注意：不使用数组的0号下标，因为`0`不利于计算左右孩子的下标
从`heap`数组的1号下标开始使用

1. 插入一个数：`heap[++ cnt] = x, up(cnt);`
2. 求集合中的最小值：`heap[1];`
3. 删除最小值：`heap[1] = heap[cnt], down(1);`
4. 删除任意一个元素：`heap[k] = heap[cnt], down(k), up(k);`
5. 修改任意一个元素：`heap[k] = x, down(k), up(k);`

为什么4、5操作需要`down`与`up`操作？因为删除或者修改某个元素之后，该位置的值可能变大可能变小。为了简化代码，不写判断条件直接进行两个操作，但是这两个操作只会执行一个

为了使堆满足小根堆的性质需要`down`操作：当前节点小于两个子节点，若不满足该性质，需要将当前节点与子节点中的较小节点进行交换
当前节点走到某个子节点的位置上，具有了新的子节点
交换后的三个节点满足了小根堆的性质，此时继续判断当前节点与其两个新的子节点是否满足小根堆的性质
若不满足则继续交换，直到满足或者当前节点无子节点

```cpp
void down(int u)
{
	int t = u;
	if (u * 2 <= n && h[u * 2] < h[t]) t = u * 2;
	if (u * 2 + 1 <= n && h[u * 2 + 1] < h[t]) t = u * 2 + 1;
	if (u != t) swap(h[t], h[u]), down(t);
}
```

对数组中下标为u的元素进行`down`操作，u作为父节点，与其两个子节点进行比较，用t保存三者中的最小值
- 若t没有变化，说明u就是三者中的最小值，满足小根堆性质，不用更新
- 若t发生了变化，交换u与三者中的最小值，使三者满足小根堆的性质
  - 然后以交换后的u作为父节点，继续以上比较直到满足性质或u走到最后成为叶子

建堆时从下标为`cnt/2`的节点开始：`x / 2`操作将得到下标为x节点的父节点
对于叶子，不需要`down`操作，因为叶子没有子节点。所以从最后一个具有子节点的节点开始，向上进行`down`操作直到根节点
最后一个具有子节点的节点，其子节点一定是叶子。问题转换下就是：最后一个叶子的父节点
`cnt`为整个堆的元素个数，`cnt`指向最后一个叶节点，该节点的父节点开始`down`到根节点，建堆操作就完成了
```cpp
for (int i = cnt / 2; i; -- i ) down(i);
```
计算每一层的down次数，利用错位相减，可以得到时间复杂度为`O(n)`

删除操作：
```cpp
h[1] = h[cnt --], down[1];
```
***
操作4~5，需要支持删除与修改第k个插入堆中的数
在堆结构中，第k个插入的数与数组（*堆结构用一维数组维护*）下标没有直接的关系。在链表结构中，第k个插入的数，k就是`e`数组的下标，这是直接映射的关系。由于堆需要满足某些性质，元素插入后可能要进行元素间的交换，由于元素后续可能的改动，所以第k个插入的元素与数组下标没有直接的关系

因此，要支持操作4~5就要维护第k个插入的数在堆中的下标，用数组`ph[k]`表示第k个插入的点在堆中的下标。假设现在修改了堆的第k个元素，此时需要对其进行`up`或者`down`操作，以维护堆的性质。不论是`up`操作还是`down`操作，都涉及到两个元素的交换。这个操作会修改两个元素在堆中（*一维数组*）中的下标，因为修改的是第k个插入数，元素交换后修改`ph[k]`为新的下标即可。但是虽然知道另一个元素在堆中的下标，交换操作可以进行，却不知道该元素是第几个操作的，也就无法维护`ph`数组

为了维护`ph`数组，可以再维护一个数组`hp[k]`：堆中下标为k的点，是第几个插入的点
这样在交换时，我们知道两个数在堆中的下标，就可以通过`hp`数组获取其是第几个插入的，进而维护`ph`数组

以上，要支持操作4~5，就要维护`ph`和`hp`数组，其中`hp`数组是为了维护`ph`数组而存在的
什么时候要维护ph数组？向堆插入元素与交换两数时
- 向堆中插入元素时，假设带元素是第k个插入的，那么该元素就存储在堆中下标为k的位置。注意：数组从1开始使用。`ph[k] = k`，之后`ph[k]`可能修改，这是因为第k个插入的数进行了交换
- 交换两数时，假设交换第i个插入和第j个插入的数
  - `swap(ph[i], ph[j])`，表示第i个插入的数现在在堆中的下标是第j个插入的数在堆中的下标，同理,第j个插入的数在堆中的下标是...
  - 但是交换前我们不知道两数是第几个插入的，我们只知道两数在堆中的下标。假设两数的下标分别是x和y，所以刚才的操作是`swap(ph[hp[x]], ph[hp[y]])`
  - 最后`swap(hp[x], hp[y])`，表示堆中下标为x的数是第j次插入的，下标为y的数是第i次插入的

注意：数组名中，p表示第几次插入，h表示数在堆中的下标
从上面的推导也可以看出，`hp`数组是为了维护`ph`数组而存在的。首先是因为我们要支持删除/修改第k个插入的数，所以需要知道插入次数到元素在堆中下标的映射，从而维护了`ph`数组
而交换操作时，由于我们需要维护`ph`数组，但我们只知道两数的下标，不知道两数的插入次数，无法在`ph`数组中索引元素
因此无法维护`ph`数组，此时创建`hp`数组，维护下标与插入次数的映射关系，先通过`hp`数组得知元素的插入次数，再维护`ph`数组，当然`ph`数组也需要维护

模板：
```cpp
// x和y为交换两数在堆中的下标
void head_swap(int x, int y)
{
	swap(h[x], h[y]);
	swap(ph[hp[x]], ph[hp[y]]);
	swap(hp[x], hp[y]);
}

void down(int u)
{
	int t = u;
	if (u * 2 <= n && h[u * 2] < h[t]) t = u * 2;
	if (u * 2 + 1 <= n && h[u * 2 + 1] < h[t]) t = u * 2 + 1;
	if (u != t) heap_swap(u, t), down(t);
}

void up(int u)
{
	while (u / 2 && h[u / 2] > h[u]) heap_swap(u, u / 2), u /= 2;
}
```
***
## Trie树练习题
### 835. Trie字符串统计
[835. Trie字符串统计 - AcWing题库](https://www.acwing.com/problem/content/837/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230623204258.png)
```cpp
#include <iostream>
using namespace std;

const int N = 1e5 + 10;

int son[N][26], cnt[N], idx;
char str[N], op[2];
int n;

void insert(char str[])
{
    int p = 0;
    for (int i = 0; str[i]; ++ i )
    {
        int u = str[i] - 'a';
        if (!son[p][u]) son[p][u] = ++ idx;
        p = son[p][u];
    }
    cnt[p]++;
}

int query(char str[])
{
    int p = 0;
    for (int i = 0; str[i]; ++ i )
    {
        int u = str[i] - 'a';
        if (!son[p][u]) return 0;
        p = son[p][u];
    }
    return cnt[p];
}

int main()
{
    scanf("%d", &n);
    while (n -- )
    {
        scanf("%s%s", op, str);
        if (op[0] == 'I') insert(str);
        else printf("%d\n", query(str));
    }
    return 0;
}
```
***
### 143. 最大异或对
[143. 最大异或对 - AcWing题库](https://www.acwing.com/problem/content/145/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230626063630.png)

先从暴力解法入手：
- 枚举数组中的每个数$a_i$
- 对于$a_i$，每次枚举数组中的一个数$a_j$
- $a_i$ ^ $a_j$，做异或运算，直到$a_j$枚举完数组中的所有数
- 枚举$a_i$ ^ $a_j$的过程中维护运算结果最大值

以上解法需要枚举`n`次$a_i$，然后再枚举`n`次 $a_j$。时间复杂度为O($n^2$)
思考如何优化暴力解法，每次枚举`n`次$a_i$，这个无法做优化。每次$a_i$需要和数组中所有的数进行异或运算吗？
根据异或运算，相同为0，不同为1。我们想要得到的结果最大，也就是结果中的1越多越好，准确的说，高位的1越多越好
题目中给的数据能够用`int`存下，且不是负数
用i遍历$a_i$的每一位，初始i指向$a_i$的第31位，最后指向$a_i$的第1位
从$a_i$的最高位开始，通过找当前位与$a_i$不同的数，理想情况下，最终能找到一个所有位和$a_i$不同的数，此时异或的结果为全1
从$a_i$的最高位开始，若有些数和$a_i$的当前位相同，那么$a_i$就不用与这些数做异或运算，因为运算结果为0，$a_i$只要和当前位与其不同的数异或即可
把和$a_i$当前位不同的数扔掉，这样就能缩小数据范围
若所有数的当前位都和$a_i$的当前位相同，即$a_i$的当前位与所有数的异或结果都是0，无法得到1，此时无法缩小数据范围

用Trie树存储所有的数据，从每一个数据的高位开始，Trie树只存储0与1，由于所有的数都是整数，所以Trie数不存储符号位，即Trie的高度为31
Trie树中，若某个元素为0，表示该元素不存在
考虑最坏情况，每个数都需要使用Trie树的31个节点存储，那么Trie树中有`31 * n`个节点

用Trie树存储每个数，在异或运算中。知道两数中的一数$a_i$，如何找到另一个数$a_j$，使得异或的结果最大？
从Trie的根节点开始，从$a_i$的`第31位`到`第1位`，根据$a_i$的每一位数，反向选择$a_j$
- 若Trie树中，存在某些数的`第i位`和$a_i$的`第i位`相反，那么这些数与$a_i$异或的结果中，`第i位`就是1
  - 假设异或结果的初始值0，此时`res += 1 << i` 
- 若Trie树中，所有数的`第i位`与$a_i$的`第i位`相同，不论怎样，$a_i$异或的结果中，`第i位`就是0，此时res（*异或结果*）的值不变

debug：`query`函数中，res为两数异或的结果，不是和$a_i$异或的$a_j$
若修改`query`的逻辑，使res为$a_j$，那么维护最大异或结果以及输出最大异或结果时，需要进行异或运算
只有Trie树存在某些数，它们的某一位与$a_i$不同时，res才会改变
```cpp
// query的res保存aj
#include <iostream>
using namespace std;

const int N = 1e5 + 10, M = 3100010;
int son[M][2], a[N], idx;
int n, res;

void insert(int x)
{
    int p = 0;
    for (int i = 30; ~i; -- i )
    {
        int u = x >> i & 1;
        if (!son[p][u]) son[p][u] = ++ idx;
        p = son[p][u];
    }
}

int query(int x)
{
    int p = 0, res = 0;
    for (int i = 30; ~i; -- i )
    {
        int u = x >> i & 1;
        if (son[p][!u]) res += !u << i, p = son[p][!u];
        else res += u << i, p = son[p][u];
    }
    return res;
}

int main()
{
    scanf("%d", &n);
    for (int i = 0; i < n; ++ i ) scanf("%d", &a[i]), insert(a[i]);
    
    for (int i = 0; i < n; ++ i ) res = max(res, a[i] ^ query(a[i]));
    printf("%d\n", res);
    
    return 0;
}
```

```cpp
// query的res保存异或运算的结果
#include <iostream>
using namespace std;

const int N = 1e5 + 10, M = 3100010;
int son[M][2], a[N], idx;
int n, res;

void insert(int x)
{
    int p = 0;
    for (int i = 30; ~i; -- i )
    {
        int u = x >> i & 1;
        if (!son[p][u]) son[p][u] = ++ idx;
        p = son[p][u];
    }
}

int query(int x)
{
    int p = 0, res = 0;
    for (int i = 30; ~i; -- i )
    {
        int u = x >> i & 1;
        if (son[p][!u]) res += 1 << i, p = son[p][!u];
        else p = son[p][u];
    }
    return res;
}

int main()
{
    scanf("%d", &n);
    for (int i = 0; i < n; ++ i ) scanf("%d", &a[i]), insert(a[i]);
    
    for (int i = 0; i < n; ++ i ) res = max(res, query(a[i]));
    printf("%d\n", res);
    
    return 0;
}
```
***
## 并查集练习题
### 836. 合并集合
[836. 合并集合 - AcWing题库](https://www.acwing.com/problem/content/838/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230624062316.png)
```cpp
#include <iostream>
using namespace std;

const int N = 1e5 + 10;
int n, m, x, y, p[N];
char op[2];

int find(int x)
{
    if (x != p[x]) p[x] = find(p[x]);
    return p[x];
}

int main()
{
    scanf("%d%d", &n, &m);
    for (int i = 0; i < n; ++ i ) p[i] = i;
    
    while (m -- )
    {
        scanf("%s%d%d", op, &x, &y);
        if (op[0] == 'M') p[find(x)] = p[find(y)];
        else printf("%s\n", find(x) == find(y) ? "Yes" : "No");
    }
    
    return 0;
}
```
debug：find中return对象是`p[x]`不是`x`，因为路径压缩中将修改x的父节点指针
***
### 837. 连通块中点的数量
[837. 连通块中点的数量 - AcWing题库](https://www.acwing.com/problem/content/839/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230625161430.png)
```cpp
#include <iostream>
using namespace std;

const int N = 1e5 + 10;
int p[N], s[N];
char op[3];
int x, y;

int n, m;

int find(int x)
{
    if (x != p[x]) p[x] = find(p[x]);
    return p[x];
}

int main()
{
    scanf("%d%d", &n, &m);
    for (int i = 1; i <= n; ++ i) p[i] = i, s[i] = 1;
    
    while (m -- )
    {
        scanf("%s", op);
        
        if (op[0] == 'C') 
        {
            scanf("%d%d", &x, &y);
            x = find(x), y = find(y);
            if (x != y) s[y] += s[x], p[x] = p[y];
        }
        else if (op[1] == '1') 
        {
	        scanf("%d%d", &x, &y);
	        printf("%s\n", find(x) == find(y) ? "Yes" : "No");
	    }
        else scanf("%d", &x), printf("%d\n", s[find(x)]);
    }
    return 0;
}
```
1. 因为`s`数组中，只有根节点对应下标有效，所以查找某个元素所属集合的元素数量时（*索引`s`数组时*），需要先进行`find`操作
2. 合并两集合中，需要特判。上一道板子题没写特判不会有问题，而这题涉及到询问集合元素数量的操作。若不进行特判，同一集合中的元素合并将导致该集合的数量翻倍
***
### 240. 食物链
[240. 食物链 - AcWing题库](https://www.acwing.com/problem/content/242/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230626063537.png)

并查集的变形题，一般的并查集只有合并两集合与判断两元素是否在同一集合中的操作
此题需要在这两个操作的基础上，维护一些额外信息。如连通块中的数量，额外维护了集合中的元素数量这一信息
根据这题的题意，我们需要额外维护的信息是：食物链关系。题目给定的食物链中只有三个物种，所以我们需要在并查集中表示这三个物种的吃与被吃的关系，同时也要表示哪些动物是同一物种

一个比较容易陷入的误区是：同一集合中的动物属于同一物种，这个很容易想到，不过此时食物链关系要如何表示？读者可以想想，一开始我就是这个思路，不过感觉食物链关系很难想，于是放弃

其中一个解法是：根据元素到根节点的距离维护食物链关系
1. 距离% 3 == 0，元素与根节点属于同一物种
2. 距离% 3 == 1，元素属于的物种吃根节点属于的物种
3. 距离% 3 == 2，元素属于的物种被根节点属于的物种吃

总之就是：父子节点之间存在吃与被吃的关系，子节点可以吃父节点
x节点的父节点为p，p的父节点为pp，pp的父节点为ppp
x吃p，p吃pp，pp吃ppp，由于食物链关系存在环，所以ppp吃x
由于食物链关系中只有三个物种，所以x节点与ppp节点为同一物种

根据题目给定的“话”，维护并查集。一开始每个动物自己为一个集合，每个集合表示根据“真话”维护的食物链关系
根据这些关系，推断接下来的话是否为“真话”
- 若为真话，根据该信息合并并查集或者向集合中添加元素
- 若为假话，则增加出现的假话数量

保存`距离数组d`，表示并查集中的元素到父节点的距离。进行并查集的查找操作时，将进行路径压缩，路径压缩需要维护`d数组`
根节点到查找元素之间的所有元素都将作为根节点的子节点，此时距离数组的含义变成了元素根节点的距离
如何维护`d数组`呢？以下是路径压缩模板
```cpp
int find(int x)
{
	if (x != p[x]) p[x] = find(p[x]);
	return p[x];
}
```
`数组p`保存节点的父节点，若当前元素不是集合的根节点，那么进行递归查找。最后修改每个节点的父节点为根节点的操作，是从根节点的子节点开始修改，向下到当前节点结束
由于`数组d`保存节点到父节点的距离（*注意这个距离和层数没有关系，这个距离指的是食物链中的距离*），假设从当前节点到根节点的路径上有4个节点（*不包括根节点*），从根节点的子节点开始维护`数组d`：`d[x] += d[px]`，节点到父节点的距离加等父节点到其父节点的距离
根节点到其父节点的距离为0（*根节点的父节点就是自己*），由于更新从根节点的子节点开始，到当前节点结束，所以这样的更新操作的结果就是：路径上所有节点在`d数组`中的含义变成了**到根节点的距离**
根据节点与根节点的绝对距离，就能推导出节点之间的相对距离，也就能推导出动物之间的食物链关系
所以递归模板就能改写为：
```cpp
int find(int x)
{
	if (x != p[x])
	{
		int t = p[x];
		p[x] = find(p[x]);
		d[x] += d[t];
	}
	return p[x];
}
```

若题目表示X与Y具有某些关系时，那么需要根据题目之前描述过的X与Y的关系，推导当前描述是否正确，此时X与Y处于同一并查集中
若题目之前没有描述过X与Y的关系，那么需要建立X与Y之间的关系，此时X与Y不处于同一并查集中

如何建立X与Y是同类的关系？X与Y分别位于两个不同的集合中，由于之前进行过`find`操作，此时X与Y的父节点就是根节点。假设X位于的集合被合并到Y位于的集合中，那么X集合的根节点作为了Y集合根节点的子节点
X集合的根节点为px，Y集合的根节点为py
即需要满足关系：`(d[px] + d[x] - d[y]) % 3 == 0`，在合并后的集合中，X到根节点的距离与Y到节点的距离同余3
假设两者距离相同，即`d[px] + d[x] == d[y]`，那么两者肯定同余3。此时修改`d[px] = d[y] - d[x]`，为什么不需要修改`d[x]`？，修改了`d[px]`，那么以px节点为祖先节点的所有节点在`数组d`中的信息都需要修改
由于find操作将维护数组d，只要修改了`d[px]`，那么其子孙节点在`数组d`中的信息都将通过`find`操作修改
由于每次查询操作都将调用`find`操作，因此就能保证每次查询的结果都是正确的

如何建立X吃Y这个关系？和建立X与Y是同类的关系一样，假设X所属集合被合并到Y所属集合中
合并后的集合中，两元素需要满足关系`(d[x] + d[px] - d[y] - 1) % 3 == 0`，即`d[x] + d[px]`与`d[y] + 1`同余，假设两者的值相等，那么肯定是同余的
所以合并后，需要修改`d[px] = d[y] + 1 - d[x]`

题目给定的话中，若两动物在同一集合中，此时不需要建立关系。只需判断两动物之间的关系是否和话中描述的一样，若存在矛盾说明这句话是错误的
```cpp
#include <iostream>
using namespace std;

const int N = 1e5 + 10;
int p[N], d[N];
int n, k, px, py;
int res, t, x, y;

int find(int x)
{
    if (x != p[x]) 
    {
        int t = p[x];
        p[x] = find(p[x]);
        d[x] += d[t];
    }
    return p[x];
}

int main()
{
    scanf("%d%d", &n, &k);
    for (int i = 0; i < n; ++ i ) p[i] = i;
    while (k -- )
    {
        scanf("%d%d%d", &t, &x, &y);
        
        if (x > n || y > n) 
        {
            res++;
            continue;
        }
       
        px = find(x), py = find(y);
        if (t == 1)
        {
            if (px == py && (d[x] - d[y]) % 3) res ++ ; // 与描述矛盾
            else if (px != py)
            {
                p[px] = py;
                d[px] = d[y] - d[x];
            }
        }
        else
        {
            if (px == py && (d[x] - d[y] - 1) % 3) res ++ ;
            else if (px != py)
            {
                p[px] = py;
                d[px] = d[y] + 1 - d[x];
            }
        } 
    }
    printf("%d", res);
    
    return 0;
}
```
***
## 堆练习题
### 838. 堆排序
[838. 堆排序 - AcWing题库](https://www.acwing.com/problem/content/840/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230622091435.png)

获取数组，建立小堆，进行m次删除操作，每次删除前输出堆顶元素即可
```cpp
#include <iostream>
using namespace std;

const int N = 1e6 + 10;
int n, m;
int h[N];

void down(int u)
{
    int t = u;
    if (u * 2 <= n && h[u * 2] < h[t]) t = u * 2;
    if (u * 2 + 1 <= n && h[u * 2 + 1] < h[t]) t = u * 2 + 1;
    if (u != t) swap(h[u], h[t]), down(t);
}

int main()
{
    scanf("%d%d", &n, &m);
    for (int i = 1; i <= n; ++ i ) scanf("%d", &h[i]);
    for (int i = n / 2; i; -- i ) down(i);
    
    while (m -- )
    {
        printf("%d ", h[1]);
        h[1] = h[n -- ], down(1);
    }
    
    return 0;
}
```
***
### 839. 模拟堆
[839. 模拟堆 - AcWing题库](https://www.acwing.com/problem/content/841/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230623173610.png)

```cpp
int main()
{
    scanf("%d", &n);
    while (n --)
    {
        scanf("%s", op);
        if (op[0] == 'I') 
        {
            scanf("%d", &x);
            h[++ cnt] = x, ph[cnt] = cnt, hp[cnt] = cnt;
            up(cnt);
        }
        else if (strcmp(op, "PM") == 0) 
            printf("%d\n", h[1]);
        else if (strcmp(op, "DM") == 0) 
            h[1] = h[cnt -- ], down(1);
        else if (op[0] == 'D') 
            scanf("%d", &k), k = ph[k], h[k] = h[cnt -- ], down(k), up(k);
        else 
            scanf("%d%d", &k, &x), k = ph[k], h[k] = x, down(k), up(k);
    }
    return 0;
}
```
debug了很久，`heap_swap`,`down`,`up`三个函数没有问题，问题出在main函数的处理上
有三个问题：
1. `down`和`up`需要的参数为某个元素在堆中具体的下标，而不是某个元素的值，以上代码已经修改此错误。这个属于写题时思路有些模糊了，一时没有注意到
2. 以前写的堆只支持操作1~3，不支持操作4~5。所以删除堆顶元素时，直接是一个赋值操作，用最后元素替换堆顶元素。赋值操作写习惯了，而且“替换操作”这个概念也深入我心。所以实现删除任意元素的操作时直接一个赋值语句，可见以上代码。因为直接赋值没有维护`ph`与`hp`数组，导致后续的操作出现偏差，所以赋值操作应该改为`heap_swap`
3. 之前写单链表时，第k个插入的元素在数组中的下标就是k。而在堆中这个说法不成立，这个我之前也强调了。但是写代码时却没有多想，新增元素时维护`ph`与`hp`数组时，直接认为第k个插入元素在堆中的下标为k。只有进行了维护，“第k个插入元素的下标才可能不是k吧”。但是这个堆有删除操作，插入第k个元素时，堆中的元素数量不一定是`k - 1`。所以第k个插入的元素，不一定使用下标为k的位置。而在单链表中，由于数组中的元素不用连续，同时不用考虑内存泄漏问题，第k个插入的元素使用的下标一定是k。而堆的元素需要连续的存储，删除了某个元素后，这样的直接映射不成立。所以需要使用两个变量保存堆的元素数量以及插入的次数

以上，是我debug1小时得出的教训
总结：`down`与`up`接收某个元素的**下标**
需要维护`ph`与`hp`数组时，应该慎重使用直接赋值替换元素的操作，是否要维护`ph`与`hp`数组？
堆的元素是连续存储的，若支持删除操作，那么第k个插入的元素在数组中的下标不一定是k，这个假设总是成立，无论是在插入数据前还是在插入数据并维护后

以下是AC代码：
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 1e5 + 10;
int n, m, k, x, cnt, h[N], ph[N], hp[N];
char op[3];

void heap_swap(int x, int y)
{
    swap(h[x], h[y]);
    swap(ph[hp[x]], ph[hp[y]]);
    swap(hp[x], hp[y]);
}

void down(int u)
{
    int t = u;
    if (u * 2 <= cnt && h[u * 2] < h[t]) t = u * 2;
    if (u * 2 + 1 <= cnt && h[u * 2 + 1] < h[t]) t = u * 2 + 1;
    if (u != t) heap_swap(u, t), down(t);
}

void up(int u)
{
    while (u / 2 && h[u / 2] > h[u]) heap_swap(u / 2, u), u /= 2;
}

int main()
{
    scanf("%d", &n);
    while (n --)
    {
        scanf("%s", op);
        if (op[0] == 'I') 
        {
            scanf("%d", &x);
            h[++ cnt] = x, ph[++ m] = cnt, hp[cnt] = m;
            up(cnt);
        }
        else if (strcmp(op, "PM") == 0) 
            printf("%d\n", h[1]);
        else if (strcmp(op, "DM") == 0) 
            heap_swap(1, cnt -- ), down(1);
        else if (op[0] == 'D') 
            scanf("%d", &k), k = ph[k], heap_swap(k, cnt -- ), down(k), up(k);
        else 
            scanf("%d%d", &k, &x), k = ph[k], h[k] = x, down(k), up(k);
    }
    return 0;
}
```