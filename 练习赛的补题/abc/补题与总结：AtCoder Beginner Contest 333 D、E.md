## 写在最前面的复盘
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202312182207273.png)
前三题属于是凑数题，下次争取快点a掉，这次wa了一次
C题写了个三指针，从小到大枚举出满足题意的数，其实可以直接暴力枚举满足题意的数，但是会有重复的，用set去重即可，赛时没想到，三指针磨了很久。原来暴力也是门艺术，什么时候适合暴力也是门学问啊，自己对于这块的理解确实不够深
以为D题读懂了题意，然后写写写，debug了很久，赛后重写了一遍，也就只有个dfs，5分钟写完+debug就过了。现在想想应该是刚打完acc的影响，脑子转不动了。还有就是题意没有想清楚就开写，导致最后一直debug
E题也想到了贪心，但是题意理解做了（又读了假题），想了一个巨复杂的贪心，看着ac人数有点多，发现不对劲。但是已经被假贪心摧残，脑子转不动，有心无力直接下班

最后只能说，以后先读懂题，再开始头脑风暴，读假题真的难绷
***
## D - Erase Leaves
[D - Erase Leaves (atcoder.jp)](https://atcoder.jp/contests/abc333/tasks/abc333_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202312182137144.png)

统计以x为根节点的子树中，节点的数量（包括x自己），用cnt数组存储
对于1号节点的所有子节点`i, i + 1, ..., j - 1, j`，计算`cnt[i] + cnt[i + 1] + ... + cnt[j - 1] + cnt[j]`，最后删去最大的cnt即为答案

cnt数组用dfs就能求出
```cpp
#include <bits/stdc++.h>
using namespace std;

const int N = 3e5 + 10;
vector<int> g[N];
int cnt[N];
bool st[N];

int dfs(int x)
{
    st[x] = true;
    for (auto y : g[x])
    {
        if (!st[y]) cnt[x] += dfs(y);
    }
    return ++ cnt[x];
}

void solve()
{
    int n; cin >> n;
    for (int i = 0; i < n - 1; ++ i)
    {
        int x, y; cin >> x >> y;
        g[x].push_back(y), g[y].push_back(x);
    }
    dfs(1);
    int mx = 0, ans = 0;
    for (auto y : g[1])
        ans += cnt[y], mx = max(mx, cnt[y]);
    cout << ans - mx + 1 << "\n";
}

int main()
{
    ios::sync_with_stdio(false), cin.tie(0),cout.tie(0);
    solve();
    return 0;
}
```
***
## E - Takahashi Quest
[E - Takahashi Quest (atcoder.jp)](https://atcoder.jp/contests/abc333/tasks/abc333_e)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202312182147470.png)

贪心思路：对于每只怪物，只拾取相应类型且离其最近的药水
最后要输出的"拾取序列"比较难模拟，可以反着考虑，从后往前遍历
记录未被击败的不同类型怪物出现次数，遇到药水时，若对应类型的怪物未被击败，那么拾取该药水，否则不拾取
最后判断是否还有怪物未被击败
```cpp
#include <bits/stdc++.h>
using namespace std;

typedef pair<int, int> PII;
typedef long long LL;
typedef unsigned long long ULL;
const int inf = 2e9 + 10;
const LL INF = 4e18 + 10;
const int mod9 = 998244353;
const int mod7 = 1e9 + 7;
const int N = 3e5 + 10;
PII a[N];

void solve()
{
    int n; cin >> n;
    for (int i = 1; i <= n; ++ i) cin >> a[i].first >> a[i].second;
    unordered_map<int, int> mp;
    vector<int> ans;
    // 倒着枚举
    for (int i = n; i >= 1; -- i)
    {
        if (a[i].first == 2) mp[a[i].second] ++ ;
        else if (mp[a[i].second] > 0) mp[a[i].second] -- , ans.push_back(1);
        else ans.push_back(0);
    }
    for (auto t : mp)
        if (t.second > 0)
        {
            cout << "-1\n";
            return;
        }
    
    reverse(ans.begin(), ans.end()); 
    int i = 0, j = 1, k = 0, cur = 0;
    // 统计最大K
    while (j <= n)
    {
        if (a[j].first == 1 && ans[i ++ ] == 1) cur ++ ;
        else if (a[j].first == 2) cur -- ;
        j ++ ; 
        k = max(k, cur);
    }
    cout << k << "\n";
    for (auto x : ans) cout << x << ' ';
}

int main()
{
    ios::sync_with_stdio(false), cin.tie(0),cout.tie(0);
    solve();
    return 0;
}
```