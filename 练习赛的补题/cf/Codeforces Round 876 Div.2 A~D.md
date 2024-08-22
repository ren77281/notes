```toc
```
## 

将n拆分两半，n/2下取整得到t，t/k上取整得到u，2 k u >= n即可，答案为2u，若小于则2u+1

5/2 2 t
2/2 1 u
2 1 2 4
3/2 1 t
1/2 1 u
2 2 1in
***
## B. Lamps
[Problem - B - Codeforces](https://codeforces.com/contest/1839/problem/B)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230824172224.png)


```cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

typedef long long LL;
int T, n, x, y;

int main()
{
    ios::sync_with_stdio(false), cin.tie(0), cout.tie(0);
    cin >> T;
    while ( T -- )
    {
        cin >> n;
        vector<vector<int>> a(n + 1);
        for (int i = 0; i < n; ++ i )
        {
            cin >> x >> y;
            a[x].push_back(y);
        }
        
        LL ans = 0;
        for (int i = 1; i <= n; ++ i )
        {
            sort(a[i].begin(), a[i].end(), greater<int>());
            for (int j = 0; j < min(i, (int)a[i].size()); ++ j )
                ans += a[i][j];
        }
        cout << ans << endl;
    }
    return 0;
}
```
debug：一直TLE，思路没问题，细节没问题不会死循环
最后发现是`vector<int> a[N]`，T为2e4时，每次都初始化N（2e5+10）的vector矩阵，N太大了导致TLE
看来多组测试用例时，千万不要静态地开N啊，能用STL就尽量用
搞了我很久
***
## C
若最后为1，输出NO
遇到1cnt++，直到遇到0，此时输出cnt，再输出cnt个0
重复以上步骤
```cpp
#include <iostream>
#include <vector>
using namespace std;

const int N = 2e5 + 10;
int a[N], T, n;

int main()
{
    ios::sync_with_stdio(false);
    cin.tie(0), cout.tie(0);
    cin >> T;
    while ( T -- )
    {
        cin >> n;
        for (int i = 0; i < n; ++ i ) cin >> a[i];
        if (a[n - 1] == 1) cout << "NO\n";
        else
        {
            vector<int> b;
            cout << "YES\n";
            for (int i = 0, cnt = 0; i < n; ++ i )
            {
                while (i < n && a[i] != 0) cnt ++ , i ++ ;
                b.push_back(cnt);
                for (int j = 0; j < cnt; ++ j ) b.push_back(0);
                cnt = 0;
            }
            for (int i = n - 1; i >= 0; -- i ) cout << b[i] << ' ';
            cout << endl;
        }
    }
    return 0;
}
```
***
## D. Ball Sorting
