```toc
```
## 迭代加深
搜索树中，某些分支的深度可能非常深，然而搜索的答案却在一个非常浅的地方，此时搜索较深的分支将浪费多余的时间。对于这个问题，可以用迭代加深进行优化
每次指定一个maxdepth，dfs时超过maxdepth时就停止搜索，即每次dfs时指定了一片搜索区域，只搜索区域中的数据。注意，迭代加深只能在答案位于较浅的位置时使用

每次更新maxdepth后再次进行dfs时，都会重复搜索上一个搜索区域，是否会浪费时间？在极端情况下，上一个搜索区间中的节点数量要远小于新增搜索区域的节点数量，那么对于总的时间来说，这些时间可以忽略不计
***
### 170. 加成序列
[170. 加成序列 - AcWing题库](https://www.acwing.com/problem/content/172/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230818211756.png)

加成序列，第一位为1，最后一位为n，n最大为100。从第二个数开始，每一个数必须由之前的数相加得到，题目要求输出最短的加成序列，那么每一位的数都要尽可能的大。最大的情况是：第i个数为第i-1个数的两倍。这样枚举一下，可以发现答案的加成序列长度很短，满足迭代加深的使用场景
所以这题从长度为2的加成序列开始搜索，搜索失败时再加深长度
搜索序列中的每个数，数越大，后续的搜索分支就越少，因为终点确定的加成序列将更短，所以考虑到搜索顺序的优化，每次选择最大的数
若当前确定的数等于n，表示搜索到了结果，搜索结束
若当前确定的数大于n，后续的搜索无意义，重新确定当签数
若当前确定的数小于其前一个数，不满足加成序列的性质，也重新确定当前数
确定第i个数时，需要枚举前i-1个数，两两相加可能有相同的结果，相同数不用枚举，所以也重新确定当前数

```cpp
#include <iostream>
using namespace std;

const int N = 110;
int p[N], n;

bool dfs(int cur, int k)
{
    if (cur > k) return p[cur - 1] == n;
    bool st[N]= {0};
    for (int i = cur - 1; i >= 1; -- i )
        for (int j = i; j >= 1; -- j )
        {
            int t = p[i] + p[j];
            if (st[t] || t > n || t <= p[cur - 1]) continue;
            p[cur] = t;
            st[t] = true;
            if (dfs(cur + 1, k)) return true;
        }
    return false;
}

int main()
{
    p[1] = 1;
    while (cin >> n, n)
    {
        int k = 1;
        while (!dfs(2, k)) k ++ ;
        for (int i = 1; i <= k; ++ i ) cout << p[i] << ' ';
        cout << endl;
    }
    return 0;
}
```
debug：st数组开在函数中需要初始化，之前都是开全局变量不用初始化
加成序列的长度要从1开始枚举，若从2开始枚举，当答案要求输出长度为1的加成序列时，代码会TLE
***
### 双向dfs：171. 送礼物
[171. 送礼物 - AcWing题库](https://www.acwing.com/problem/content/173/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230819084906.png)

直接爆搜，每个物品有选或不选两种选择，时间复杂度为$2^n$，虽然能加入剪枝优化，但是依然会超时
将物品分成差不多的两堆，分两次进行dfs，即双向dfs（或许叫二次dfs更贴切些？）
空间换时间，第一次将搜索结果保存下来，将第二次dfs的结果与第一次的搜索结果相加，维护出一个不超过W的最大值
假设用weight数组保存第一次dfs的结果，根据第二次dfs的结果，在weight数组中二分出ans，使得ans + 第二次dfs的结果 <= W
这道题的n最大为46，这题的时间为$2^{23} + 2^{23}log2^{23}$，即$2^{23} + 23 * 2^{23}$

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230819073838.png)

```cpp
#include <iostream>
#include <algorithm>
using namespace std;

typedef long long LL;
const int N = 50;
LL w[N], n, W, ans;
int k, weight[1 << 27], idx;

void dfs1(int cnt, LL s) // cnt为已进行的选择次数，s为当前重量
{
    if (s > W) return;
    if (cnt == k) 
    {
        weight[idx ++ ] = s;
        return;
    }
    
    dfs1(cnt + 1, s);
    dfs1(cnt + 1, s + w[cnt]);
}

void dfs2(int cnt, LL s)
{
    if (s > W) return;
    if (cnt == n)
    {
        int l = 0, r = idx - 1;
        while (l < r)
        {
            int mid = l + r + 1 >> 1;
            if (weight[mid] + s <= W) l = mid;
            else r = mid - 1;
        }
        if (weight[l] + s <= W) ans = max(ans, weight[l] + s);
        return;
    }
    dfs2(cnt + 1, s);
    dfs2(cnt + 1, s + w[cnt]);
}

int main()
{
    cin >> W >> n;
    k = n / 2;
    for (int i = 0; i < n; ++ i ) cin >> w[i];
    sort(w, w + n, greater<int>());
    dfs1(0, 0);
    sort(weight, weight + idx);
    
    dfs2(k, 0);
    cout << ans << endl;
    return 0;
}
```
用vector存储weight的话，会TLE
***
## IDA*
IDA\*通常和迭代加深一起使用，每次dfs时有一个maxdepth，对于当前搜索到的节点来说，预估该节点到答案需要走的步数，若当前走的步数加上预估步数超过mandepth，则停止搜索
所以，IDA\*其实就是搜索过程中的一种特殊剪枝。本质和A\*一样，对于每个节点保证估计值小于等于真实值
***
### 180. 排书
[180. 排书 - AcWing题库](https://www.acwing.com/problem/content/182/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230819094111.png)

考虑已经排好序的数组中，每个数之间的关系，用于数组长度为n，且每个数都是1~n之间的数，所以对于排好序的数组中的任意一个数来说，i的后面一定是i+1
这样的关系，长度为n的数组有n-1个
将数组中$[a_l, a_r]$之间的数移动到$a_k$后面，将改变三个数的后继关系：$a_{l-1}, a_r, a_k$
若给定的数组中有k个错误的关系，那么至少要移动$k / 3$上取整次，即$(k + 2) / 3$下取整
因为这个值是最少需要搜索的次数，一定小于真实搜索的次数，所以能将其作为估计值
若当前的搜索次数加上估计值大于maxdepth，停止搜索

每次搜索枚举所有能移动的区间，由区间长度从小到大枚举
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230819094043.png)
