```toc
```

### 裸题：904. 虫洞
[904. 虫洞 - AcWing题库](https://www.acwing.com/problem/content/906/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230806205731.png)

```cpp
// 虫洞是负权且单向边，道路是正权且双向边，题目较裸，判断有无负环即可
#include <iostream>
#include <cstring>
using namespace std;

const int N = 510, M = 6010;
int h[N], e[M], ne[M], w[M], idx;
int n, m, k;
int dis[N], cnt[N];
int q[N];
bool st[N];

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

bool spfa()
{
    int tt = 0, hh = 0;
    memset(cnt, 0, sizeof(cnt));
    memset(dis, 0, sizeof(dis));
    memset(st, 0, sizeof(st));
    for (int i = 1; i <= n; ++ i ) st[i] = true, q[tt ++ ] = i;
    while (hh != tt)
    {
        int x = q[hh ++ ];
        if (hh == N) hh = 0;
        st[x] = false;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (dis[y] > dis[x] + w[i])
            {
                cnt[y] = cnt[x] + 1;
                if (cnt[y] >= n) return true;
                dis[y] = dis[x] + w[i];
                if (!st[y]) 
                {
                    st[y] = true;
                    q[tt ++ ] = y;
                    if (tt == N) tt = 0;
                }
            }
        }
    }
    return false;
}

int main()
{
    int T;
    scanf("%d", &T);
    while (T -- )
    {
        memset(h, -1, sizeof(h));
        idx = 0;
        scanf("%d%d%d", &n, &m, &k);
        int x, y, d;
        for (int i = 0; i < m; ++ i )
        {
            scanf("%d%d%d", &x, &y, &d);
            add(x, y, d), add(y, x, d);
        }
        for (int i = 0; i < k; ++ i )
        {
            scanf("%d%d%d", &x, &y, &d);
            add(x, y, -d);
        }
        if (spfa()) puts("YES");
        else puts("NO");
    }
    return 0;
}
```
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230806215858.png)
这个\=\=真的服，调半天，还有，邻接表的大小又设置错了
***
### 01分数规划：361. 观光奶牛
[361. 观光奶牛 - AcWing题库](https://www.acwing.com/problem/content/363/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807085334.png)

在图论问题中，所有形如：某个部分之和除以某个部分之和最大的问题，被称为01分数规划，通常使用二分解决这类问题
根据题意，这道题的答案范围在$(0, 1000]$中，我们需要二分这个区间找到答案
若点权之和/边权之和大于等于mid，则说明答案在$[mid, r]$之间
反之，点权之和/边权小于mid，则说明答案在$[l, mid]$之间
根据这个二段性，我们能二分出ans，使得边权之和/边权之和的最大值 = ans

现在的问题是check如何实现？
整理不等式，如下图：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807085246.png)

一个常用的技巧：若图中的环既有点权又有边权，那么可以将点权加到出边或者入边上
那么不等式的求和可以提到外面，结合这个技巧，将点权和边权结合
若一条边由x->y，权值为w，那么将其权值设置为$f_x-mid*w$，$f_x$为x的点权
问题就转换成了图中是否存在一个正环？
求正环只要修改三角不等式即可：`dis[y] < dis[x] + w[i]`

总结下：check判断图中是否存在一个环，其点权之和/边权之和大于等于mid，转换成图中是否存在一个正环（或权值和为0的环），若存在，则l = mid，否则r = mid，

1. 思考题目的二段性
2. 根据不等式重置边/点权
3. 根据不等式判断题目的具体问题：负环/最小生成树/最短路
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 1010, M = 5010;
int h[N], e[M], ne[M], w[M], idx;
int f[N];
double dis[N];
int cnt[N]; bool st[N];
int q[N];

int n, m;

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

bool check(double mid)
{
    memset(dis, 0, sizeof(dis));
    memset(cnt, 0, sizeof(cnt));
    int tt = 0, hh = 0;
    
    for (int i = 1; i <= n; ++ i ) st[i] = true, q[tt ++ ] = i;
    while (hh != tt)
    {
        int x = q[hh ++ ];
        if (hh == N) hh = 0;
        st[x] = false;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (dis[y] <= dis[x] + f[x] - mid * w[i])
            {
                dis[y] = dis[x] + f[x] - mid * w[i];
                cnt[y] = cnt[x] + 1;
                if (cnt[y] >= n) return true;
                if(!st[y])
                {
                    st[y] = true;
                    q[tt ++ ] = y;
                    if (tt == N) tt = 0;
                }
            }
        }
    }
    return false;
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d", &n, &m);
    for (int i = 1; i <= n; ++ i ) scanf("%d", &f[i]);
    int x, y, d;
    for (int i = 0; i < m; ++ i )
    {
        scanf("%d%d%d", &x, &y, &d);
        add(x, y, d);
    }
    
    double l = 0, r = 1000;
    while (r - l > 1e-4)
    {
        double mid = (l + r) / 2;
        if (check(mid)) l = mid;
        else r = mid;
    }
    
    printf("%.2lf\n", r);
    return 0;
}
```
debug：点权需要从数组1号下标开始读取
***
### 特殊建图与01分数规划+trick：1165. 单词环
[1165. 单词环 - AcWing题库](https://www.acwing.com/problem/content/1167/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807101531.png)

估算一下这题的数据量，如果按照题意建图，不仅爆空间还会爆时间，所以这题需要考虑其他建图方式
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807101449.png)

题目给定的建图方式是：单词为点，若两单词能相连，那么边的权值为1
考虑新的建图方式，以单词的前两个字符为起点，最后两个字符为终点，建立一条有向边，权值为单词的长度。这种建图方式中，点的数量最多为26 \* 26，边的数量为$10^5$

其次，题目要求环中所有单词的长度之和 / 环中的单词数量最大，显然是01分数规划
二分答案，答案的范围是$(0, 1000]$，最大的答案为每个单词长度都是1000，而最小的答案0是取不到的，最小的情况应该是1，0用来表示无解
整理不等式，重新设置边权为$w_i - 1 * mid$，1是由环中点的数量累加后（第二个式子）再把累加提到外面（第三个等式）得到的
check：每次根据mid判断图中是否存在正环或零环，若存在返回true，反之返回false

trick：如果spfa更新了很多次还没有结束循环，那么有极大概率可以认为图中存在环，这里设置阈值为10000（点数的十几倍），当循环次数超过该值时，直接认为图中存在环、
不过这样的trick在正规比赛中不会出现
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 27 * 27, M = 1e5 + 10;
int h[N], e[M], ne[M], w[M], idx;
double dis[N];
int cnt[N], q[N];
bool st[N];

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

bool check(double mid)
{
    memset(dis, 0, sizeof(dis));
    memset(cnt, 0, sizeof(cnt));
    int tt = 0, hh = 0, count = 0;
    for (int i = 0; i < N - 1; ++ i ) q[tt ++ ] = i, st[i] = true;
    while (hh != tt )
    {
        int x = q[hh ++ ];
        if (hh == N) hh = 0;
        st[x] = false;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (dis[y] <= dis[x] + w[i] - mid)
            {
                cnt[y] = cnt[x] + 1;
                if (cnt[y] >= N) return true;
                if (++ count >= 10000) return true;
                dis[y] = dis[x] + w[i] - mid;
                if (!st[y])
                {
                    st[y] = true;
                    q[tt ++ ] = y;
                    if (tt == N) tt = 0;
                }
            }
        }
    }
    return false;
}

int main()
{
    int m;
    char str[1010];
    while (scanf("%d", &m), m)
    {
        memset(h, -1, sizeof(h));
        idx = 0;
        for (int i = 0; i < m; ++ i )
        {
            scanf("%s", str);
            int len = strlen(str);
            if (len >= 2)
            {
                int x = (str[0] - 'a') * 26 + str[1] - 'a';
                int y = (str[len - 2] - 'a') * 26 + str[len - 1] - 'a';
                add(x, y, len);
            }
        }
        
        double l = 0, r = 1000;
        while (r - l > 1e-4)
        {
            double mid = (l + r) / 2;
            if (check(mid)) l = mid;
            else r = mid;
        }
        
        if (r < 1e-4) puts("No solution");
        else printf("%.2lf\n", r);
    }
    return 0;
}
```

debug：dis数组的类型开成int，想着边的权值为整数，int就行，然而边权被重置，类型是浮点数


