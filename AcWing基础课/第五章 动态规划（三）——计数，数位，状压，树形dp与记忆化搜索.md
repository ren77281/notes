```toc
```

## 计数dp
### 900. 整数划分
[900. 整数划分 - AcWing题库](https://www.acwing.com/problem/content/902/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230721092825.png)

状态表示：
集合：总和为1~n的所有数。用两个维度限制，将$i$个数相加，总和为$j$的所有数
属性：方案数
$f(i , j)$表示：将$i$个数相加，总和为$j$的所有方案数

状态计算：
思考如何划分$f(i ,j)$这个集合？这个划分比较难想，所有方案中，最小值为1的方案与最小值不为1的方案，这些方案能不重不漏的组成$f(i, j)$
对于最小值为1的方案，将1删除，那么该集合就能表示为$f(i-1, j-1)$
对于最小值不为1的方案，将所有数减去1，总共减去i，此时的集合表示为$f(i, j-i)$
综上，$f(i, j) = f(i-1, j-1) + f(i, j-i)$

最后需要将总和为n的所有方案（*方案中的数字个数从1~n*）相加

仔细审题，题目规定$k >= 1$，一个数划分后至少要有一个数，极限情况是1被划分为1，而0不能被划分，所以初始化$f[1][1] = 1$
划分中的每个数都是正整数，不能出现5 = 5 + 0 + 0 + 0这样的划分，所以要保证$j >= i$时，再更新状态

```cpp
#include <iostream>
using namespace std;

const int N = 1010, mod = 1e9 + 7;
int f[N][N];

int main()
{
    int n, res = 0;
    scanf("%d", &n);
    f[1][1] = 1;
    
    
    for (int j = 2; j <= n; ++ j )
        for (int i = 1; i <= j; ++ i )
            f[i][j] = (f[i - 1][j - 1] + f[i][j - i]) % mod;
            
    for (int i = 1; i <= n; ++ i ) res = (res + f[i][n]) % mod;

    printf("%d\n", res);
    return 0;
}
```
***
看成完全背包问题：
忽略背包问题中的价值，将n的划分看成用体积为1~n的物品恰好装满背包，每个物品使用次数不限
状态表示：
集合：所有装满背包的方法，用两个维度限制集合。只用前i个物品装满背包，背包的容积为j
属性：方法数
$f(i, j)$表示从前i个物品中选择物品装入背包，将容积为j的背包装满的方法数

状态计算：
如何划分$f(i, j)$这个集合？背包问题思考第i个物品是否选择的两种情况。不选择第i个物品，此时的集合为$f(i - 1, j)$。选择第i个物品，由于可以选择无数个物品直到上限，所以集合为$f(i, j-k*w[i]), k >= 1, j - k * w[i] >= 0$，该集合可以优化成$f(i, j - w[i])$
所以，$f(i, j) = f(i - 1, j) + f(i, j - w[i])$

```cpp
#include <iostream>
using namespace std;

typedef long long LL;
const int N = 1010, mod = 1e9 + 7;
int f[N][N];

int main()
{
    int n;
    scanf("%d",&n);
    f[0][0] = 1;
    
    for (int i = 1; i <= n; ++ i )
        for (int j = 0; j <= n; ++ j )
        {
            f[i][j] += f[i - 1][j];
            if (i <= j) f[i][j] = ((LL)f[i][j] + f[i][j - i]) % mod;
        }
            
    printf("%d\n", f[n][n]);
    return 0;
}
```
***
## 数位dp
### 338. 计数问题
[338. 计数问题 - AcWing题库](https://www.acwing.com/problem/content/340/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230721101307.png)

求一个范围内，某个数的出现次数：`count(n ,x)`表示1~n中x的出现次数
当范围为$[a, b]$时，x的出现次数为`count(b, x) - count(a - 1, x)`，类似前缀和
分情况讨论
当$n = abcdefg, x = 1$时，求1在第4位出现的次数
$1 <= xxx1yyy <= abcdefg$
1. $xxx < abc$，出现次数为$abc * 1000$
2. $xxx = abc$
  -  $d < 1$，出现次数为0
  -  $d = 1$，出现次数为$efg + 1$
  - $d > 1$，出现次数为1000

将以上两种情况的x出现次数相加，就能得到在1 ~ abcdefg中，x在第4位的出现次数，枚举每一位即可得到x在所有位的出现次数
其中，考虑边界情况，x在最高位与最低位的出现次数
x在最高位的出现次数只要考虑第二种情况
x在最低位的出现次数两种情况都要考虑
当$x=0$时，第一种情况的出现次数为$(abc-1) * 1000$，需要减去$xyz=000$的情况。因为当$xyz=000$时，这将和第二种情况重复计算
同时，不能统计0在最高位出现的次数，这是不合法的

总结下边界情况：
1. 统计$x$在最高位的出现次数时，忽略第一种情况，当$x=0$时，两种情况都忽略
2. 统计0的出现次数时，第一种情况的出现次数为$(abc-1) * 1000$

```cpp
#include <iostream>
using namespace std;

int power10(int x)
{
    int res = 1;
    while (x -- ) res *= 10;
    return res;
}

int get(int num, int l, int r)
{
    int a = power10(r);
    int b = power10(l - 1);
    return num % a / b;
}

int count(int n, int x)
{
    int len = 1;
    for (int i = 10; n / i; i *= 10) len ++ ;
    
    int res = 0;
    for (int i = len - !x; i > 0; -- i )
    {
        if (i < len)
            res += (get(n, i + 1, len) - !x) * power10(i - 1);
        if (get(n, i, i) > x) res += power10(i - 1);
        if (get(n, i, i) == x) res += get(n, 1, i - 1) + 1;
    }
    return res;
}

int main()
{
    int a, b;
    while (scanf("%d %d", &a, &b), a || b)
    {
        if (a > b) swap(a, b);
        for (int i = 0; i < 10; ++ i ) 
            printf("%d ", count(b, i) - count(a - 1, i));
        printf("\n");
    }
    return 0;
}
```
注意：$a$不是严格小于$b$的
***
## 状压dp
### 291. 蒙德里安的梦想
[291. 蒙德里安的梦想 - AcWing题库](https://www.acwing.com/problem/content/293/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230721154941.png)

核心：先放横着的，再放竖着的。先把所有横着的摆完，在摆放合法的情况下，将竖着的方块直接放入剩下的格子中，这些方块只有一种放法。所以问题的关键就来到了如何摆放横着的方块
（*若无特别说明，方块的意思是横着的方块*）
对于方格的某两列来说，假设现在要将方块摆放到第i列上，怎样摆放是合法的？若i-1列也摆放了方块，那么第i列的同一行就不能摆放方块，因为这个位置被前一列的方块占用
用二进制表示某一列摆放的方块状态：比如10010表示该列的第1行与第4行摆放了木块
合法的情况为：两组二进制数相与，结果为0。即将当前列的二进制状态与前一列的二进制状态做按位与操作，得到的结果为0，说明没有方块冲突，当前列的方块可以摆放
当前列的方块摆放完成后，若出现连续的空格且空格数量为奇数，那么这个位置无法摆放竖着的方块，说明当前列的摆放是非法的
总结下：两种不合法摆放
1. 当前列的摆放和前一列的摆放冲突
2. 当前摆放完成后，出现连续且数量为奇数的空格

状态表示：
集合：所有的摆放方式，用两个条件限制这个集合：摆放第i列且前一列的状态为j的方案
属性：摆放的方案数
注意，j为二进制表示，这是状态压缩的体现，1表示当前行的前一列已经摆放方格，0表示当前行的前一列没有摆放方格
$f(i, j)$表示摆放第i列且前一列的状态为j的方案数
假设有m行，答案返回摆放第m+1行且前一列的状态为全0（*最后一列不摆放方格为合法状态*）的方案数

状态计算：思考如何划分$f(i, j)$？思考最后一步，枚举前一列的前一列的摆放状态k，在k的状态上摆放j状态，所有使得摆放后状态合法的k状态数量就是$f(i, j)$的值
注意，j状态固定，k状态需要从0~$2^{n}-1$之间枚举，若状态合法，$f(i, j) += f(i-1, k)$ 

思考初始状态，由于不存在第0列，所以$f(1, 0) = 1$，表示摆放第1列且第0列的状态为全0的方案数为1，$k>0$时，$f(i, k)=0$，因为第0列无法摆放方格

因为出现连续空格且空格数量为奇数的状态时，该状态不合法，所以先预处理所有合法状态，用st数组存储
```cpp
#include <iostream>
#include <cstring>
using namespace std;

typedef long long LL;
const int N = 15, M = 1 << N;
bool st[M];
LL f[N][M];

int main()
{
    int n, m;
    while (scanf("%d%d", &n, &m), n || m)
    {
        // 预处理所有的状态，出现连续奇数个空格为非法
        for (int i = 0; i < 1 << n; ++ i )
        {
            int cnt = 0; // 连续空格的数量
            st[i] = true;
            for (int j = 0; j < n; ++ j )
            {
                if ((i >> j) & 1)
                {
                    if (cnt & 1) st[i] = false;  // 连续奇数个空格
                    cnt = 0;
                }
                else cnt ++ ;
            }
            if (cnt & 1) st[i] = false;
        }
        
        memset(f, 0, sizeof(f));
        f[1][0] = 1;
        for (int i = 2; i <= m + 1; ++ i )
            for (int j = 0; j < 1 << n; ++ j )
                for (int k = 0; k < 1 << n; ++ k )
                    if ((j & k) == 0 && st[j | k])
                        f[i][j] += f[i - 1][k];
                        
        printf("%ld\n", f[m + 1][0]);
    }
     
    return 0;
}
```
注意：不能在while循环之前以N和M预处理st数组，因为n和m会影响st的值，所以要在读取n和m之后再预处理st数组
debug：注意处理多组测试数据时，f数组要重制为0
&的优先级低于\==，所以`if ((j & k == 0) && st[j | k])`是错误的

更新f数组中，每次i循环时，都要重复判断j和k是否合法，可以先预处理j状态下合法的所有k状态
```cpp
#include <iostream>
#include <cstring>
#include <vector>
using namespace std;

typedef long long LL;
const int N = 14, M = 1 << N;
LL f[N][M];
bool st[M];
vector<int> state[M];

int main()
{
    int n, m;
    while (scanf("%d%d", &n, &m), n || m)
    {
        for (int i = 0; i < 1 << n; ++ i )
        {
            int cnt = 0;
            st[i] = true;
            for (int j = 0; j < n; ++ j )
            {
                if ((i >> j) & 1)
                {
                    if (cnt & 1) st[i] = false;
                    cnt = 0;
                }
                else cnt ++ ;
            }
            if (cnt & 1) st[i] = false;
        }
        
        for (int j = 0; j < 1 << n; ++ j )
        {
            state[j].clear();
            for (int k = 0; k < 1 << n; ++ k )
                if ((j & k) == 0 && st[j | k])
                    state[j].push_back(k);
        }
        
        memset(f, 0, sizeof(f));
        f[1][0] = 1;
        for (int i = 2; i <= m + 1; ++ i )
            for (int j = 0; j < 1 << n; ++ j )
                for (auto k : state[j])
                    f[i][j] += f[i - 1][k];
                        
        printf("%ld\n", f[m + 1][0]);
    }
    return 0;
}
```
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230721173311.png)
***
### 91. 最短Hamilton路径
[91. 最短Hamilton路径 - AcWing题库](https://www.acwing.com/problem/content/93/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230722072345.png)

题目要求每个点只能走一次，且点的数量只有20，考虑状态压缩
将是否走过每个点的状态用0和1表示，压缩成二进制数
状态表示：
集合：图中所有的路径，用两个维度进行限制。一是当前走到的点i，一是已经走过的状态j，那么j中一定包含走过i点的状态
属性：路径的最小权值和
$f(i, j)$表示当前走到第i个点，且已经走过的点为j的最小路径和，j是一个二进制表示

状态计算：
如何划分$f(i, j)$？当前位于第i个点，若走过第k个点（*j包含走过第k个点的状态*），那么集合就能划分成$f(k, j')$，$j$的第$i$位为1，而$j'$的第$i$位为0
所以，枚举状态j的所有走过的点，此时的$f(k, j')$就是集合$f(i, j)$的划分
此时$f(i, j) = min(f(i, j), f(k, j - (1 << i)))$

```cpp
#include <cstring>
#include <iostream>
using namespace std;

const int N = 21, M = 1 << N;
int n, f[N][M], w[N][N];

int main()
{
    scanf("%d", &n);
    memset(f, 0x3f, sizeof(f));
    for (int i = 0; i < n; ++ i)
        for (int j = 0; j < n; ++ j )
            scanf("%d", &w[i][j]);
    
    f[0][1] = 0;
    for (int j = 1; j < 1 << n; j += 2 )
        for (int i = 0; i < n; ++ i )
            if ((j >> i) & 1)
                for (int k = 0; k < n; ++ k )
	                    if ((j >> k) & 1)
                        f[i][j] = min(f[i][j], f[k][j - (1 << i)] + w[i][k]);
                        
    printf("%d\n", f[n - 1][(1 << n) - 1]);
    
    return 0;
}
```
与线性dp不同，状态dp的代码实现，一般都是枚举所有的状态
为什么`j += 2`，因为所有的状态都要保证起点为第0个点，`j ++`其实也行，不过更新了一半的无效数据，慢了一倍
***
## 树形dp
### 285. 没有上司的舞会
[285. 没有上司的舞会 - AcWing题库](https://www.acwing.com/problem/content/287/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230721184546.png)

树形dp，从根节点的角度出发思考问题
父节点是直接上属，子节点是直接下属。直接上属参加了舞会，直接下属就不参加。直接上属不参加舞会，直接下属可以参加也可以不参加

状态表示：
从节点的角度出发，以当前节点是否参加舞会作为状态。两个维度限制节点，一个是唯一表示节点的编号，一个是是否参加舞会的标识。用1表示当前节点参加舞会，0表示不参加舞会
属性：当前节点参加舞会与不参加舞会的最大快乐指数
所以$f(i, 0)$表示第i个节点不参加舞会的最大快乐指数，$f(i, 1)$表示第i个节点参加舞会的最大快乐指数

状态计算：题意很明确，第i个节点参加舞会，其子节点不参加舞会
`f(i, 1) += f(k, 0)`，k为i的所有子节点枚举，最后`f(i, 1) += hpy[i]`加上自己的快乐指数
第i个节点不参加舞会，其子节点可参加也可不参加
`f(i, 1) += max(f(k, 0), f(k, 1))`，k为i所有子节点的枚举，从子节点参加与不参加舞会的最大快乐指数中取较大值，最后也要加上自己的快乐指数

由于父节点的状态计算需要用到子节点的状态，所以子节点的状态需要先计算出来，需要使用后序的方式遍历树
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 6010;
int hpy[N], f[N][2];
int h[N], e[N], ne[N], idx = 1;
bool st[N];

void add(int x, int y)
{
    e[idx] = y, ne[idx] = h[x], h[x] = idx ++ ;
}

void dfs(int x)
{
    f[x][1] = hpy[x];
    
    for (int i = h[x]; i != -1; i = ne[i])
    {
        int y = e[i];
        dfs(y);
        f[x][1] += f[y][0];
        f[x][0] += max(f[y][0], f[y][1]);
    }
}

int main()
{
    memset(h, -1, sizeof(h));
    int n;
    scanf("%d", &n);
    for (int i = 1; i <= n; ++ i ) scanf("%d", &hpy[i]);
    
    int x, y;
    for (int i = 1; i < n; ++ i )
    {
        scanf("%d%d", &x, &y);
        add(y, x);
        st[x] = true;
    }
    int root = 1;
    while (st[root]) root ++ ;
    
    dfs(root);
    
    printf("%d\n", max(f[root][0], f[root][1]));
    return 0;
}
```

## 记忆化搜索
> 递归实现动态规划
### 901. 滑雪
[901. 滑雪 - AcWing题库](https://www.acwing.com/problem/content/903/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230721222347.png)

题意很简单，直接分析
状态表示：
集合：从某个点开始滑雪，所有的滑雪轨迹，用两个维度表示一个点
属性：最长的滑雪轨迹
所以$f(i, j)$表示从以(i, j)为起点，所有滑雪轨迹中的最大值

状态计算：如何划分$f(i, j)$？以(i, j)四个方向上相邻的点作为起点，所有的滑雪轨迹
相邻的点满足小于关系时，所有相邻点表示的集合不重不漏地组成$f(i, j)$
取$f(i, j)$子集中的max，加上1就能得到$f(i, j)$

问题是计算$f(i, j)$要知道其相邻点的状态，如$f(i-1, j)$，但是按照迭代的方式更新f数组，先计算的是$f(i, j)$，后计算的是$f(i-1, j)$。此时计算$f(i, j)$需要的状态没有被计算出来
可以用递归的方式更新f数组，类似树的后续遍历。先更新边界情况再往中间更新

由于题目给定的是一张拓扑图，即有向无环图，所以这里不需要使用访问数组也不需要担心因为环路导致死循环
```cpp
#include <iostream>
#include <cstring>
using namespace std;

const int N = 310;
int g[N][N], f[N][N];
int n, m;
int dx[4] = {0, 1, 0, -1}, dy[4] = {1, 0, -1, 0};

int dp(int x, int y)
{
    int &u = f[x][y];
    if (u != -1) return f[x][y];
    u = 1;
    
    for (int i = 0; i < 4; ++ i )
    {
        int nx = x + dx[i], ny = y + dy[i];
        if (nx >= 1 && nx <= n && ny >= 1 && ny <= m && g[x][y] > g[nx][ny])
            u = max(u, dp(nx, ny) + 1);
    }
    return u;
}

int main()
{
    memset(f, -1, sizeof(f));
    scanf("%d%d", &n, &m);
    for (int i = 1; i <= n; ++ i )
        for (int j = 1; j <= m; ++ j )
            scanf("%d", &g[i][j]);
            
    int res = 0;
    for (int i = 1; i <= n; ++ i )
        for (int j = 1; j <= m; ++ j )
            res = max(res, dp(i, j));
    printf("%d\n", res);
    
    return 0;
}
```