```toc
```
## 连通性搜索
### 1112. 迷宫
[1112. 迷宫 - AcWing题库](https://www.acwing.com/problem/content/1114/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230816172638.png)

先对起点和终点的合法性进行检查，若合法，则进行dfs搜索
通过四个方向进行扩展，每次搜索都要标记st数组，否则dfs无法结束，将陷入无限地递归
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 110;
char g[N][N];
int n, k; bool st[N][N];
int x1, y1, x2, y2;
int dx[4] = { 0, 0, 1, -1 }, dy[4] = { 1, -1, 0, 0 };

bool dfs(int x, int y)
{
    st[x][y] = true;
    if (x == x2 && y == y2) return true;
    for (int i = 0; i < 4; ++ i )
    {
        int nx = x + dx[i], ny = y + dy[i];
        if (nx < 0 || nx >= n || ny < 0 || ny >= n) continue;
        if (st[nx][ny] || g[nx][ny] == '#') continue;

        if (dfs(nx, ny)) return true;
    }
    return false;
}

int main()
{
    cin >> k;
    while ( k -- )
    {
        cin >> n;
        for (int i = 0; i < n; ++ i )
            cin >> g[i];
        
        cin >> x1 >> y1 >> x2 >> y2;
        if (g[x1][y1] == '#' || g[x2][y2] == '#') puts("NO");
        else
        {
            memset(st, false, sizeof(st));
            if (dfs(x1, y1)) puts("YES");
            else puts("NO");
        }
    }
    return 0;
}
```
***
### 1113. 红与黑
[1113. 红与黑 - AcWing题库](https://www.acwing.com/problem/content/1115/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230816173908.png)

dfs实现的Floor Fill算法
处理数据得到起点的坐标，然后从起点开始往四个方向的黑色瓷砖dfs即可
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 25;
char g[N][N];
bool st[N][N];
int n, m, ans;

int dx[4] = { 0, 0, 1, -1 }, dy[4] = { 1, -1, 0, 0 };

void dfs(int x, int y)
{
    ans ++ ;
    st[x][y] = true;
    for (int i = 0; i < 4; ++ i )
    {
        int nx = x + dx[i], ny = y + dy[i];
        if (nx < 0 || nx >= n || ny < 0 || ny >= m) continue;
        if (st[nx][ny] || g[nx][ny] == '#') continue;
        dfs(nx, ny);
    }
}

int main()
{
    while (scanf("%d%d", &m, &n), n || m)
    {
        for (int i = 0; i < n; ++ i )
            scanf("%s", g[i]);
        ans = 0;
        memset(st, false, sizeof(st));
        for (int i = 0; i < n; ++ i )
            for (int j = 0; j < m; ++ j )
                if (g[i][j] == '@')
                    dfs(i, j);
        printf("%d\n", ans);
    }
    return 0;
}
```

debug：由于有多组测试数据，所以记得重置st数组
***
## 搜索顺序 
dfs只要保证不遗漏的爆搜所有情况，就能求出正确答案
一般有以下几种搜索顺序：
1. 每次的搜索方向确定，枚举所有方向即可
2. 提前预处理所有可能的搜索方向，搜索时按照预处理结果搜索
***
### 1116. 马走日
[1116. 马走日 - AcWing题库](https://www.acwing.com/problem/content/1118/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230816175046.png)


如何得知n \* m矩阵中的所有点被遍历完？用cnt记录当前dfs已经遍历的点的数量，当cnt = n \* m时，说明矩阵被遍历完
在进行某个点的dfs之前标记状态（cnt，st），完成该点的dfs后撤销状态
每次dfs向八个方向中未被标记过的格子扩展
```cpp
#include <iostream>
using namespace std;

const int N = 10;
bool st[N][N];
int t, n, m, cnt = 0, ans;
int dx[8] = { -2, -1, 1, 2, 2, 1, -1, -2 }, dy[8] = { 1, 2, 2, 1, -1, -2, -2, -1 };

void dfs(int x, int y)
{
    st[x][y] = true;
    cnt ++ ;
    if (cnt == n * m) ans ++ ;
    for (int i = 0; i < 8; ++ i )
    {
        int nx = x + dx[i], ny = y + dy[i];
        if (nx < 0 || nx >= n || ny < 0 || ny >= m) continue;
        if (!st[nx][ny]) dfs(nx, ny);
    }
    cnt -- ;
    st[x][y] = false;
}

int main()
{
    int x, y;
    scanf("%d", &t);
    while (t -- )
    {
        scanf("%d%d%d%d", &n, &m, &x, &y);
        ans = 0;
        dfs(x, y);
        printf("%d\n", ans);
    }
    return 0;
}
```
***
### 1117. 单词接龙
[1117. 单词接龙 - AcWing题库](https://www.acwing.com/problem/content/1119/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230816183640.png)

爆搜的第二种搜索顺序，先预处理出所有可能的搜索方向，根据题意，单词的后缀和其他单词的前缀相同时，两者可以进行拼接。也就是说，相同前缀和后缀的单词之间有一个搜索方向，我们要预处理所有单词之间的是否有“搜索方向”
但是单词可能有多个后缀与其他单词的前缀相同，需要把这些信息都保存下来吗？题目要求计算最长单词串的长度，所以如果两个单词有多种拼接方式的话，保存拼接后长度最长的单词即可。也就是长度最短的相同前缀与后缀
预处理的代码如何实现？两层循环枚举所有的单词，接着从最小的长度1开始枚举相同前后缀，枚举借助`substr`实现

```cpp
#include <iostream>
#include <cstring>
#include <string>
using namespace std;

const int N = 25;
string words[N];
int g[N][N];
int n; char bg;
int cnt[N], ans;

void dfs(string str, int pre)
{
    ans = max(ans, (int)str.size());
    cnt[pre] ++ ;
    
    for (int i = 0; i < n; ++ i )
    {
        if (g[pre][i] && cnt[i] < 2)
            dfs(str + words[i].substr(g[pre][i]), i);
    }
    
    cnt[pre] -- ;
}

int main()
{
    cin >> n;
    for (int i = 0; i < n; ++ i )
        cin >> words[i];
    
    cin >> bg;
    
    for (int i = 0; i < n; ++ i )
        for (int j = 0; j < n; ++ j )
        {
            string a = words[i], b = words[j];
            int over = min(a.size(), b.size());
            for (int k = 1; k < over; ++ k )
            {
                if (a.substr(a.size() - k, k) == b.substr(0, k))
                {
                    g[i][j] = k;
                    break;
                }
            }
        }
        
    for (int i = 0; i < n; ++ i )
        if (words[i][0] == bg)
            dfs(words[i], i);
        
    cout << ans << endl;
    return 0;
}
```
***
### 1118. 分成互质组
[1118. 分成互质组 - AcWing题库](https://www.acwing.com/problem/content/1120/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230817075409.png)

可以用二分图的最大团求解，也可以直接爆搜
首先考虑搜索时的两个方向：1. 加入组中 2. 新开一组
对于加入一组，每次加入的组都是最后一组。枚举所有数，考虑每个数是否能加入**最后**一组，注意是最后一组。如果考虑每个数是否能加入已经存在组的哪一组，那么dfs的决策树将多出很多的分支，使搜索过程变慢
其次，对于当前数来说，能加入最后一组时，就不用新开一组。假设当前数没有加入最后一组，而是新开了一组，那么对于新的一组来说，将当前数去掉加入最后一组。每组中的每个数依然是两两互质的，并且这样做能使得组数更少，所以正确的决策是只要能加入最后一组就加入最后一组，否则新开一组
dfs参数：gi表示当前组的下标，gcnt表示当前组的元素数量，cnt表示已经分好组的元素数量，start表示每次搜索数据范围的左边界
每次dfs时，试着将start确定的数据范围中的数加入当前组gi中，判断是否互质可以用gcd函数求最大公约数。只要有数加入了当前组，就不需要新开一组，此时进行下一次dfs。只有当数据范围内的所有数都无法加入当前组gi时，才会新开一组，gi + 1
```cpp
#include <iostream>
using namespace std;

const int N = 15;
int n, ans = N;
int a[N], g[N][N];
bool st[N];

int gcd(int a, int b)
{
    return b ? gcd(b, a % b) : a;
}

bool check(int gi, int gcnt, int x) // 遍历当前组中所有的数，判断x是否与所有数互质
{
    for (int i = 0; i < gcnt; ++ i ) 
    {
        if (gcd(g[gi][i], x) > 1)
            return false;
    }
    return true;
}

void dfs(int gi, int gcnt, int cnt, int start)
{
    if (gi + 1 >= ans) return;
    if (cnt == n) 
    {
        ans = min(ans, gi + 1);
        return;
    }
    
    bool flag = true;
    for (int i = start; i < n; ++ i )
    {
        if (!st[i] && check(gi, gcnt, a[i]))
        {
            flag = false;
            st[i] = true;
            g[gi][gcnt] = a[i];
            dfs(gi, gcnt + 1, cnt + 1, i + 1);
            st[i] = false;
        }
    }
    if (flag) dfs(gi + 1, 0, cnt, 0);
}

int main()
{
    cin >> n;
    for (int i = 0; i < n; ++ i ) cin >> a[i];
    
    dfs(0, 0, 0, 0);
    cout << ans << endl;
    return 0;
}
```
debug：新开一组时，start要重置为0不能写`dfs(gi + 1, 0, cnt, start)`
start确定的是：剩下的数是否能被放入当前组中，新开一组时，要将其重置为0，否则将漏掉很多数