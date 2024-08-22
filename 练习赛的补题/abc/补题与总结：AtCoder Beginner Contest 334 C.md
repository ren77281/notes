## C - Socks 2（贪心 暴力优化）
[C - Socks 2 (atcoder.jp)](https://atcoder.jp/contests/abc334/tasks/abc334_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202312241611320.png)

贪心：若存在两只相同颜色的袜子，则直接配对。如剩下的袜子颜色为a, b, b, c（a到c的值依次增大），最佳配对方式有两种：(b - a) + (c - b)以及(b - b) + (c - a)，两种方式的差异值相同，区别在于是否将相同颜色的袜子配对在一起，所以可以证明：是否将相同颜色的袜子配对在一起对结果没有影响。则只考虑不同颜色的袜子，减少思维量

对于颜色不同的袜子，若它们的颜色为a, b, c, d ...，则最佳配对方式的差异值为：(b - a) + (d - c), ...，可以证明其他配对方式的差异值一定大于上述方式

若颜色不同袜子的数量为偶数，直接计算差异值输出即可
若为奇数，则要考虑哪只袜子不会被使用。若$O(n)$暴力考虑，每次都需要重新计算差异值，总时间复杂度为$O(n^2)$。此时考虑优化，(a, b, c, d, e)的最终配对序列可能为
(b - a)(d - c)，不使用e
(c - b)(e - d)，不使用a
(b - a)(e - d)，不使用c
可以证明不使用b和d一定比上述序列的差异值大，综上我们只要考虑不使用第奇数个袜子的最终序列
可以注意到最终序列的差异值将多次用到相邻两个袜子的差异值，将这些值预处理成前缀和数组，即可$O(1)$得到最终差异值，在这些差异值中取min即为答案

对于差异值的预处理，我分别预处理了一个前缀和与一个后缀和数组，具体处理方式因人而异，明白思路即可：
```cpp
#include <bits/stdc++.h>
using namespace std;

typedef long long LL;
const int N = 3e5 + 10;
LL a[N], pre[N], suf[N];

void solve()
{
    LL n, k; cin >> n >> k;
    for (int i = 1; i <= k; ++ i) cin >> a[i];
    for (int i = 1; i <= k; ++ i)
    {
        pre[i] = pre[i - 1];
        if (i % 2 == 0) pre[i] += a[i] - a[i - 1];
    }
    for (int i = k; i >= 1; -- i)
    {
        suf[i] = suf[i + 1];
        if (i % 2 == 0) suf[i] += a[i + 1] - a[i];
    }
    if (k % 2 == 0) cout << pre[k] << "\n";
    else
    {
        LL ans = INF;
        for (int i = 1; i <= k; i += 2)
            ans = min(ans, pre[i - 1] + suf[i + 1]);
        cout << ans << "\n";
    }
}

int main()
{
    ios::sync_with_stdio(false), cin.tie(0),cout.tie(0);
    solve();
    return 0;
}
```


