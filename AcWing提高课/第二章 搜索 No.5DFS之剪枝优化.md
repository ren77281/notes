```toc
```

1. 优化搜索顺序，优先搜索节点较少的分支
2. 排除等效冗余，组合型搜索
3. 可行性剪枝，当前方案不合法，停止搜索
4. 最优性剪枝，当前方案不是最优解，停止搜索
### 165. 小猫爬山
[165. 小猫爬山 - AcWing题库](https://www.acwing.com/problem/content/167/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230817093603.png)

每只小猫都有两种选择，加入已经有猫的车和加入没有猫的车
每次dfs枚举每只小猫的两种选择，当枚举完小猫时更新ans即可
思考如何优化：
优化搜索顺序：如果先放重量轻的小猫，将剩余更多的空间，枚举其他小猫时dfs的分支更多。所以可以先放重量重的小猫，减少后续dfs的分支
最优性剪枝：ans最初为正无穷，之后将被更新，逼近最优解。当使用的车的数量超过ans时，说明此时的方案一定不是最优解，停止搜索

dfs的参数，gcnt表示已经使用的车的数量，cnt表示已经坐上车的小猫数量
这题不用开st数组判重，因为我们的搜索方向决定了所有小猫将不重不漏地被搜索到，也就是每次的dfs都会使得一只小猫坐上车，只要按照小猫顺序dfs即可
```cpp
#include <iostream>
#include <algorithm>
using namespace std;

const int N = 20;
int g[N], w[N];
int n, W, ans = N;

void dfs(int gcnt, int cnt)
{
    if (gcnt >= ans) return;
    if (cnt == n) 
    {
	    ans = min(ans, gcnt);
	    return;
	}
    // 坐已经有猫的车
    for (int i = 0; i < gcnt; ++ i )
        if (g[i] + w[cnt] <= W)
        {
            g[i] += w[cnt];
            dfs(gcnt, cnt + 1);
            g[i] -= w[cnt];
        }
        
    // 坐没有猫的车
    g[gcnt] += w[cnt];
    dfs(gcnt + 1, cnt + 1);
    g[gcnt] -= w[cnt];
}

int main()
{
    cin >> n >> W;
    for (int i = 0; i < n; ++ i ) cin >> w[i];
    
    sort(w, w + n, greater<int>());
    dfs(0, 0);
    cout << ans << endl;
    return 0;
}
```
这题和[1118. 分成互质组 - AcWing题库](https://www.acwing.com/problem/content/1120/)这题很像，但是用分成互质组的解法不能ac这道题
分成互质组的解法中，每次dfs都是组合型枚举特定范围内的数，将能放入最后一组的数加入最后一组。只有范围内的所有数都无法放入最后一组时，才会新开一组，并且试着放入剩下的数。这样的做法是贪心，需要经过证明，并且思路与证明比较难想
而小猫这题的思路就是一种比较通用的解法，按顺序枚举所有猫，每次有两种选择：1. 将猫放入有猫的车上 2. 将猫放入没有猫的车上。这样子枚举能保证dfs至少是不遗漏的
当然，分成互质组也能使用这样通用的解法，以下是ac代码
```cpp
#include <iostream>
#include <vector>
using namespace std;

const int N = 15;
int a[N], n, ans = N;
vector<vector<int>> g;

int gcd(int a, int b)
{
    return b ? gcd(b, a % b) : a;
}

bool check(int gi, int x)
{
    for (int i = 0; i < g[gi].size(); ++ i )
    {
        if (gcd(g[gi][i], x) > 1) 
            return false;
    }
    return true;
}

void dfs(int gcnt, int cnt)
{
    if (gcnt >= ans) return;
    if (cnt == n) 
    {
        ans = gcnt;
        return;
    }
    
    for (int i = 0; i < gcnt; ++ i )
    {
        if (check(i, a[cnt]))
        {
            g[i].push_back(a[cnt]);
            dfs(gcnt, cnt + 1);
            g[i].pop_back();
        }
    }
    g[gcnt].push_back(a[cnt]);
    dfs(gcnt + 1, cnt + 1);
    g[gcnt].pop_back();
}

int main()
{
    cin >> n;
    for (int i = 0; i < n; ++ i ) cin >> a[i];
    g.resize(N);
    dfs(0, 0);
    cout << ans << endl;
    return 0;
}
```
***
### 166. 数独
[166. 数独 - AcWing题库](https://www.acwing.com/problem/content/168/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230817143622.png)

先考虑如何搜索能不遗漏的搜索完所有情况：每次搜索空着的格子，枚举该格子可以填的所有数即可
以上过程可以剪枝，首先可行性剪枝很简单，空着的格子无法填任何数时，停止搜索即可
其次是搜索顺序，若一个格子能填入的数多，那么它的dfs分支就多，应该放到dfs的后面搜索。所以我们应该枚举从能填入的数少的格子开始枚举

用状态压缩表示每个格子的状态：题目要求每一行，每一列，每一个九宫格中都不能出现重复的数字。用`row[N]，col[N]，cell[3][3]`分别表示行，列，九宫格的状态，每个元素表示一行，一列与一个单元格。每个元素中，1表示可以填入数值为该下标+1的数，0表示已经有了这样的数，不能填入
对于空着的格子，每次将者三个状态做&运算，即只有在三个状态下都能填入的数，才能填入这个位置

初始化时，将三个数组的所有数的0~8位设置为1，即(1 << 9) - 1
接着根据题目输入的原始矩阵，将出现的数字在对应数组中的对应位置设置为0，这一步不用位运算，直接减去就行
```cpp
#include <iostream>
using namespace std;

const int N = 9, M = 1 << N;
int row[N], col[N], cell[3][3];
int mp[M]; // log2，下标为2的i次方时，存储i
int ones[M]; // i的二进制表示中有几个1
string str;

void init()
{
    for (int i = 0; i < N; ++ i ) 
        row[i] = col[i] = (1 << N) - 1;
    for (int i = 0; i < 3; ++ i )
        for (int j = 0; j < 3; ++ j )
            cell[i][j] = (1 << N) - 1;
}

void draw(int x, int y, int t, bool flag) // flag为true表示将这个格子设置为数字，否则设置为空格
{
    // 维护矩阵
    if (flag) str[x * N + y] = '1' + mp[t];
    else str[x * N + y] = '.';
    
    // 维护三个数组
    if (!flag) t = -t;
    row[x] -= t;
    col[y] -= t;
    cell[x / 3][y / 3] -= t;
}

int lowbit(int x)
{
    return x & -x;
}

int get(int x, int y)
{
    return row[x] & col[y] & cell[x / 3][y / 3];
}

bool dfs(int cnt)
{
    if (cnt == 0) return true;
    // 找能填的数最少的格子
    int x, y, t = 10;
    for (int i = 0; i < N; ++ i )
        for (int j = 0; j < N; ++ j )
            if (str[i * 9 + j] == '.')
            {
                int state = get(i, j);
                if (t > ones[state])
                {
                    t = ones[state];
                    x = i, y = j;
                }
            }
        
    int state = get(x, y);
    for (int i = state; i; i -= lowbit(i))
    {
        draw(x, y, lowbit(i), true); // 填入数字
        if (dfs(cnt - 1)) return true;
        draw(x, y, lowbit(i), false); // 没有成功就恢复空格
    }
    
    return false;
}

int main()
{
    for (int i = 0; i < 1 << N; ++ i )
        for (int j = 0; j < N; ++ j )
            ones[i] += (i >> j) & 1;
    for (int i = 0; i < N; ++ i )
        mp[1 << i] = i;
        
    while (cin >> str, str[0] != 'e')
    {
        init();
        int cnt = 0; // 需要填的格子数量
        for (int i = 0, k = 0; i < N; ++ i )
            for (int j = 0; j < N; ++ j, ++ k )
            {
                if (str[k] != '.')
                {
                    int t = 1 << (str[k] - '1');
                    draw(i, j, t, true);
                }
                else cnt ++ ;
            }
        
        dfs(cnt);
        cout << str << endl;
    }
    return 0;
}
```
吐槽一下：这题思路比较好想，就是写完脑袋里全是0和1
***
### 167. 木棒
[167. 木棒 - AcWing题库](https://www.acwing.com/problem/content/169/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230817194229.png)

题目说砍断木棒后，木棍的长度一定小于等于50，并没有说原来的木棒长度大于50

题目要求计算最小的木棒长度，这题的搜索思路可以这样：组合式地从小到大枚举所有可能的木棒长度。当然，所有木棍的总长度之和必须时木棒长度的倍数
假设每次枚举的木棒长度为length，每次dfs选择木棍组成木棒。考虑搜索顺序，每次选择长度最长的木棍可以减少后续的搜索分支。所以选择长度最长的木棍拼接木棒
当拼接长度小于length时，进行拼接。拼接长度大于length时，选择更小的木棍进行拼接。当拼接长度等于length时，拼接下一个木棒。若最后恰好使用完所有的木棍，并且拼接好了所有的木棒，说明此时搜索到了最优解

以上，有三种剪枝：搜索顺序，可行性剪枝以及排除等效冗余。其中排除等效冗余包括了组合型枚举，此外还有三种排除等效冗余的剪枝
1. 当使用长度为len的木棍拼接长度为length的木棒失败时，对于这个木棒，不用再选择长度为len的木棍拼接
2. 当使用长度为len的木棍作为第一根木棍拼接长度为length的木棒失败时，说明无法拼接长度为length的木棒，直接枚举其他length即可
3. 当使用长度为len的木棍作为最后一根木棍拼接长度为length的木棒失败时，说明无法拼接长度为length的木棒，直接枚举其他length即可

后两个剪枝比较相似，证明也比较相似：反证，若使用长度为len的木棍作为第一根木棍拼接某个长度为length的木棒失败了，但是使用长度为len的木棍可以成功地拼接其他木棒，并且得到了最优解
证明，可以这样操作，一个木棒中，交换两个木棍位置，木棒的长度不变。长度为len的木棍一定在某个木棒中，将其与交换第一个位置的木棍，木棒长度不变，但是出现了“将len作为第一根木棍并且得到最优解”，由于前提是“将len作为第一根木棍无法拼接长度为length的木棒”，出现矛盾，所以假设不成立
```cpp
#include <iostream>
#include <cstring>
#include <algorithm>
using namespace std;

const int N = 70;
int w[N], n, sum, length;
bool st[N];

bool dfs(int cnt, int curl, int start)
{
    if (cnt * length == sum) return true;
    if (curl == length) return dfs(cnt + 1, 0, 0);
    
    for (int i = start; i < n; ++ i )
    {
        if (st[i] || w[i] + curl > length) continue;
        
        st[i] = true;
        if (dfs(cnt, curl + w[i], i + 1)) return true;
        st[i] = false;
        
        if (!curl || curl + w[i] == length) return false;
        
        int j = i;
        while (j < n && w[j] == w[i]) j ++ ;
        i = j - 1;
    }
    return false;
}

int main()
{
    while (cin >> n, n)
    {
        sum = 0, length = 1;
        for (int i = 0; i < n; ++ i) 
        {
            cin >> w[i];
            sum += w[i];
        }
        sort(w, w + n, greater<int>());
        
        while (true) // 题目一定有解
        {
            if (sum % length == 0 && dfs(0, 0, 0)) // 已经拼接的木棒数量，当前木棒的长度，组合型枚举
            {
                memset(st, false, sizeof st);
                cout << length << endl;
                break;
            }
            length ++ ;
        }
    }
    
    return 0;
}
```
debug：每次枚举length进行dfs之前，都要重置st数组
***
### 168. 生日蛋糕
[168. 生日蛋糕 - AcWing题库](https://www.acwing.com/problem/content/description/170/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230818081920.png)
题目给定N，表示蛋糕的体积为Nπ，所以在计算体积和表面积时，我们可以忽略π

题目要求下面的蛋糕高度和半径要严格大于上面的蛋糕高度和半径，所以下面的蛋糕高度和半径是最大的，考虑搜索顺序的优化，从下面的蛋糕开始枚举所有可能的半径和高度
这样枚举不会出现等效冗余的情况，所以不用思考这方面的剪枝

接着是可行性剪枝，假设自顶向下的蛋糕编号为1~m
对于$R_i$，$i <= R_i <= R_{i+1}-1$，最小的情况是每层的半径都比下一层的半径少1，那么第i层的半径最小为i。同时$R_i$能取到的最大值要比下一层的半径少1，此外题目还限制了总体的体积为n，因为是从下到上枚举的，假设第i+1层到第m层的体积为$VS_{i+1}$，那么$R_i * R_i * H_i <= n - VS_{i+1}$，即$R_i <= sqrt((n - VS_{i+1}) / H_i)$，将$H_i$取$i$，因为第i层的H最小值为i，H取最小，R就能取最大
所以对于$R_i$，$i <= R_i <= min(R_{i+1}-1, sqrt(n-VS_{i+1}/i)))$

除了$R_i$还需要枚举$H_i$，显然$i <= H_i <= H_{i+1}-1$
由于体积的限制$R_i * R_i * H_i <= n - VS_{i+1}$，那么$H_i <= (n - VS_{i+1})/R_i^2$
由于我们先枚举出了$R_i$，所以这里的$R_i$就不用取1了
对于$H_i$，$i <= H_i <= min(H_{i+1}-1, (n-VS_{i+1})/R_i^2)$

在这两个范围内枚举$R_i$和$H_i$，然后考虑最优性剪枝，题目要返回最小的忽略π的外表面面积，假设ans为dfs过程中最优解，若当前（第i+1层到第m层）的外表面面积+前i层的外表面面积的最小值 >= ans，那么停止搜索。同理，若当前（第i+1层到第m层）的体积+前i层的体积的最小值 > n，那么停止搜索


![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230818075348.png)

