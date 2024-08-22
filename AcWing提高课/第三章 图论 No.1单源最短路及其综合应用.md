```toc
```
> 做乘法的最短路时，若权值>=0，只能用spfa来做，相等于加法中的负权边
### 1129. 热浪
[1129. 热浪 - AcWing题库](https://www.acwing.com/problem/content/1131/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230729151337.png)

单源最短路，稀疏图，用堆优化Dijkstra即可，就是板子套了个背景
```cpp
// 稀疏图，用堆优化Dijkstra存储
#include <iostream>
#include <cstring>
#include <queue>
using namespace std;

const int N = 2510, M = 2 * 6210;
typedef pair<int, int> PII;
priority_queue<PII, vector<PII>, greater<PII>> q;
int h[N], e[M], ne[M], w[M], idx = 1;
int n, m, start, ed;
int dis[N];
bool st[N];

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

void dijkstra()
{
    memset(dis, 0x3f, sizeof(dis));
    dis[start] = 0;
    q.push({0, start});
    while (q.size())
    {
        auto t = q.top(); q.pop();
        int x = t.second, d = t.first;
        if (st[x]) continue;
        st[x] = true;
        
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (dis[y] > d + w[i])
            {
                dis[y] = d + w[i];
                q.push({dis[y], y});
            }
        }
    }
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d%d%d", &n, &m, &start, &ed);
    int x, y, d;
    for (int i = 1; i <= m; ++ i )
    {
        scanf("%d%d%d", &x, &y, &d);
        add(x, y, d), add(y, x, d);
    }
    
    dijkstra();
    printf("%d\n", dis[ed]);
    return 0;
}
```
debug：由于是无向图，边的数量要开两倍。但是`w[N]`没改，debug了很久
所以`e[M], ne[M], w[M]`，只有`h[N]`，其他的数组存储的都是边
***
### 1128. 信使
[1128. 信使 - AcWing题库](https://www.acwing.com/problem/content/1130/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230729151159.png)

单源最短路，稀疏图，用堆优化Dijkstra即可，最后统计所有dis的最大值并返回即可
```cpp
// 单源最短路，最后将第1个点到其他点的距离累加即可
// 稀疏图，依然是堆优化的dijkstra
#include <iostream>
#include <cstring>
#include <queue>
using namespace std;

typedef pair<int, int> PII;
const int N = 110, M = 410;
int h[N], e[M], ne[M], w[M], idx = 1;
priority_queue<PII, vector<PII>, greater<PII>> q;
int n, m;
bool st[N]; int dis[N];

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

void dijkstra()
{
    memset(dis, 0x3f, sizeof(dis));
    dis[1] = 0;
    q.push({0, 1});
    while (q.size())
    {
        auto t = q.top(); q.pop();
        int x = t.second, d = t.first;
        
        if (st[x]) continue;
        st[x] = true;
        for (int i = h[x]; i != -1; i = ne[i]) 
        {
            int y = e[i];
            if (dis[y] > d + w[i])
            {
                dis[y] = d + w[i];
                q.push({dis[y], y});
            }
        }
    }
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d", &n, &m);
    int x, y, d;
    for (int i = 1; i <= m; ++ i )
    {
        scanf("%d%d%d", &x, &y, &d);
        add(x, y, d), add(y, x, d);
    }
    
    dijkstra();
    
    int res = 0;
    bool flag = true;
    for (int i = 2; i <= n; ++ i )
    {
        if (dis[i] == 0x3f3f3f3f) 
        {
            flag = false;
            break;
        }
        res = max(res, dis[i]);
    }
    if (flag) printf("%d\n", res);
    else puts("-1");

    return 0;
}
```
debug：`%d`打成了`5d`，乐，竟然能编译通过
判断某个点的最短距离是否已经确定，打成了`if(dis[x]); dis[x] = true`，这也debug了半天
***
### 1127. 香甜的黄油
[1127. 香甜的黄油 - AcWing题库](https://www.acwing.com/problem/content/description/1129/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230729170239.png)

```cpp
// 多源汇最短路，由于flody可能超时，所以用单源最短路求解
// 这里用spfa
#include <iostream>
#include <cstring>
using namespace std;

const int N = 810, M = 3000;
int h[N], e[M], ne[M], w[M], idx = 1;
int dis[N]; bool st[N];
int s, n, m;
int q[N];
int id[N];

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

void spfa(int k)
{
    memset(dis, 0x3f, sizeof(dis));
    int hh = 0, tt = 1;
    dis[k] = 0;
    q[tt ++ ] = k, st[k] = true;
    while (tt != hh)
    {
        int x = q[hh ++ ];
        if (hh == N) hh = 0;
        st[x] = false;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (dis[y] > dis[x] + w[i])
            {
                dis[y] = dis[x] + w[i];
                if (!st[y]) 
                {
                    q[tt ++ ] = y, st[y] = true;
                    if (tt == N) tt = 0;
                }
            }
        }
    }
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d%d", &s, &n, &m);
    for (int i = 1; i <= s; ++ i ) scanf("%d", &id[i]);
    
    int x, y, d;
    for (int i = 1; i <= m; ++ i )
    {
        scanf("%d%d%d", &x, &y, &d);
        add(x, y, d), add(y, x, d);
    }
    
    int res = 0x3f3f3f3f;
    for (int i = 1; i <= n; ++ i )
    {
        spfa(i);
        int sum = 0; bool flag = true;
        for (int j = 1; j <= s; ++ j ) 
        {
            if (dis[id[j]] == 0x3f3f3f3f)
            {
                flag = false;
                break;
            }
            sum += dis[id[j]];
        }
        if (flag) res = min(res, sum);
    }
    
    printf("%d\n", res);
    return 0;
}
```
用自己写的循环队列代替queue，时间大概快了一倍
需要注意的是，循环队列的hh和tt从0开始使用，并且都是后置++，++后还要维护循环性质

以下是用queue实现的版本
```cpp
// 多源汇最短路，由于flody可能超时，所以用单源最短路求解
// 这里用spfa
#include <iostream>
#include <queue>
#include <cstring>
using namespace std;

typedef long long LL;
const int N = 810, M = 1500 * 2;
int h[N], e[M], ne[M], w[M], idx = 1;
int dis[N]; bool st[N];
int s, n, m;
queue<int> q;
int id[N];

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

void spfa(int k)
{
    memset(dis, 0x3f, sizeof(dis));
    dis[k] = 0;
    q.push(k), st[k] = true;
    while (q.size())
    {
        int x = q.front(); q.pop();
        st[x] = false;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (dis[y] > dis[x] + w[i])
            {
                dis[y] = dis[x] + w[i];
                if (!st[y]) q.push(y), st[y] = true;
            }
        }
    }
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d%d", &s, &n, &m);
    int t;
    for (int i = 1; i <= s; ++ i ) scanf("%d", &id[i]);
    
    int x, y, d;
    for (int i = 1; i <= m; ++ i )
    {
        scanf("%d%d%d", &x, &y, &d);
        add(x, y, d), add(y, x, d);
    }
    
    LL res = 1e18;
    for (int i = 1; i <= n; ++ i )
    {
        spfa(i);
        LL sum = 0; bool flag = true;
        for (int i = 1; i <= s; ++ i )
        {
            if (dis[id[i]] == 0x3f3f3f3f) 
            {
                flag = false;
                break;
            }
            sum += dis[id[i]];
        }
        if (flag) res = min(res, sum);
    }
    
    printf("%lld\n", res);
    return 0;
}
```
***
### 1126. 最小花费
[1126. 最小花费 - AcWing题库](https://www.acwing.com/problem/content/1128/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230729183711.png)

转账金额A，当手续费的比例为z时，只能转账A(1 - z)
若每条边的权值为实际转账的金额比例，那么从起点到终点，这个比例越大，花费的初始金额就越少，所以这题需要找一条从起点到终点比例最大的路径
即从起点到终点经过的所有边的比例相乘最大，由于比例的大小为$0 < z <= 1$，所以这里初始化dis数组以及邻接矩阵为全0
接着用spfa或者dijkstra求“最短路”即可
```cpp
// 同样的单源最短路，spfa解决

#include <iostream>
using namespace std;

const int N = 2010;
double g[N][N];
int n, m, start, ed;
double dis[N]; bool st[N];
int q[N], tt, hh;

void spfa()
{
    dis[start] = 1.0;
    q[tt ++ ] = start, st[start] = true;
    while (tt != hh)
    {
        int x = q[hh ++ ];
        if (hh == N) hh = 0;
        st[x] = false;
        for (int j = 1; j <= n; ++ j )
        {
            if (dis[j] < dis[x] * g[x][j])
            {
                dis[j] = dis[x] * g[x][j];
                if (!st[j]) 
                {
                    q[tt ++ ] = j, st[j] = true;
                    if (tt == N) tt = 0;
                }
            }
        }
    }
}

int main()
{
    scanf("%d%d", &n, &m);
    int x, y, z;
    for (int i = 1; i <= m; ++ i )
    {
        scanf("%d%d%d", &x, &y, &z);
        g[x][y] = g[y][x] = 1 - (z / 100.0);
    }
    scanf("%d%d", &start, &ed);
    
    spfa();
    
    printf("%.8lf\n", 100 / dis[ed]);
    return 0;
}
```
***
### 920. 最优乘车
[920. 最优乘车 - AcWing题库](https://www.acwing.com/problem/content/922/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230730145324.png)

没之前的题目裸，需要一些思考
虽然题目没有给定边的权重，但是给定了公交路线，可知公交路线$a$中，$a_i$到$a_j$的权重为1，表示从i地到j地需要的乘车次数，当如$a_i$在$a$中的下标要出现在$a_j$之前，这是一张有向图
所以，根据题目给定的公交路线，找出图中所有的有向边，最后用单源最短路就行了
由于边的权值为1，甚至用bfs也行
```cpp
// 一条路线之间的点可达，存在有向边，线路中有n个点，存在Cn2条边
// 边的权值为1，表示需要乘车的次数，由于权值为1，所以可以用bfs
#include <iostream>
#include <sstream>
#include <string>
#include <cstring>
using namespace std;

const int N = 510, M = 11;;
int a[N], g[N][N], q[N];
int dis[N];
int n, m, tt, hh;

int bfs()
{
    dis[1] = 0;
    q[tt ++ ] = 1;
    while (tt != hh)
    {
        int x = q[hh ++ ];
        if (hh == N) hh = 0;
        
        for (int y = 1; y <= n; ++ y )
        {
            if (y != x && !dis[y] && g[x][y]) 
            {
                dis[y] = dis[x] + 1;
                if (y == n) return dis[y];
                q[tt ++ ] = y;
            }
        }
    }
    return 0;
}

int main()
{
    scanf("%d%d", &m, &n);
    string line;
    getline(cin, line);
    for (int i = 0; i < m; ++ i )
    {
        getline(cin, line);
        stringstream s(line);
        int idx = 0, p;
        while (s >> p) a[idx ++ ] = p;
        
        for (int j = 0; j < idx; ++ j )
            for (int k = j + 1; k < idx; ++ k )
                g[a[j]][a[k]] = 1;
    }
    
    int t = bfs();
    if (t == 0) puts("NO");
    else printf("%d\n", t - 1);
    
    return 0;
}
```
***
### 903. 昂贵的聘礼
[903. 昂贵的聘礼 - AcWing题库](https://www.acwing.com/problem/content/905/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230731195301.png)

以下是我的思路，我只是记录一下不建议看
要获得某个物品，有两种方式：1. 直接购买 2. 在拥有替代品的前提下，以优惠价格购买
此外无法找出其他方法，需要考虑的是：以优惠价格购买时，拥有替代品的代价是什么？
若从正面考虑，假设酋长的10000金币是一个物品，现可以直接购买或者使用替代品。若使用替代品以更低的价格购买，那么购买替代品的价格是多少？同样，购买替代品也有两种方式，是直接购买省钱还是用替代品的替代品购买省钱。这是一个无止尽的过程，什么时候会停止？物品没有替代品时停止，这就意味着我们要递归到最底部后，在向上回溯的过程中，才能计算两种方式中哪种最省钱
这是直接的思路，可能能做出来吧，没试过
***
正解：
将酋长的10000金币看成物品，并且是有向图的终点。物品之间有什么关系？从一个点走到另一个点，前提是当前点是下一点的替代品，那么从当前点到下一点的代价就是用当前物品购买下一物品的优惠价格
所以边的权重为优惠价格，当前除了以优惠价格购买，还能直接购买。直接购买时，当前点的含义不再是一个物品，所以这里的点是一个虚拟点，到其他点的权重为直接购买的价格

若当前已经购买了物品，走向下一物品时，直接购买的价格肯定比使用替代品高，所以不用设置已经购买物品时，再直接购买其他物品的边。这样的边没有意义
由于存在一些物品无法使用替代品购买，所以只要设置手上没有物品时，直接购买物品的边
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230731201324.png)
虚拟点为上图的s
起点为s，终点为1号物品（*酋长的10000金币*），跑一个最短路即可
为什么起点是s？存在一些只能直接购买的物品

最后考虑等级限制，等级差距不超过m的人可以交换物品，由于题目数据量小，可以直接暴力枚举。假设酋长的等级为a，那么需要枚举的区间为$[a-m, a+m]$，枚举这个区间中长度为m的区间，每次枚举跑一次最短路即可
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 110;
int dis[N], g[N][N];
int level[N];
bool st[N];
int n, m;

int dijkstra(int l, int r)
{
    memset(dis, 0x3f, sizeof(dis));
    memset(st, false, sizeof(st));
    dis[0] = 0;
    for (int i = 0; i <= n; ++ i )
    {
        int t = -1;
        for (int j = 0; j <= n; ++ j)
            if (!st[j] && (t == -1 || dis[j] < dis[t])) t = j;
        st[t] = true;
        for (int j = 1; j <= n; ++ j )
            if (l <= level[j] && level[j] <= r)
                dis[j] = min(dis[j], dis[t] + g[t][j]);
    }
    return dis[1];
}

int main()
{
    memset(g, 0x3f, sizeof(g));
    scanf("%d%d", &m, &n);
    int p, x, t, v;
    for (int i = 1; i <= n; ++ i ) g[i][i] = 0;
    for (int i = 1; i <= n; ++ i )
    {
        scanf("%d%d%d", &p, &level[i] , &x);
        g[0][i] = min(g[0][i], p);
        while ( x -- )
        {
            scanf("%d%d", &t, &v);
            g[t][i] = min(g[t][i], v);
        }
    }
    
    int res = 0x3f3f3f3f;
    for (int i = level[1] - m; i <= level[1]; ++ i )
    {
        int t = dijkstra(i, i + m);
        res = min(t, res);
    }
    printf("%d\n", res);
    return 0;
}
```
***
### 1135. 新年好
[1135. 新年好 - AcWing题库](https://www.acwing.com/problem/content/1137/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230801074443.png)

先用堆优dijkstra跑单源最短路，一共是6个源点，分别是家所在的点以及亲戚所在的点，保存最短距离
然后dfs爆搜，以1号点为起点，剩下五个点的全排列为搜索顺序，记录全排列中路径的最短距离
```cpp
#include <iostream>
#include <cstring>
#include <queue>
using namespace std;

const int N = 5e5 + 10, M = 2e5 + 10;
const int INF = 1e9;
int h[N], e[M], ne[M], w[M], idx = 1;
int dis[6][N]; bool st[N];
int tar[10];
int n, m;
typedef pair<int, int> PII;
priority_queue<PII, vector<PII>, greater<PII>> q;

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;    
}

void dijkstra(int start, int si) // si为start在dis数组中的一维下标
{
    memset(st, false, sizeof(st));
    dis[si][start] = 0;
    q.push({ 0, start });
    while (q.size())
    {
        auto t = q.top(); q.pop();
        int x = t.second, d = t.first;
        if (st[x]) continue;
        st[x] = true;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (dis[si][y] > d + w[i])
            {
                dis[si][y] = d + w[i];
                q.push({ dis[si][y], y});
            }
        }
    }
}

int dfs(int cnt, int si, int d) // cnt为已经拜访的亲戚数量，si为当前所在点在dis数组中的一维下标，d为递达当前点的“距离”
{
    if (cnt == 5)  return d;
    int res = INF;
    for (int i = 1; i <= 5; ++ i )
    {
        if (!st[i])
        {
            int y = tar[i]; // 要访问的下一个点在图中的编号
            st[i] = true;
            res = min(res, dfs(cnt + 1, i, d + dis[si][y]));
            st[i] = false;
        }
    }
    return res;
}

int main()
{
    memset(h, -1, sizeof(h));
    memset(dis, 0x3f, sizeof(dis));
    scanf("%d%d", &n, &m);
    for ( int i = 1; i <= 5; ++ i ) scanf("%d", &tar[i]);
    int x, y, t;
    while ( m -- )
    {
        scanf("%d%d%d", &x, &y, &t);
        add(x, y, t), add(y, x, t);
    }
    
    dijkstra(1, 0);
    for (int i = 1; i <= 5; ++ i ) dijkstra(tar[i], i);
    
    memset(st, false, sizeof(st));
    printf("%d\n", dfs(0, 0, 0));
    return 0;
}
```
debug：双向边，M又少开一倍，但是TLE，以后M都开两倍吧，这太难调试了
dfs最后要返回res，因为min要使用dfs的返回值
***
### 340. 通信线路
[340. 通信线路 - AcWing题库](https://www.acwing.com/problem/content/342/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230802215326.png)

分析题意：对于每条1~n的路径，我们只要支付第k+1大的边的金额即可。若边的数量少于k+1，那么支付金额为0
所以本题的最短路，短在第k+1大的边权中，最小的那个。最大中找最小，自然地想到二分。二分边权，得到ans，使得ans是所有1~n路径中，第k+1大的权值中最小的那个

二分的范围：$[0, 1e6+1]$，权值最小值-1~权值最大值+1
check：分析ans的性质，所有1~n的路径中，第k+1大的边权中最小值为ans
说明至少存在一条路径，经过了k条边权大于ans的边，即and是这条路径中第k+1大的边
那么其他路径就一定经过了多于k条权值大于ans的边
由此得到check的检查逻辑：根据参数x，判断1~n的所有路径中，是否存在一条路径，经过了小于等于k条的权值大于x的边。若存在返回true，否则返回false
试想一下，x越大，check的性质越容易满足，x越小，性质越难满足

两种特殊情况：ans为0，说明1~n的所有路径中，至少存在一条路径，经过了k条权值大于0的边，那么第k+1条边的权值为0，需要支付金额为0
ans为$1e6+1$，说明1~n的所有路径中，至少存在一条路径，经过了k条权值大于$1e6+1$的边，由于边权最大为$1e6$，所以这样的路径是不存在的，即1~n之间不连通

如何得知1~n的所有路径中，经过了几条权值大于x的边？
对于权值大于x的边，将其权值设置为1，否则设置为0，跑一遍最短路，就能知道1~n的最短路径经过了几条权值大于x的边
对于权值只有0和1的最短路问题，可以用双端队列bfs

```cpp
#include <iostream>
#include <cstring>
#include <deque>
using namespace std;

const int N = 1010, M = 20010;
int n, p, k;
int h[N], e[M], ne[M], w[M], idx = 1;
int dis[N]; bool st[N];
deque<int> q;

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ; 
}

bool check(int bound)
{
    memset(dis, 0x3f, sizeof(dis));
    memset(st, false, sizeof(st));
    dis[1] = 0;
    q.push_back(1);
    while (q.size())
    {
        int x = q.front(); q.pop_front();
        if (st[x]) continue;
        st[x] = true;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i], d = w[i] > bound;
            if (dis[y] > dis[x] + d)
            {
                dis[y] = dis[x] + d;
                if (d) q.push_back(y);
                else q.push_front(y);
            }
        }
    }
    return dis[n] <= k;
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d%d", &n, &p, &k);
    int x, y, d;
    while ( p -- )
    {
        scanf("%d%d%d", &x, &y, &d);
        add(x, y, d), add(y, x, d);
    }
    
    int l = 0, r = 1e6 + 1;
    while ( l < r )
    {
        int mid = l + r >> 1;
        if (check(mid)) r = mid;
        else l = mid + 1;
    }
    
    if (l == 1e6 + 1) puts("-1");
    else printf("%d\n", l);
    return 0;
}
```
***
### 342. 道路与航线
[342. 道路与航线 - AcWing题库](https://www.acwing.com/problem/content/description/344/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230803193351.png)

直接用spfa会被卡两个数据
根据题意：道路，双向边，存在环，正权；航线：单向边，无环，负权
其中关于是否有环以及权值的正负是选择算法的关键
若图中没有负权边，直接用堆d
若图中无环，直接按照拓扑序遍历所有点就能确定最短路

结合以上两者，可以得到以下算法：
将所有点根据道路的连通性分成多个连通块，每个连通块之间不连通
连通每个连通块的是航线，单向边。读取所有的航线后，按照拓扑序遍历每个连通块，对于每个连通块，跑堆d求最短路

用id数组表示每个点所属的连通块编号，用二维数组blocks存储每个连通块中点的编号
读入所有道路后，dfs可以遍历连通块中的所有点，维护出id和blocks数组信息
读入所有航线，维护出in数组，计算所有点的入度后按照拓扑序跑堆d
```cpp
// 道路是双向的，航线单向，可能是负数也可能是正数
// 并且图中保证不会存在环路
// 直接spfa？
#include <iostream>
#include <cstring>
#include <vector>
#include <queue>
using namespace std;

typedef pair<int, int> PII;
const int N = 3e4, M = 2e5;
int h[N], e[M], ne[M], w[M], idx = 1;
int n, p, r, s;
int dis[N]; bool st[N];
int id[N], cnt, in[N];
vector<int> blocks[N];
queue<int> q;

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

void dfs(int x)
{
    id[x] = cnt;
    blocks[cnt].push_back(x);
    for (int i = h[x]; i != -1; i = ne[i])
    {
        int y = e[i];
        if (!id[y]) dfs(y);
    }
}

void dijkstra(int k)
{
    priority_queue<PII, vector<PII>, greater<PII>> hp;
    for (auto x : blocks[k]) hp.push({ dis[x], x });
    
    while (hp.size())
    {
        auto t = hp.top(); hp.pop();
        int x = t.second, d = t.first;
        if (st[x]) continue;
        st[x] = true;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (id[x] != id[y] && -- in[id[y]] == 0) q.push(id[y]);
            if (dis[y] > d + w[i])
            {
                dis[y] = d + w[i];
                if (id[x] == id[y]) hp.push( {dis[y], y });
            }
        }
    }
}

void topsort()
{
    memset(dis, 0x3f, sizeof(dis));
    dis[s] = 0;
    for (int i = 1; i <= cnt; ++ i )
        if (!in[i]) q.push(i);
    
    while (q.size())
    {
        int x = q.front(); q.pop(); // 连通块id
        dijkstra(x);
    }
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d%d%d", &n, &r, &p, &s);
    int x, y, d;
    for (int i = 0; i < r; ++ i )
    {
        scanf("%d%d%d", &x, &y, &d);
        add(x, y, d), add(y, x, d);
    }
    
    for (int i = 1; i <= n; ++ i )
    {
        if (!id[i]) 
        {
            cnt ++ ;
            dfs(i); // 维护出i所在的连通块
        }
    }
    
    for (int i = 0; i < p; ++ i )
    {
        scanf("%d%d%d", &x, &y, &d);
        add(x, y, d);
        in[id[y]] ++ ;
    }
    
    topsort();
    
    for (int i = 1; i <= n; ++ i ) 
    {
        if (dis[i] >= 0x3f3f3f3f / 2) puts("NO PATH");
        else printf("%d\n", dis[i]);
    }
    return 0;
}
```
***
### 341. 最优贸易
[341. 最优贸易 - AcWing题库](https://www.acwing.com/problem/content/343/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230803202851.png)

状态表示：
集合：所有从1~n的路径，用分界点将这些路径一分为二
属性：在分界点之前（包括分界点）买入 - 在分界点（包括分界点）之后能卖出的最大价值，即最大卖出价值 - 最小买入价值
$f(i)$表示以i号点为分界点的1~n路径中，可以赚取的最大价值

状态划分：分界点从1~n，此时划分的子集不遗漏地组成了整个集合（虽然有重复，但是题目要求最优解，重复的情况不影响最优解）

如何求路径中的前半段最小值$dmin(k)$？
若与k直接相连的点有t个，那么$dmin(k) = min(dmin(s_1), dmin(s_2), ... dmin(s_t), w_k)$
走到k的最小值就等于从**所有走到与k直接相连的点的最小值以及自己的权值中**的最小值
由于这些状态依赖可能存在环，即当dp推导出的状态不是最优解，所以这里用最短路求这些状态
最短路用dijkstra或者spfa，是否能用dijkstra？由于在最短路问题中，当前点的最短路径由与之相连的所有点的最短路径累加上自己的权值确定，并且权值只能是正的
根据这个性质，已经确定最短路的点在之后的更新中不可能出现更短的路径

虽然这题的边权为正，但是这题的状态不是由之前的状态累加，而是从之前的状态中取min或者max，这就导致dijkstra每次的更新无法求得最优解，需要后续的再次更新，此时的时间复杂度无法保证，这是最关键的，所以不使用dijkstra

那spfa呢？spfa是bellman-ford的优化，bellman根据边的数量进行更新，每次更新所有边
所以bellman能够保证最短路的边数，即时间复杂度是确定的，所以可以使用sfpa

用sfpa跑出dmin和dmax数组的信息，分别存储了从1到k的最小值，以及从k到n的最大值
由于从k到n的起点是变化的，但是终点不变，所以这里可以建一张反向图，以n为起点，更新dmax
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 1e5 + 10, M = 2e6;
int w[N];
int hs[N], ht[N], e[M], ne[M], idx;
int dmin[N], dmax[N]; bool st[N];
int n, m;
int q[N];

void spfa(int h[], int dis[], int t)
{
    int tt = 0, hh = 0;
    if (t == 1)
    {
        memset(dis, 0x3f, sizeof(dmin));
        q[tt ++ ] = 1; st[1] = true;
        dis[1] = w[1];
    }
    else
    {
        memset(dis, -0x3f, sizeof(dmax));
        q[tt ++ ] = n; st[n] = true;
        dis[n] = w[n];
    }
    
    while (tt != hh)
    {
        int x = q[hh ++ ];
        if (hh == N) hh = 0;

        st[x] = false;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if ((t == 1 && dis[y] > min(dis[x], w[y])) || (t == 2 && dis[y] < max(dis[x], w[y])))
            {
                if (t == 1) dis[y] = min(dis[x], w[y]);
                else dis[y] = max(dis[x], w[y]);
                if (!st[y]) 
                {
                    st[y] = true, q[tt ++ ] = y;
                    if (tt == N) tt = 0;
                }
            }
        }
    }
}

void add(int h[], int x, int y)
{
    e[idx] = y, ne[idx] = h[x], h[x] = idx ++ ;
}

int main()
{
    memset(hs, -1, sizeof(hs));
    memset(ht, -1, sizeof(ht));
    scanf("%d%d", &n, &m);
    for (int i = 1; i <= n; ++ i ) scanf("%d", &w[i]);
    int x, y, t;
    while ( m -- )
    {
        scanf("%d%d%d", &x, &y, &t);
        add(hs, x, y), add(ht, y, x);
        if (t == 2) add(hs, y, x), add(ht, x, y);

    }
    
    spfa(hs, dmin, 1);
    spfa(ht, dmax, 2);
    
    int res = 0;
    for (int i = 1; i <= n; ++ i )
    {
        res = max(res, dmax[i] - dmin[i]);
    }
    
    printf("%d\n", res);
    
    return 0;
}
```

