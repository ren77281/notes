```toc
```

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230808164543.png)

## lca
$O(mlogn)$，n为节点数量，m为询问次数，lca是一种在线处理询问的算法
自己也是自己的祖先

倍增：
$fa(i, j)$表示从i开始，向上走$2^j$步走到的点
j = 0，走到父节点
j > 0，分两步走，先走到$2^{j-1}$步再走$2^{j-1}$步，那么一共就会走$2^j$步，$fa(i, j) = fa(fa(i, j-1), j-1)$
$depth(i)$表示层数
1. 将两点跳到同一层
2. 让两个点同时往上跳，直到跳到公共祖先的下一层

第一步：基于二进制的思想，x和y之间的层数差距为$depth(x) - depth(y)$，假设y的层数小于x的层数，此时x要往上跳
若要跳k层，那么根据k的二进制表示将k拆分成多个2的幂相加，由于我们已经预处理了$f(i, j)$，所以根据$f(i, j)$的值往上跳即可
当$f(x, k) = f(y, k)$时，即x往上跳k步和y往上跳k步后，位于同一个位置，此时找到了一个公共祖先，但不是最近公共祖先，所以这里要减小k的值，直到$f(x, k) ≠ f(y, k)$，此时才找到了最近公共祖先

规定$depth(0) = 0$，0号点为哨兵，位于第0层，根节点位于第一层
向上跳$2^k$步后，若跳出了这颗树，那么$f(i, k) = 0$

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230808090125.png)

预处理depth和fa的板子：
```cpp
void bfs(int root)
{
	memset(dis, 0x3f, sizeof(dis));
	q[tt ++ ] = root;
	depth[root] = 1, depth[0] = 0;
	while (tt >= hh)
	{
		int x = q[hh ++ ];
		for (int i = h[x]; i != -1; i = ne[i])
		{
			int y = e[i];
			if (depth[y] > depth[x] + 1)
			{
				depth[y] = depth[x] + 1;
				q[tt ++ ] = y;
				fa[y][0] = x;
				for (int k = 1; k <= c; ++ k )
					fa[y][k] = fa[fa[y][k - 1]][k - 1];
			}
		}
	}
}
```
lca板子：
```cpp
int lca(int x, int y)
{
	if (depth[x] < depth[y]) swap(x, y);
	for (int k = c; k >= 0; -- k )
		if (depth[fa[x][k]] >= depth[y])
			x = fa[x][k];
	if (x == y) return y;
	for (int k = c; k >= 0; -- k )
		if (fa[x][k] != fa[y][k])
		{
			x = fa[x][k];
			y = fa[x][k];
		}
	return fa[x][0];
}
```
***
## Tarjan
$O(n + m)$ ，n为节点数量，m为询问次数
tarjan是一种离线处理询问的算法，是向上标记法的优化
在深度优先遍历时，将所有点分成三大类
1. 已经遍历过且已经回溯的点
2. 已经遍历但正在搜索的分支
3. 还未搜索到的点

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230808121012.png)
将已经遍历且回溯的点标记成2，正在遍历的点标记成1，未遍历的点标记成0
对于正在回溯的点，需要处理所有有关该点的询问信息：若询问的另外一个点已经遍历过（回溯完成），那么该点将被分到一个集合中，集合的代表点就是两点的最近公共祖先
比如上图，当前正在遍历x这个点，已经遍历完的点为绿色部分，这些点所属集合的代表点位于红色分支上

模板：
```cpp
// 用dfs维护dis数组
void dfs(int x, int fa)
{
	for (int i = h[x]; i != -1; i = ne[i])
	{
		int y = e[i];
		if (y != fa)
		{
			dis[y] = dis[x] + w[i];
			dfs(y, x);
		}
	}
}

// tarjan处理询问
void tarjan(int x)
{
	st[x] = 1; // 当前正在遍历该点
	for (int i = h[x]; i != -1; i = ne[i])
	{
		int y = e[i];
		if (!st[y]) // 当前为遍历ydian
		{
			tarjan(y);
			p[y] = u;
		}
	}
	for (auto item : query[x]) // 处理有关当前节点的询问
	{
		int y = item.first, id = item.first; // id为该询问的唯一编号
		if (st[y] == 2) // 询问的另一点已经遍历过
		{
			int anc = find(y); // 找到集合的代表点，公共祖先
			res[id] = dis[x] + dis[y] - 2 * dis[anc]; // 根据id将答案存储到数组中
		}
	}
	st[x] = 2; // 当前节点遍历完
}
```
***
### 板子题：1172. 祖孙询问
[1172. 祖孙询问 - AcWing题库](https://www.acwing.com/problem/content/1174/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230808104415.png)

由于树的层数最大有40000层，所以一个节点最多向上跳$log400000$层，大概是一个大于$2^{15}$小于$2^{16}$的数，所以最多跳$2^{16} - 1$层，二进制位取15即可
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 4e4 + 10, M = 2 * N;
int h[N], e[M], ne[M], idx;
int q[N], hh, tt = -1;
int depth[N], fa[N][16]; // 最多跳2^16-1
int n;

void add(int x, int y)
{
    e[idx] = y, ne[idx] = h[x], h[x] = idx ++ ;
}

void bfs(int root)
{
    q[++ tt] = root;
    memset(depth, 0x3f, sizeof(depth));
    depth[0] = 0, depth[root] = 1;
    while (tt >= hh)
    {
        int x = q[hh ++ ];
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (depth[y] > depth[x] + 1)
            {
                depth[y] = depth[x] + 1;
                fa[y][0] = x;
                q[++ tt ] = y;
                for (int k = 1; k <= 15; ++ k )
                    fa[y][k] = fa[fa[y][k - 1]][k - 1];
            }
        }
    }
}

int lca(int x, int y)
{
    if (depth[x] < depth[y]) swap(x, y);
    for (int k = 15; k >= 0; -- k )
        if (depth[fa[x][k]] >= depth[y])
            x = fa[x][k];
    if (x == y) return y;
    for (int k = 15; k >= 0; -- k )
        if (fa[x][k] != fa[y][k])
        {
            x = fa[x][k];
            y = fa[y][k];
        }
    return fa[x][0];
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d", &n);
    int root;
    int a, b;
    for (int i = 0; i < n; ++ i )
    {
        scanf("%d%d", &a, &b);
        if (b == -1) root = a;
        else add(a, b), add(b, a);
    }
    
    bfs(root);
    int m;
    scanf("%d", &m);
    for (int i = 0; i < m; ++ i )
    {
        scanf("%d%d", &a, &b);
        int t = lca(a, b);
        if (t == a) puts("1");
        else if (t == b) puts("2");
        else puts("0");
    }
    return 0;
}
```

***
### lca或tarjan：1171. 距离
[1171. 距离 - AcWing题库](https://www.acwing.com/problem/content/1173/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230808142501.png)

与上题一样，题目要处理很多询问，可以用lca或者离线tarjan解决
树的最短路问题可以从公共祖先的角度考虑，假设x和y的公共祖先为t，$dis(t)$为根节点到t的距离，那么x和y之间的最短距离就是$dis(x) + dis(y) - 2 * dis(t)$

题目没有给定根节点，我们任意指定一个点为根节点即可

lca解法：
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 1e4 + 10, M = 2 * N;
int h[N], e[M], ne[M], w[M], idx;
int depth[N], fa[N][16];
int q[N], hh, tt = -1;
int dis[N];

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

void dfs(int root)
{
    memset(depth, 0x3f, sizeof(depth));
    depth[0] = 0, depth[root] = 1;
    q[++ tt ] = root;
    while (tt >= hh)
    {
        int x = q[hh ++ ];
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (depth[y] > depth[x] + 1)
            {
                depth[y] = depth[x] + 1;
                dis[y] = dis[x] + w[i];
                fa[y][0] = x;
                q[++ tt ] = y;
                for (int k = 1; k <= 15; ++ k )
                    fa[y][k] = fa[fa[y][k - 1]][k - 1];
            }
        }
    }
}

int lca(int x, int y)
{
    if (depth[x] < depth[y]) swap(x, y);
    for (int k = 15; k >= 0; -- k )
        if (depth[fa[x][k]] >= depth[y])
            x = fa[x][k];
    if (x == y) return y;
    for (int k = 15; k >= 0; -- k )
        if (fa[x][k] != fa[y][k])
        {
            x = fa[x][k];
            y = fa[y][k];
        }
    return fa[x][0];
}

int main()
{
    memset(h, -1, sizeof(h));
    int n, m;
    scanf("%d%d", &n, &m);
    int x, y, d;
    for (int i = 0; i < n - 1; ++ i )
    {
        scanf("%d%d%d", &x, &y, &d);
        add(x, y, d), add(y, x, d);
    }
    dfs(1);
    
    for (int i = 0; i < m; ++ i )
    {
        scanf("%d%d", &x, &y);
        int t = lca(x, y);
        printf("%d\n", dis[x] + dis[y] - 2 * dis[t]);
    }
    return 0;
}
```

tarjan解法：
```cpp
#include <iostream>
#include <cstring>
#include <vector>
using namespace std;

const int N = 1e4 + 10, M = 2 * N;
typedef pair<int, int> PII;
vector<PII> query[N];
int h[N], e[M], ne[M], w[M], idx;
int res[M], dis[N], st[N], p[N];
int n, m;

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

void dfs(int x, int fa)
{
    for (int i = h[x]; i != -1; i = ne[i])
    {
        int y = e[i];
        if (y != fa)
        {
            dis[y] = dis[x] + w[i];
            dfs(y, x);
        }
    }
}

int find(int x)
{
    if (p[x] != x) p[x] = find(p[x]);
    return p[x];
}

void tarjan(int x)
{
    st[x] = 1;
    for (int i = h[x]; i != -1; i = ne[i])
    {
        int y = e[i];
        if (!st[y])
        {
            tarjan(y);
            p[y] = x;
        }
    }
    
    for (auto item : query[x])
    {
        int y = item.first, id = item.second;
        if (st[y] == 2)
        {
            int anc = find(y);
            res[id] = dis[x] + dis[y] - 2 * dis[anc];
        }
    }
    st[x] = 2;
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d", &n, &m);
    int x, y, d;
    for (int i = 0; i < n - 1; ++ i )
    {
        scanf("%d%d%d", &x, &y, &d);
        add(x, y, d), add(y, x, d);
    }
    for (int i = 0; i < m; ++ i )
    {
        scanf("%d%d", &x, &y);
        query[x].push_back({y, i}); // 将查询的另一个点与查询编号保存
        query[y].push_back({x, i});
    }
    
    dfs(1, -1);
    for (int i = 1; i <= n; ++ i ) p[i] = i;
    tarjan(1);
    for (int i = 0; i < m; ++ i ) printf("%d\n", res[i]);
    
    return 0;
}
```


debug：维护的query信息要不同地插入两次
```cpp
query[x].push_back({y, i});
query[y].push_back({x, i});
```
因为当前询问有关x的信息时，y可能没有遍历完，但是询问y有关的信息时，x是遍历完的

***
### 356. 次小生成树
[356. 次小生成树 - AcWing题库](https://www.acwing.com/problem/content/description/358/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230808190720.png)

$d1(i, j)$表示从i往上跳$2^j$层后，路径上的最大边权
$d2(i, j)$表示从i往上跳$2^j$层后，路径上的次大边权
跳$2^j$步是一个类似分治的过程，分成两部分跳，这两部分依旧能分成两部分跳
路径上的最大值为每一段的最大值取max，次大值为所有子路径的最大值和次大值中的第二大，每次遍历子线段时维护信息即可
```cpp
if (d > d1) d2 = d1, d1 = d; 
else if (d != d1 && d > d2) d2 = d;
```

```cpp
#include <iostream>
#include <cstring>
#include <algorithm>
using namespace std;

typedef long long LL;
const int N = 1e5 + 10, M = 3e5 + 10, INF = 0x3f3f3f3f;
struct Edge
{
    int x, y, w;
    bool f;
    bool operator<(const Edge& e) const 
    {
        return w < e.w;
    }
}edges[M];

int h[N], e[M], ne[M], w[M], idx;
int p[N], depth[N], fa[N][17];
int q[N], hh, tt = -1;
int d1[N][17], d2[N][17];
int d[M];
int n, m;

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

int find(int x)
{
    if (x != p[x]) p[x] = find(p[x]);
    return p[x];
}

LL kruskal()
{
    LL res = 0;
    sort(edges, edges + m);
    for (int i = 1; i <= n; ++ i ) p[i] = i;
    for (int i = 0; i < m; ++ i )
    {
        auto t = edges[i];
        int x = t.x, y = t.y, w = t.w;
        int px = find(x), py = find(y);
        if (px != py)
        {
            res += w;
            p[px] = py;
            edges[i].f = true;
            add(x, y, w), add(y, x, w);
        }
    }
    return res;
}

void bfs()
{
    q[++ tt ] = 1;
    memset(depth, 0x3f, sizeof(depth));
    depth[0] = 0, depth[1] = 1;
    while (tt >= hh)
    {
        int x = q[hh ++ ];
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (depth[y] > depth[x] + 1)
            {
                depth[y] = depth[x] + 1;
                fa[y][0] = x;
                d1[y][0] = w[i], d2[y][0] = -INF;
                q[++ tt ] = y;
                for (int k = 1; k <= 16; ++ k )
                {
                    int mid = fa[y][k - 1];
                    fa[y][k] = fa[mid][k - 1];
                    d1[y][k] = d2[y][k] = -INF;
                    int a[4] = { d1[mid][k - 1], d2[mid][k - 1], d1[y][k - 1], d2[y][k - 1] };
                    for (int i = 0; i < 4; ++ i )
                    {
                        if (a[i] > d1[y][k]) d2[y][k] = d1[y][k], d1[y][k] = a[i];
                        else if (a[i] != d1[y][k] && a[i] > d2[y][k]) d2[y][k] = a[i];
                    }
                }
            }
        }
    }
}


int lca(int x, int y, int w)
{
    int cnt = 0;
    if (depth[x] < depth[y]) swap(x, y);
    for (int k = 16; k >= 0; -- k )
        if (depth[fa[x][k]] >= depth[y])
        {
            d[cnt ++ ] = d1[x][k];
            d[cnt ++ ] = d2[x][k];
            x = fa[x][k];
        }
    if (x != y)
    {
        for (int k = 16; k >= 0; -- k )
        {
            if (fa[x][k] != fa[y][k])
            {
                d[cnt ++ ] = d1[x][k], d[cnt ++ ] = d2[x][k];
                d[cnt ++ ] = d1[y][k], d[cnt ++ ] = d2[y][k];
                x = fa[x][k], y = fa[y][k];
            }
            d[cnt ++ ] = d1[x][0], d[cnt ++ ] = d2[x][0];
            d[cnt ++ ] = d1[y][0], d[cnt ++ ] = d2[y][0];
        }
    }
    int dmax1 = -INF, dmax2 = -INF;
    for (int i = 0; i < cnt; ++ i )
    {
        if (d[i] > dmax1) dmax2 = dmax1, dmax1 = d[i];
        else if (d[i] != dmax1 && d[i] > dmax2) dmax2 = d[i];
    }
    
    if (w > dmax1) return w - dmax1;
    return w - dmax2;
}


int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d", &n, &m);
    for (int i = 0; i < m; ++ i )
        scanf("%d%d%d", &edges[i].x, &edges[i].y, &edges[i].w);
        
    LL sum = kruskal(); // 最小生成树的权值
    bfs();
    
    LL res = 1e19;
    for (int i = 0; i < m; ++ i )
        if (!edges[i].f)
            res = min(res, sum + lca(edges[i].x, edges[i].y, edges[i].w));
    
    printf("%lld\n", res);
    return 0;
}
```
***
### 352. 闇の連鎖
[352. 闇の連鎖 - AcWing题库](https://www.acwing.com/problem/content/description/354/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230808192234.png)

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230808192805.png)

树上差分，将x到y的最短路径上所有的边加上c，若p为x和y的公共祖先，那么
`d(x) += c, d(y) += c, d(p) -= 2c`
如何计算某条边的权值？以这条边的子节点为根的子树中，所有边的权值相加为这条边的权值

在一颗树中，删除任意一条边，那么这颗树将被切成两个连通块
若向树中再添加一条边，那么这条非树边和树边就一定构成环，要向将此时的“树”切成两个连通块，就要删除环中的任意一条树边与这条非树边
题目限制只能先切树边，再切非树边，一共两次，两次过后还没有切成两个连通块，说明这个方案行不通
当切除树边，不用再切除非树边就得到两个连通块时，由于题目限制，还需要切除一条非树边，假设非树边有m条，那么此时可以选择m条边中的任意一条切除，此时的方案数为m
若切除树边后，还要再切除一条非树边，才能得到两个连通块时，此时的方案数为1，只能切除这条环中的非树边
若切除树边后，还要再切除大于一条的非树边，此时无法再切除，方案数为0

现在的问题是，如何知道切除某条树边后，还需要再切除几条非树边？
假设现在已经用树边建立了一棵树，此时再添加非树边将构成环，将环中的所有树边权值加1，假设初始权值为0，此时可以使用树上差分
然后遍历所有的树边，根据边权计算方案数

```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 1e5 + 10, M = 2e5 + 10;
int h[N], e[M], ne[M], idx;
int depth[N], fa[N][17];
int q[N], hh, tt = -1;
int d[N];
int n, m, ans;

void add(int x ,int y)
{
    e[idx] = y, ne[idx] = h[x], h[x] = idx ++ ;
}

void bfs()
{
    memset(depth, 0x3f, sizeof(depth));
    depth[0] = 0, depth[1] = 1;
    q[++ tt ] = 1;
    while (tt >= hh)
    {
        int x = q[hh ++ ];
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (depth[y] > depth[x] + 1)
            {
                depth[y] = depth[x] + 1;
                fa[y][0] = x;
                q[++ tt ] = y;
                for (int k = 1; k <= 16; ++ k )
                    fa[y][k] = fa[fa[y][k - 1]][k - 1];
            }
        }
    }
}

int lca(int x, int y)
{
    if (depth[x] < depth[y]) swap(x, y);
    for (int k = 16; k >= 0; -- k )
        if (depth[fa[x][k]] >= depth[y])
            x = fa[x][k];
    if (x == y) return x;
    for (int k = 16; k >= 0; -- k )
        if (fa[x][k] != fa[y][k])
        {
            x = fa[x][k];
            y = fa[y][k];
        }
    return fa[x][0];
}

int dfs(int x, int f)
{
    int res = d[x];
    for (int i = h[x]; i != -1; i = ne[i])
    {
        int y = e[i];
        if (f != y)
        {
            int t = dfs(y, x);
            if (t == 0) ans += m;
            else if (t == 1) ans ++;
            res += t;
        }
    }
    return res;
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d", &n, &m);
    int x, y;
    for (int i = 0; i < n - 1; ++ i )
    {        
        scanf("%d%d", &x, &y);
        add(x, y), add(y, x);
    }
    
    bfs();
    int res = 0;
    for (int i = 0; i < m; ++ i)
    {
        scanf("%d%d", &x, &y);
        int p = lca(x, y);
        d[x] ++, d[y] ++, d[p] -= 2;
    }
    dfs(1, -1);
    printf("%d\n", ans);
    return 0;
}
```
