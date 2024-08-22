```toc
```

## 定义
> 小学一笔画问题，每条边只经过一次

判断图是否存在欧拉回路：判断图是否连通（存在孤立边），再根据有向/无向具体判断

对于无向图来说，**欧拉路径**中，起点和终点的度数为奇数，中间点的度数为偶数
起点和终点：开始和结束时必须经过一条边，其余情况为：从一条边进入，再从另一条边离开，即度数为`1 + 2 * n`
中间点：一条边进入，一条边离开，度数为`2 * n`

欧拉回路中，所有点的度数为偶数
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230813174916.png)

七桥问题中，由于每个点的度数为奇数，所以不可能存在欧拉路径
以下结论直接记：
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230813175508.png)
***
### 欧拉路径的性质：1123. 铲雪车
[1123. 铲雪车 - AcWing题库](https://www.acwing.com/problem/content/1125/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230814072250.png)

根据题意，所有的街道都是双向，说明这张图是一张有向图。每个城市都连接了街道，说明这张图是连通图。将每个城市看成点，那么每个点的入度都等于出度，这张图存在欧拉回路，所以无论如何，我们都有一条路径，能够不重复不遗漏的遍历所有的边，而这道题要求时间（最短路径），我们不需要算出具体的欧拉回路，以为欧拉回路的距离一定是所有路径之和，只要根据题意将路径转换成时间即可
```cpp
#include <iostream>
#include <cmath>
using namespace std;

int main()
{
    double x1, y1, x2, y2;
    scanf("%lf%lf", &x1, &y1);
    double sum = 0;
    while (~scanf("%lf%lf%lf%lf", &x1, &y1, &x2, &y2))
    {
        double x = x1 - x2, y = y1 - y2;
        sum += sqrt(x * x + y * y) * 2;
    }
    
    int mte = round(sum / 1000 * 60 / 20);
    int hour = mte / 60;
    mte %= 60;
    printf("%d:%02d\n", hour, mte);
    return 0;
}
```
debug：距离要乘2，每条道路都是双向的
***
### 边编号输出欧拉路径：1184. 欧拉回路
[1184. 欧拉回路 - AcWing题库](https://www.acwing.com/problem/content/1186/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230814084157.png)

欧拉路径的板子：
若一张图存在欧拉路径，可以用dfs递归完所有的边，在dfs回溯时记录边的编号到数组中，最后逆着输出该数组即可
用uesd数组标记已经遍历过的边，同时每遍历完一条边就删除这条边，防止重复遍历
其实有向图不需要使用used数组，只要删除遍历完的边即可
开uesd数组是因为无向图，由于我们用单链表存储边，虽然可以删除当前正在遍历的边，但是在无向图中，删除反向边比较麻烦，所以这里用uesd数组标记反向边，保证每条边只会遍历一次

需要注意，不要遍历孤立点，若图中存在欧拉路径，孤立点不会影响欧拉路径
还要注意，这题的无向图的欧拉回路中，若边的方向与输入给定的方向不同，那么需要输出编号的负数
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 1e5 + 10, M = 4e5 + 10;
int h[N], e[M], ne[M], idx;
int din[N], dout[N];
int ans[M / 2], cnt;
bool used[M];
int type, n, m;

void add(int x, int y)
{
    e[idx] = y, ne[idx] = h[x], h[x] = idx ++ ;
}

void dfs(int x)
{
    for (int &i = h[x]; i != -1;)
    {
        if (used[i])
        {
            i = ne[i];
            continue;
        }
        
        int t; // 当前边的编号
        used[i] = true;
        if (type == 1) 
        {
            used[i ^ 1] = true;
            t = i / 2 + 1;
            if (i & 1) t = -t;
        }
        else t = i + 1;
        
        int y = e[i];
        i = ne[i];
        dfs(y);
        
        ans[++ cnt] = t;
    }
}

int main()
{
    memset(h, -1, sizeof(h));
    cin >> type >> n >> m;
    int x, y;
    for (int i = 0; i < m; ++ i )
    {
        scanf("%d%d", &x, &y);
        add(x, y);
        if (type == 1) add(y, x);
        din[y] ++ , dout[x] ++ ;
    }
    
    if (type == 1)
        for (int i = 1; i <= n; ++ i )
            if ((din[i] + dout[i]) % 2)
            {
                puts("NO");
                return 0;
            }

    if (type == 2)
        for (int i = 1; i <= n; ++ i )
            if (din[i] != dout[i])
            {
                puts("NO");
                return 0;
            }
            
    for (int i = 1; i <= n; ++ i )
        if (h[i] != -1)
        {
            dfs(i);
            break;
        }
    
    if (cnt < m)
    {
        puts("NO");
        return 0;
    }
    puts("YES");
    for (int i = m; i ; -- i )
        printf("%d ", ans[i]);
    puts("");
    
    return 0;
}
```

要特别注意：求欧拉路径后，必须要判断这条路径是否遍历了所有的边
***
### 点编号字典序最小输出欧拉路径：1124. 骑马修栅栏
[1124. 骑马修栅栏 - AcWing题库](https://www.acwing.com/problem/content/description/1126/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230814100630.png)

题目要求按照点的编号最小字典序输出欧拉路径，可以`dfs(x)`时，可以先遍历与x相连的编号较小的点，那么与x相连的点中，相较于编号较大的点，编号较小的点将被存储到序列的靠后位置，逆序后编号较小的点位于序列的靠前位置，满足最小字典序

为什么不使用邻接表存储图？
用邻接表存储图时，选择编号较小的点会比较麻烦，需要对单链表中的元素进行排序
用邻接矩阵存储图，直接按照下标从小到大dfs与x相连的点即可

需要注意的是，题目可能存在重边，通常邻接矩阵存储边权的最小/大值，但是这题需要存储两点间边的数量，以保证之后的dfs中每条边都被遍历
还有一点：图是无向图，但题目没有说存在欧拉回路还是欧拉路径，如果存在欧拉路径，那么起点和终点的度数为奇数。如果存在欧拉回路，那么所有点的度数为偶数
所以dfs之前需要找度数为奇数的点，若找到，说明该图一定存在欧拉路径，可能不存在欧拉回路。此时从度数为偶数的点dfs将无法正确遍历欧拉路径，只有存在欧拉回路的情况下，才能从度数为偶数的点dfs
并且，1不是最小的点编号，点编号范围在1~500，所以要找到一个度数非0的点编号，再找是否存在欧拉路径，最后再dfs
由于使用邻接矩阵存储无向图，所以不需要开used数组，每次遍历一条边，能够很简单地删除这条边和其反向边
```cpp
#include <iostream>
using namespace std;

const int N = 510, M = 2100;
int g[N][N], d[N];
int ans[M], cnt;
int n, m;

void dfs(int x)
{
    for (int y = 1; y < N; ++ y )
    {
        if (g[x][y])
        {
            g[x][y] -- , g[y][x] --;
            dfs(y);
        }
    }
    ans[ ++ cnt ] = x;
}

int main()
{
    scanf("%d", &m);
    int x, y;
    for (int i = 0; i < m; ++ i )
    {
        scanf("%d%d", &x, &y);
        g[x][y] ++ , g[y][x] ++ ;
        d[x] ++ , d[y] ++ ;
    }
    
    int start = 1;
    while (!d[start]) start ++ ;
    for (int i = start; i < N; ++ i )
        if (d[i] % 2)
        {
            start = i;
            break;
        }

    dfs(start);

    for (int i = cnt; i; -- i ) printf("%d\n", ans[i]);
    return 0;
}
```
***
### 并查集判断有向图是否存在欧拉路径：1185. 单词游戏
[1185. 单词游戏 - AcWing题库](https://www.acwing.com/problem/content/1187/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230814102015.png)

建图方式：以单词的第一个字母或最后一个字母为点，从前往后建一条有向边
问题转换成：判断有向图中是否存在欧拉路径，当然可能也是欧拉回路
由于不需要找出具体的欧拉路径，所以可以不用dfs，只要满足所有点的入度等于出度（欧拉回路），或者（欧拉路径）入度比出度多1（终点），出度比入度多1（起点）
该图就存在欧拉路径，当然，还需要判断是否有孤立边，由于这题特殊的建图方式，若存在一个单词是孤立的，那么在图中表现为两点一边的孤立
判断是否存在孤立边，可以dfs跑一遍欧拉路径，判断边数是否小于图的边数
但是可以用更简单的并查集，维护每个点属于的集合，建完图后，一旦出现两点属于不同集合，那么图中一定存在孤立边（因为该图中的点一定连接着一条边，所以点孤立=边孤立）
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 30, M = 1e5 + 10;
bool st[N]; 
int din[N], dout[N];
int p[N];
int T, m;

int find(int x)
{
    if (x != p[x]) p[x] = find(p[x]);
    return p[x];
}

int main()
{
    scanf("%d", &T);
    while (T -- )
    {
        memset(st, false, sizeof(st));
        memset(din, 0, sizeof(din));
        memset(dout, 0, sizeof(dout));
        for (int i = 0; i < 26; ++ i ) p[i] = i;
        char str[1010];
        scanf("%d", &m);
        for (int i = 0; i < m; ++ i )
        {
            scanf("%s", str);
            int len = strlen(str);
            int x = str[0] - 'a', y = str[len - 1] - 'a';
            st[x] = st[y] = true;
            din[y] ++ , dout[x] ++ ;
            p[find(x)] = p[find(y)];
        }
        
        int start = 0, end = 0; // 起点和终点的数量
        bool flag = true;
        for (int i = 0; i < 26; ++ i )
            if (din[i] != dout[i])
            {
                if (din[i] + 1 == dout[i]) end ++ ;
                else if (din[i] == dout[i] + 1) start ++ ;
                else 
                {
                    flag = false;
                    break;
                }
            }
            
        if (flag && !((!start && !end) || (start == 1 && end == 1))) // 起点数和终点数不正确
            flag = false;
        // 判断是否存在孤立边
        int t = -1;
        for (int i = 0; i < 26; ++ i )
            if (st[i])
            {
                if (t == -1) t = find(i);
                else if (t != find(i))
                {
                    flag = false;
                    break;
                }
            }
            
        if (flag) puts("Ordering is possible.");
        else puts("The door cannot be opened.");
    }
    return 0;
}
```
