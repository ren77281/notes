```toc
```
## 区间线段树
若要支持区间修改操作，需要引入懒标记，以维护sum信息的区间线段树为例，懒标记为add，表示以当前节点为根节点的子树中（不包括当前节点），所有节点需要加上的值
那么给一个区间中的所有数加上一个数时，只要给它们的父节点添加懒标记即可
当`query`或`modify`（需要使用或修改具体节点的信息）时，就需要将懒标记**向下传**，以保证数据的准确性
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230731110005.png)

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230731110459.png)
### pushdown操作
向下传递懒标记的操作叫做pushdown，若当前节点的懒标记不为0，此时pushdown操作将被执行
```cpp
void pushdown(int u)
{
	auto &root = tr[u], &left = tr[u << 1], &right = tr[u << 1 | 1];
	if (root.add)
	{
		left.add += root.add, left.sum += (left.r - left.l + 1) * root.add;
		right.add += root.add, right.sum += (right.r - right.l + 1) * root.add;
		root.add = 0;
	}
}
```
父节点为root，其左右节点为left与right，若节点保存的信息是区间和sum
首先要将子节点的懒标记累加上父节点的懒标记
其次，还要根据区间的节点数量维护sum信息
最后清除父节点的懒标记

根据懒标记的定义，将当前节点的懒标记向下传，将会清除当前节点的懒标记并修改其所有直接子节点的值与懒标记

除了pushdown操作，其他操作的实现与单点线段树相同，只不过要注意：每次要使用节点的值（`modify`和`query`）之前，先要pushdown懒标记，以维护数据的正确性

对于`modify`操作，若当前区间是修改区间的子集，此时没有必要修改当前区间的子区间，直接为当前节点添加懒标记即可，注意：还需要将当前节点的sum累加上所有子节点需要累加的add值
若当前区间不是修改区间的子区间，那么就需要pushdown当前节点，使当前节点维护的信息正确并传递懒标记，再递归modify其子节点。最后pushup当前节点，因为修改了当前节点的子节点信息
```cpp
void modify(int u, int l, int r, int d)
{
	if (l <= tr[u].l && tr[u].r <= r) 
	{
		tr[u].sum += (tr[u].r - tr[u].l + 1) * d;
		tr[u].add += d;
	}
	else
	{
		int mid = tr[u].l + tr[u].r >> 1;
		pushdown(u);
		if (l <= mid) modify(u << 1, l, r, d);
		if (r > mid) modify(u << 1 | 1, l, r, d);
		pushup(u);
	}
}
```
***
## 区间线段树练习题
### 243. 一个简单的整数问题2
[243. 一个简单的整数问题2 - AcWing题库](https://www.acwing.com/problem/content/244/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230731145451.png)

裸题，两个操作：区间修改与区间求和，完美匹配区间线段树的使用
```cpp
#include <iostream>
using namespace std;

typedef long long LL;
const int N = 1e5 + 10;
struct Node
{
    int l, r;
    LL sum, add;
}tr[4 * N];

int a[N];

void pushup(int u)
{
    tr[u].sum = tr[u << 1].sum + tr[u << 1 | 1].sum;
}

void pushdown(int u)
{
    Node &root = tr[u], &left = tr[u << 1], &right = tr[u << 1 | 1];
    if (root.add)
    {
        left.add += root.add, left.sum += (left.r - left.l + 1) * root.add;   
        right.add += root.add, right.sum += (right.r - right.l + 1) * root.add;
        root.add = 0;
    }
}

void build(int u, int l, int r)
{
    if (l == r) tr[u] = { l, r, a[l], 0 };
    else
    {
        int mid = l + r >> 1;
        tr[u] = { l, r };
        build(u << 1, l, mid), build(u << 1 | 1, mid + 1, r);
        pushup(u);
    }
}

void modify(int u, int l, int r, int d)
{
    if (l <= tr[u].l && tr[u].r <= r)
    {
        tr[u].sum += (tr[u].r - tr[u].l + 1) * d;
        tr[u].add += d;
    }
    else
    {
        int mid = tr[u].l + tr[u].r >> 1;
        pushdown(u);
        if (l <= mid) modify(u << 1, l, r, d);
        if (r > mid) modify(u << 1 | 1, l, r, d);
        pushup(u);
    }
}

LL query(int u, int l, int r)
{
    if (l <= tr[u].l && tr[u].r <= r) return tr[u].sum;
    else
    {
        pushdown(u);
        int mid = tr[u].l + tr[u].r >> 1;
        LL v = 0;
        if (l <= mid) v = query(u << 1, l, r);
        if (r > mid) v += query(u << 1 | 1, l, r);
        return v;
    }
}

int main()
{
    int n, m;
    scanf("%d%d", &n, &m);
    for (int i = 1; i <= n; ++ i ) scanf("%d", &a[i]);
    build(1, 1, n);
    
    char op[2]; int l, r, d;
    while ( m -- )
    {
        scanf("%s%d%d", op, &l, &r);
        if (op[0] == 'C')
        {
            scanf("%d", &d);
            modify(1, l, r, d);
        }
        else
        {
            printf("%lld\n", query(1, l, r));
        }
    }
    return 0;
}
```
***

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230731153013.png)
坐标轴的方向并不重要，无论怎样求出的面积都是相同的
根据y轴建立线段树，每个矩形有左右两条与y轴平行的边，根据x轴的方向从左往右扫描整个坐标中的矩形
遇到某个矩形的左边（与y轴平行）时，根据这条边所有的y坐标，将线段树中对应的线段+1
遇到某个矩形的右边（与y轴平行）时，根据这条边所有的y坐标，将线段树中对应的线段-1
根据每个矩形左右边的x轴坐标$x_i$扫描整个坐标系，每次扫描获取线段树中长度大于0的线段长度h，下次扫描$x_{i+1}$时，$h * |x_i - x_{i+1}|$的值就是要计算的一部分面积，将这些面积累加起来，得到所有矩形的面积并

综上，线段树需要支持两个操作，1. 将$[l, r]$的值加上x   2. 询问$[l, r]$中，区间长度大于0的区间总长度。由于这题的特殊性，每次询问操作的对象总是整个区间（整个y轴），即只返回根节点的信息
考虑节点需要保存的信息：1. cnt，类似与区间和线段树中的sum，表示当前区间被cnt个矩形覆盖 2. len，表示当前区间中，长度大于0的区间长度，每次询问返回len值

考虑如何实现线段树：由于每次只会询问根节点信息，所有query不需要使用pushdown操作
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230731153358.png)
