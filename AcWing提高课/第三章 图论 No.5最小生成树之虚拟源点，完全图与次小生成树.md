```toc
```

### 虚拟源点：1146. 新的开始
[1146. 新的开始 - AcWing题库](https://www.acwing.com/problem/content/description/1148/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230806091030.png)

与一般的最小生成树问题不同，本题需要在建立电站的电井之间建立电网，在两个电站之间建立电网需要花费金额，可以看成一条具有权值的边
但是建立电网的前提是：其中一个电井需要建立电站，建立电站也需要费用
已经建立电站的两个电井之间无需建立电网，即一张电网中只需要存在一个建立电站的电井
可以将建立电站也看成具有权值的边，设置虚拟源点，在第i个电井建立电站可以转换成虚拟源点与i点之间的边，权值为建立电站的费用
此时跑个最小生成树即可

为什么不能直接跑最小生成树，再选择某个点建立一个最便宜的电站？只建立一个电站虽然能保证所有电井有电，但是两个电井建立电网的费用可能高于直接建立电站的费用
所以可能会建立多个电站，即最小生成“森林”，设置虚拟选点就是将每个森林连接，即最小生成树，此时需要跑最小生成树的算法即可
```cpp
// 跑一遍最小生成树，记录其中点的最小值
#include <iostream>
#include <cstring>
using namespace std;

const int N = 310;
int g[N][N], dis[N];
bool st[N];
int n;

int prim()
{
    memset(dis, 0x3f, sizeof(dis));
    int res = 0;
    for (int i = 0; i <= n; ++ i )
    {
        int x = -1;
        for (int j = 0; j <= n; ++ j )
            if (!st[j] && (x == -1 || dis[x] > dis[j])) x = j;
        st[x] = true;
        if (i) res += dis[x];
        for (int y = 0; y <= n; ++ y )
            dis[y] = min(dis[y], g[x][y]);
    }
    return res;
}

int main()
{
    scanf("%d", &n);
    for (int i = 1; i <= n; ++ i ) 
    {
        scanf("%d", &g[0][i]);
        g[i][0] = g[0][i];
    }
    for (int i = 1; i <= n; ++i )
        for (int j = 1; j <= n; ++ j )
            scanf("%d", &g[i][j]);
            
    printf("%d\n", prim());
    
    return 0;
}
```
***
### 贪心或kruskal性质：1145. 北极通讯网络
[1145. 北极通讯网络 - AcWing题库](https://www.acwing.com/problem/content/1147/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230806092012.png)

三种解法，第一种，一眼想到的就是贪心：
跑个最小生成树，升序记录所有边权
若有k个卫星，选择第k大的边即可，因为k个卫星使得k个村庄可以直接通信
k-1条最大边连接的村庄用卫星，其他用发电设备通信，此时d为最大的边权
```cpp
#include <iostream>
#include <algorithm>
#include <cmath>
using namespace std;

#define x first
#define y second
typedef pair<int, int> PII;
const int N = 510, INF = 0x3f3f3f3f;
double g[N][N], res[N];
double dis[N];
PII a[N];
bool st[N];
int n, k;

double get_dis(PII a, PII b)
{
    int x = a.x - b.x, y = a.y - b.y;
    return sqrt(x * x + y * y);
}

double prim()
{
    for (int i = 1; i <= n; ++ i ) dis[i] = INF;
    for (int i = 0; i < n; ++ i )
    {
        int x = -1;
        for (int j = 1; j <= n; ++ j )
            if (!st[j] && (x == -1 || dis[x] > dis[j])) x = j;
        st[x] = true;
        if (i) res[i] = dis[x];
        
        for (int y = 1; y <= n; ++ y )
            dis[y] = min(dis[y], g[x][y]);
    }
    sort(res + 1, res + n);
    return res[n - k]; // 1 ~ n-1 为最小生成树的边权升序排序
}

int main()
{
    scanf("%d%d", &n, &k);
    for (int i = 1; i <= n; ++ i )
        scanf("%d%d", &a[i].x, &a[i].y);
    
    if (k >= n) puts("0.00");
    else
    {
        for (int i = 1; i <= n; ++ i )
            for (int j = i + 1; j <= n; ++ j )
                g[i][j] = g[j][i] = get_dis(a[i], a[j]);
                
        printf("%.2lf\n", prim());
    }
    
    return 0;
}
```
***
kurskal的性质：
每次kruskal选择当前最小边更新时，本质是在建立连通块，初始每个点各自为连通块，数量为n，每次更新连通块的数量-1，更新n-1次选择了n-1条边后，连通块的数量为1，此时最小生成树构建完成

转换题意，找到一个最小d值，删除所有大于等于d的边后，剩下的连通块数量不超过k，两个连通块中的村庄用卫星通信通信
利用kruskal的性质，更新t次后，$n-t = k$时，表示已经建立了k个连通块，这些连通块中的最大边权为答案
```cpp
// 跑个最小生成树，升序记录所有边权
// 若有k个卫星，选择第k大的边即可，因为k个卫星使得k个村庄可以直接通信
// k-1个最大边连接的存在用卫星，其他用发电设备通信
#include <iostream>
#include <algorithm>
#include <cmath>
using namespace std;

typedef pair<int, int> PII;
const int N = 510, M = N * N / 2;
int p[N];
PII a[N];
bool st[N];
int n, k, cnt;

struct Edge
{
    int x, y;
    double w;
    bool operator<(const Edge& e)
    {
        return w < e.w;
    }
}edges[M];

double get_dis(PII a, PII b)
{
    int x = a.first - b.first, y = a.second - b.second;
    return sqrt(x * x + y * y);
}

int find(int x)
{
    if (x != p[x]) p[x] = find(p[x]);
    return p[x];
}

double kruskal()
{
    double res;
    int u = 0;
    sort(edges, edges + cnt);
    for (int i = 1; i <= n; ++ i) p[i] = i;
    for (int i = 0; i < cnt; ++ i )
    {
        if (n - u == k) break;
        auto t = edges[i];
        int x = t.x, y = t.y;
        double w = t.w;
        x = find(x), y = find(y);
        if (x != y)
        {
            u ++ ;
            res = w;
            p[x] = y;
        }
    }
    return res;
}

int main()
{
    scanf("%d%d", &n, &k);
    for (int i = 1; i <= n; ++ i )
        scanf("%d%d", &a[i].first, &a[i].second);
    
    if (k >= n) puts("0.00");
    
    else
    {
        for (int i = 1; i <= n; ++ i )
            for (int j = i + 1; j <= n; ++ j )
                edges[cnt ++ ] = { i, j, get_dis(a[i], a[j])};
                
        printf("%.2lf\n", kruskal());
    }
    
    return 0;
}
```
debug：Edge中的w要用double存
***
###  最小生成树与完全图：346. 走廊泼水节
[346. 走廊泼水节 - AcWing题库](https://www.acwing.com/problem/content/description/348/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230806154437.png)

如何合并完全图？开始时图中的每个点各自为一个集合，用集合合并的方式，保证合并后的集合为一个完全图
若集合a有x个点，b有y个点，要使得两集合合并后是个完全图（合并前两集合分别是完全图），就要将属于不同集合的点之间建一条边，总共需要建立xy条边

现在的问题是，将两个集合合并成完全图的边权为多大才能满足题意？
将树的每条边从小到大排序，每次合并当前枚举的边连接的两个集合，保证合并后的集合是一个完全图
由于要保证完全图的最小生成树唯一，所以要保证用来建立完全图的边权大于原生成树的边权

即当前枚举第i条边，这条边连接两个点$x_i, y_i$，将$x_i, y_i$所属的两个集合合并成完全图，需要在两个集合中的每个点之间建立一条边，并且该边的权值需要大于$w_i$

```cpp
#include <iostream>
#include <algorithm>
using namespace std;

const int N = 6010;
struct Edge
{
    int x, y, w;
    bool operator<(const Edge& e) const 
    {
        return w < e.w;
    }
}edges[N];
int p[N], sz[N];

int find(int x)
{
    if (x != p[x]) p[x] = find(p[x]);
    return p[x];
}

int main()
{
    int T;
    scanf("%d", &T);
    while (T -- )
    {
        int n;
        scanf("%d", &n);
        for (int i = 0; i < n - 1; ++ i )
            scanf("%d%d%d", &edges[i].x, &edges[i].y, &edges[i].w);
        
        sort(edges, edges + n - 1);
        long long res = 0;
        for (int i = 1; i <= n; ++ i ) p[i] = i, sz[i] = 1;
        for (int i = 0; i < n - 1; ++ i )
        {
            auto t = edges[i];
            int x = find(t.x), y = find(t.y), w = t.w;
            if (x != y)
            {
                int n1 = sz[x], n2 = sz[y];
                p[x] = y;
                sz[y] += sz[x];
                res += (n1 * n2 - 1) * (w + 1);
            }
        }
        printf("%lld\n", res);
    }
    
    return 0;
}
```
debug：edges忘记了排序
***
### 次小生成树：1148. 秘密的牛奶运输
[1148. 秘密的牛奶运输 - AcWing题库](https://www.acwing.com/problem/content/1150/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230806201724.png)

次小生成树：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230806163725.png)

先求出最小生成树，删除最小生成树中的一条边
重复n-1次，得到的最小生成树就是次小生成树
注意：只能求出非严格次小生成树，即次小生成树的权值和可能等于最小生成树
时间复杂度$O(mlngm + nm)$
***
先求最小生成树，枚举不在树中的边，同时删除最小生成树（构成环）中的最大边，使得最终得到的图仍然是一颗树，次小生成树一定在这些树中

在生成树中任意添加一条边，必定构成环，此时需要在这个环路中删除一条边，使得该图再次成为一颗树，由于要保证权值和最小，所以要删除一条最大边
每次枚举非树边时，需要判断该边权值是否大于环的最大边，若大于则删除环中的最大值，此时能保证次小生成的权值和严格大于最小生成树
不过这种情况有一个特例，若非树边的权值等于最大边，当环中所有边的权值都等于最大边时，不更新次小生成树没有问题。若环中存在边权小于最大边的权值呢？此时可以删除这条次大的边，更新次小生成树。所以只通过判断非树边是否大于最大值将漏掉一些情况，导致少枚举次小生成树，最终导致答案的错误
因此，除了要维护两点间的最大边，还需要维护两点间的次大边，并且次大边需要严格小于最大边

若要生成非严格的次小生成树，只需要修改判断条件，在非树边的权值大于等于环的最大值时更新。多了权值相等的情况，所以删除的最大边可能和加入非树边权值相等，那么生成的次小生成树权值和就是相同的
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230806163108.png)

先预处理树中两点间的最大边，从根节点出发做一个dfs，每次都要$O(n)$，时间复杂度是$O(n^2)$
总的次小生成树的时间复杂度为$O(mlngm + n^2 + m)$

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230806164329.png)
由于第二种求次小生成树的方式既可以求最小生成树也能求次小生成树，所以这里实现第二种
用两个二维数组表示最小生成树中任意两点的最大边与次大边
```cpp
#include <iostream>
#include <cstring>
#include <algorithm>
using namespace std;

typedef long long LL;
const int N = 510, M = 1e4 + 10;
struct Edge
{
    int x, y, w;
    bool f;
    bool operator<(const Edge& e) const 
    {
        return w < e.w;
    }
}edges[M];

int p[N];
int h[N], e[M], ne[M], w[M], idx;
int dmax1[N][N], dmax2[N][N];
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

// 走到x的最大边与次大边
void dfs(int x, int f, int d1, int d2, int xmax1[], int xmax2[])
{
    xmax1[x] = d1, xmax2[x] = d2;
    for (int i = h[x]; i != -1; i = ne[i])
    {
        int y = e[i];
        if (y != f)
        {
            int t1 = d1, t2 = d2;
            if (w[i] > t1) t2 = t1, t1 = w[i];
            else if (w[i] < t1 && w[i] > t2) t2 = w[i];
            dfs(y, x, t1, t2, xmax1, xmax2);
        }
    }
}

LL kruskal()
{
    LL sum = 0;
    sort(edges, edges + m);
    for (int i = 1; i <= n; ++ i ) p[i] = i;
    for (int i = 0; i < m; ++ i )
    {
        auto t = edges[i];
        int x = t.x, y = t.y, w = t.w;
        int px = find(t.x), py = find(t.y);
        if (px != py)
        {
            sum += w;
            p[px] = py;
            add(x, y, w), add(y, x, w); // 存储最小生成树
            edges[i].f = true; // 树边
        }
    }
    return sum;
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d", &n, &m);
    for (int i = 0; i < m; ++ i ) scanf("%d%d%d", &edges[i].x, &edges[i].y, &edges[i].w);
    LL sum = kruskal();
    for (int i = 1; i <= n; ++ i ) dfs(i, -1, -1e9, -1e9, dmax1[i], dmax2[i]);
    LL res = 1e19;
    for (int i = 0; i < m; ++ i )
    {
        if (!edges[i].f)
        {
            int x = edges[i].x, y = edges[i].y, w = edges[i].w;
            if (w > dmax1[x][y]) res = min(res, sum - dmax1[x][y] + w);
            else if (w > dmax2[x][y]) res = min(res, sum - dmax2[x][y] + w);
        }
    }
    printf("%lld\n", res);
    return 0;
}
```
debug：构建最小生成树的同时，用邻接表存储最小生成树
```cpp
auto t = edges[i];
int x = find(t.x), y = find(t.y), w = t.w;
if (x != y)
{
	sum += w;
	p[x] = y;
	add(x, y, w), add(y, x, w); // 存储最小生成树
	edges[i].f = true; // 树边
}
```
若这样写，构建的最小生成树是正确的，但是存储的最小生成树确实不正确的
因为x和y是边的两点所属的集合，不一定是边的两点，所以此时add无法正确保存最小生成树