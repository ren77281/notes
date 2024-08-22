```toc
```

偏数学的一个知识点
两个应用：
1. 求不等式组的可行解
2. 如何求可行解的最大值或者最小值

每个不等式形如：$x_i <= x_j + c$
在图论问题中，可以看成一条从j->i，边权为c的边。以除了$x_i$之外的任意一点为起点求出到$x_i$的最短路，那么该不等式一定成立
对于每个不等式，我们都求一次最短路，使该等式成立，最终就能求出这组方程的其中一个可行解

需要满足条件： 源点必须满足从源点出发，一定能走到所有边（不是所有点）的性质，为什么不是所有点？因为不等式可以转换成边，要是所有不等式成立，就要使所有的边满足该不等式，对于那些孤立的点，它的取值不会影响不等式的解

所以差分约束的步骤为：
1. 找到不等式关系，将每个不等式转换成边，j->i，权值为c
2. 找一个源点，使得源点一定能走到所有边
3. 求最短路
最终`dis[i]`为一个可行解

若图中存在负环，则可以将负环上的边转换成不等式，然后不断放缩，最终能推出一个矛盾的式子
放缩：每条边转换成不等式后，都与相邻的不等式存在关系，按照固定的方向进行不等式的替换
最终能推出：$x_i <= x_i + c_1 + c_2 + ...  + c_k$，由于该环是负环所以：$x_i < x_i$，即存在矛盾
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807155229.png)

若图中存在负环，那么不等式组无解，若不等式组无解，那么图中一定存在负环

补充：可以用最长路求可行解
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807155949.png)

转换不等式成为一个最长路的形式，i->j，权值为-c的边，向图中添加这条边即可
***
**求可行解中的最小值或者最大值**，最值指的是每个变量的最值
如果求的是最小值，则应该用最长路。如果求的是最大值，则应该用最短路
由于可行解之间是相对的，将每个可行解加上一个常数后，不等式组依然成立，所以此时不存在最值
只有不等式组中，有一个常数约束时，才存在最值
也可以理解成，链式关系放缩后，右边不存在变量，只存在常数，此时才存在最值

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807160332.png)

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807160908.png)

$x_i <= c$，c为一个常数，如何将其转换成边？
设置虚拟源点$x_0$，$x_i <= x_0 + c$，等价于$x_0->x_i$，权值为c的边
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807161123.png)

需要找出$x_i$的所有不等式链中的最小值，转换成图论问题就是以$x_i$为终点的最短路
***
### 裸题：1169. 糖果
[1169. 糖果 - AcWing题库](https://www.acwing.com/problem/content/1171/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807161954.png)

根据题意建立不等式组，题目要求可行解中的最小值，用最长路求解，每个不等式转换成>=的形式
由于每个人至少要有一个糖果，所以$x_i >= 1$，设置虚拟源点$x_0$，建立不等式$x_i >= x_0 + 1$，$x_i$为图中的所有点，所以该虚拟源点能走到图中的所有边，以该点为最长路的源点，用sfpa判断负环的同时求出最长路

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807161920.png)

```cpp
#include <iostream>
#include <cstring>
using namespace std;

typedef long long LL;
const int N = 1e5 + 10, M = 3e5 + 10;
int h[N], e[M], ne[M], w[M], idx;
int dis[N], cnt[N];
bool st[N];
int q[N], hh, tt;

int n, k;

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

bool spfa()
{
    memset(dis, -0x3f, sizeof(dis));
    dis[0] = 0;
    q[tt ++ ] = 0;
    while (hh != tt)
    {
        int x = q[-- tt];
        st[x] = false;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (dis[y] < dis[x] + w[i])
            {
                dis[y] = dis[x] + w[i];
                cnt[y] = cnt[x] + 1;
                if (cnt[y] >= n + 1) return true;
                if (!st[y]) 
                {
                    st[y] = true;
                    q[tt ++ ] = y;
                }
            }
        }
    }
    return false;
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d%d", &n, &k);
    int t, a, b;
    for (int i = 0; i < k; ++ i )
    {
        scanf("%d%d%d", &t, &a, &b);
        if (t == 1) add(a, b, 0), add(b, a, 0);
        else if (t == 2) add(a, b, 1);
        else if (t == 3) add(b, a, 0);
        else if (t == 4) add(b, a, 1);
        else add(a, b, 0);
    }
    for (int i = 1; i <= n; ++ i ) add(0, i, 1);

    if (spfa()) puts("-1");
    else
    {
        LL res = 0;
        for (int i = 1; i <= n; ++ i ) res += dis[i];
        printf("%lld\n", res);
    }
    return 0;
}
```
debug；M要开三倍，最坏情况下每次要建两条边，最后设置虚拟源点，每个点都要与之建一条边，所以三倍
***
### 前缀和+差分约束：362. 区间
[362. 区间 - AcWing题库](https://www.acwing.com/problem/content/364/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807174314.png)

由于$c_i$是小于等于区间长度的，所以我们可以将0~50000之间的数加入集合，此时的集合一定满足所有的限制条件。当然，这是最坏的情况
对于所有0~50000之间的数，我们有两种选择，选或不选其加入集合
用$s_i$表示前i个数中，被选择进入集合的数量
那么就有不等式：$s_i >= s_{i-1} , s_i - s_{i-1} <= 1, s_b - s_{a-1} >= c$
由于这些不等式需要使用前缀和，所以而前缀和数组一般不使用0下标，所以我们读入的所有数+1，$s_i$中i的范围从1~50001

现需要思考是否存在一个点，其能到达所有边，根据第一个不等式，我们得知：$s_0->s_1->s_2->...s_{50001}$，所以0号点能到达所有的点，也就能到达所有的边

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807174010.png)

```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 50010, M = 3 * N;
int h[N], e[M], ne[M], w[M], idx;
int dis[N], q[N], tt, hh;
bool st[N];

int n;

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

void spfa()
{
    memset(dis, -0x3f, sizeof(dis));
    q[tt ++ ] = 0;
    dis[0] = 0;
    while (hh != tt )
    {
        int x = q[hh ++ ];
        if (hh == N) hh = 0;
        st[x] = false;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (dis[y] < dis[x] + w[i])
            {
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
}

int main()
{
    memset(h, -1, sizeof(h));
    scanf("%d", &n);
    int a, b, c;
    for (int i = 0; i < n; ++ i )
    {
        scanf("%d%d%d", &a, &b, &c);
        a ++ , b ++ ;
        add(a - 1, b, c);
    }
    
    for (int i = 1; i < N; ++ i )
    {
        add(i, i - 1, -1);
        add(i - 1, i, 0);
    }
    
    spfa();
    printf("%d\n", dis[50001]);
    
    return 0;
}
```
debug：不等式的转换没有想清楚，写了`add(i, i - 1, 1)`，也不知道为什么TLE
***
### 1170. 排队布局
[1170. 排队布局 - AcWing题库](https://www.acwing.com/problem/content/1172/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230808071653.png)

根据题意，可以分析出三个不等式：$x_{i+1} >= x_i, |x_b - x_a| <= L, |x_b - x_a| >= D$
将这些不等式转换成边，建图
第一个问题：不存在要求的方案，即图中存在负环，不等式组无解
第二个问题：1和n的距离无限大，即图中1和n不属于同一连通块，1和n没有不等关系
第三个问题：若1和n位于同一连通块，1和n的最大距离，最大值问题用最短路。然而题目给定的不等式关系都是相对关系，没有绝对关系，无法求得最值。但是1和n的最大距离是一个相对距离，因此可以假设1或者n在数轴的0点，两者之间的最短路就是最大距离

差分约束还需要一个能递达所有边的点，显然第一个不等式说明了n点能递达所有点，即也能递达所有的边

分两次spfa求解三个问题，第一次spfa判断图中是否存在负环，第二次spfa求两点之间的最短路
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230807211738.png)
```cpp
#include <iostream>
#include <cstring>
#include <algorithm>
using namespace std;

const int N = 1010, M = 3e4 + 10;
int h[N], e[M], ne[M], w[M], idx;
int dis[N], cnt[N];
bool st[N];
int q[N];

int n, ml, md;

void add(int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

bool spfa(int t)
{
    int tt = 0, hh = 0;
    memset(dis, 0x3f, sizeof(dis));
    memset(cnt, 0, sizeof(cnt));
    for (int i = 1; i <= t; ++ i )
    {
        st[i] = true;
        q[tt ++ ] = i;
        dis[i] = 0;
    }
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
                cnt[y] = cnt[x] + 1;
                if (cnt[y] >= n) return true;
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
    memset(h, -1, sizeof(h));
    scanf("%d%d%d", &n, &ml, &md);
    for (int i = 1; i < n; ++ i ) add(i + 1, i, 0);
    int a, b, c;
    for (int i = 0; i < ml; ++ i )
    {
        scanf("%d%d%d", &a, &b, &c);
        if (b < a)  swap(a, b);
        add(a, b, c);
    }
    for (int i = 0; i < md; ++ i )
    {
        scanf("%d%d%d", &a, &b, &c);
        if (b < a) swap(a, b);
        add(b, a, -c);
    }
    
    if (spfa(n)) puts("-1");
    else
    {
        spfa(1);
        if (dis[n] == 0x3f3f3f3f) puts("-2");
        else printf("%d\n", dis[n]);
    }
    
    
    return 0;
}
```
debug：`scanf("%d%d%d", &a, &n, &c)`，把b打成n了，乐