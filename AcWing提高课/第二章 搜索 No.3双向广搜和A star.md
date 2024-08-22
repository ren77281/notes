```toc
```

## 双向广搜
双向广搜一般应用于最小步数模型，Floor Fill和最短路模型中不会使用，因为这两个模型的搜索范围比较小，直接bfs就能过掉。而最小步数模型的空间增长为指数级别，很容易爆空间，所以经常采用双向广搜优化
### 190. 字串变换
[190. 字串变换 - AcWing题库](https://www.acwing.com/problem/content/192/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230816074456.png)

最小步数模型，考虑最坏情况的时间
由于字符串最长有20个字符，假设每次的变换都能选择其中10个位置进行变换。每次从6种变换中选择1种进行变换，题目限制最多进行10次变换，最坏情况下的bfs将进行$60^{10}$次变换
肯定超时+爆空间，因此使用双向广搜进行bfs优化

维护两个队列，从起点和终点各自开始bfs，每次选择长度较小的队列进行bfs，每次bfs都要完整地扩展一层再停止
第一次搜索到了对方搜索过的点，说明找到最短路径

另外，这题需要匹配字符串的子串以进行变换，可以使用`substr`暴力遍历字符串每个位置，判断是否相等以进行替换
```cpp
#include <iostream>
#include <cstring>
#include <string>
#include <queue>
#include <unordered_map>
using namespace std;

unordered_map<string, int> da, db;
queue<string> qa, qb;
const int N = 7;
string start, ed;
string a[N], b[N];
int n;

// 将q队列向外扩展一层
int extend(queue<string> &q, unordered_map<string, int> &da, unordered_map<string, int> &db, string a[], string b[])
{
    int d = da[q.front()]; // 保证每次遍历一层
    while (q.size() && da[q.front()] == d)
    {
        auto t = q.front();
        q.pop();
        for (int i = 0; i < n; ++ i )
            for (int j = 0; j + a[i].size() <= t.size(); ++ j ) // 从j开始匹配规则
                if (t.substr(j, a[i].size()) == a[i]) 
                {
                    string u = t.substr(0, j) + b[i] + t.substr(j + a[i].size());
                    if (db.count(u)) return da[t] + db[u] + 1;
                    if (!da.count(u))
                    {
                        da[u] = da[t] + 1;
                        q.push(u);
                    }
                }
    }
    return 11;
}

int bfs()
{
    if (start == ed) return 0;
    qa.push(start), qb.push(ed);
    da[start] = 0, db[ed] = 0;
    int step = 0;
    while (qa.size() && qb.size())
    {
        int t;
        if (qa.size() > qb.size()) t = extend(qb, db, da, b, a);
        else t = extend(qa, da, db, a, b);
        if (t <= 10) return t;
        if ( ++ step == 10 ) return 11;
    }
    return 11;
}

int main()
{
    cin >> start >> ed;
    while (cin >> a[n] >> b[n]) n ++ ;
    
    int t = bfs();
    if (t <= 10) cout << t << endl;
    else puts("NO ANSWER!");
    return 0;
}
```
debug：extend时，不能在取出队头元素后判断其是否被另一队列搜索过，必须在每次向外扩展后就进行判断。因为每次的扩展只会扩展一层，第一种判断方式只有在下次的扩展中才会生效，若下次的扩展strp == 11，那么第一种判断将不会被执行，所以只能采用第二种判断
由于采用第二种判断，每次扩展后都进行判断，此时将漏掉一开始start和ed相等的情况，所以还必须加上特判
***
## A*
将bfs的队列换成优先队列（小根堆），元素存储真实距离和估计距离，用两者之和来排序，每次取出队头，当取出终点后break，此时的`dis[终点]`一定是最小值
A\*算法可以处理边权非负的图，不只是边权相等的图
可以将Dijkstra看成一种特殊的A\*算法，即所有估计距离都取0的A\*算法，但是两者的本质，思想是不一样的

A\*算法需要满足的条件：假设当前点为i，`d[i]`为起点到i的真实距离，`g[i]`为i到终点的真实距离，`f[i]`为i到终点的估计距离，必须满足`f[i] <= g[i]`
如果起点和终点不连通，不能使用A\*，因为其效率比朴素bfs更低

反证法证明A\*正确性：
假设第一次取出终点T时，`dis[T]`不是最小值，那么`dis[T] > 最优解` ，说明队列中存在一个最优解，解释最优解经过u，那么`d[u] + f[u] <= d[u] + g[u] = 最优解`，即`dis[T] > 最优解 >= d[u] + f[u]`，即说明了队列中存在一个元素u，其距离小于当前出队元素T。由于优先队列每次只会取出队列中的最小值，此时的情况与前提矛盾，即假设不成立
注意：第一次出队的值为最小值，只对终点成立，并且除了终点之外每个点不会只被扩展一次
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230816154913.png)

***
### 179. 八数码
[179. 八数码 - AcWing题库](https://www.acwing.com/problem/content/description/181/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230816100823.png)

八数码问题一定有解的充要条件是：逆序对个数为偶数
所以先将矩阵转换成一维数组，统计逆序对的数量，为偶数再继续求解

朴素bfs可能超时，这题可以用双向bfs或者A\*，这里用A\*
最小步数模型需要将状态保存下来，题目描述的状态为二维矩阵，这里将其转换成一维字符串
A\*的队列为优先队列，并且是小根堆，根据与起点的真实距离加上与终点的估计距离作为排序的数值。求估计距离的方法通常只有几种，这里使用曼哈顿距离表示估计距离，具体是当前点的坐标与正确位置的坐标的哈密顿距离
```cpp
#include <iostream>
#include <string>
#include <queue>
#include <cmath>
#include <algorithm>
#include <unordered_map>
using namespace std;

typedef pair<int, string> PIS;
string start, seq, ans;
string ed = "12345678x";
unordered_map<string, int> dis;
unordered_map<string, pair<string, char>> last; // 记录当前状态由上一状态经过操作得到
priority_queue<PIS, vector<PIS>, greater<PIS>> h;
int dx[4] = { 0, 0, 1, -1 }, dy[4] = { 1, -1, 0, 0 };
char op[5] = "rldu";

int f(string& str)
{
    int res = 0;
    // i为正确坐标
    for (int i = 0; i < str.size(); ++ i )
        if (str[i] != 'x')
        {
            int t = str[i] - '1';
            res += abs(i / 3 - t / 3) + abs(i % 3 - t % 3);
        }
    return res;
}

void bfs()
{
    h.push({f(start), start});
    dis[start] = 0;
    while (h.size())
    {
        auto t = h.top();
        h.pop();
        string cur = t.second;
        if (cur == ed) break;
        
        int x, y;
        for (int i = 0; i < cur.size(); ++ i )
            if (cur[i] == 'x')
                x = i / 3, y = i % 3;
        
        string tmp = cur;
        for (int i = 0; i < 4; ++ i )
        {
            int nx = x + dx[i], ny = y + dy[i];
            if (nx < 0 || nx >= 3 || ny < 0 || ny >= 3) continue;
            swap(tmp[x * 3 + y], tmp[nx * 3 + ny]);
            if (!dis.count(tmp) || dis[tmp] > dis[cur] + 1) // 每个点可能被更新多次
            {
                dis[tmp] = dis[cur] + 1;
                h.push({dis[tmp] + f(tmp), tmp});
                last[tmp] = { cur, op[i] };
            }
            swap(tmp[x * 3 + y], tmp[nx * 3 + ny]);
        }
    }

    while (ed != start)
    {
        ans += last[ed].second;
        ed = last[ed].first;
    }
    reverse(ans.begin(), ans.end());
    for (int i = 0; i < ans.size(); ++ i ) printf("%c", ans[i]);
}

int main()
{
    string c;
    while (cin >> c)
    {
        if (c != "x") seq += c;
        start += c;
    }
    
    int t = 0;
    for (int i = 0; i < seq.size(); ++ i )
        for (int j = i + 1; j < seq.size(); ++ j )
            if (seq[i] > seq[j]) t ++ ;
            
    if (t % 2) puts("unsolvable");
    else bfs();
    
    return 0;
}
```
***
### 178. 第K短路
[178. 第K短路 - AcWing题库](https://www.acwing.com/problem/content/180/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230816135750.png)

性质：A\*算法中，终点第i次从队列中取出时，`dis[终点]`为第i小的值

要求起点到终点的第k短路径，只要在A\*中取出k次终点即可
现在的问题是：如何求估计距离？
由于估计距离一定要小于等于真实距离，那么估计距离就可以取一个最小值，即i到终点的最短距离。可以反向建图，用单源最短路Dijkstra求出终点（反向建图后的起点）到所有点的最短距离，以此作为估计距离进行A\*的求解

在八数码问题中，只有遇到还未更新的点或者可以更新成更小值的点，才需要对其进行更新，并且将（实际距离+估计距离）放入优先队列
但是在第k短路的问题中，不应该对更新进行限制。若对更新添加了限制，那么A\*在更新出递达终点的最短路后就会停止更新，因为这些限制将导致更新无法进行
所以在第k短路的问题中，每次搜索到一个点，就需要对其进行更新，将（实际距离+估计距离）放入优先队列。但是直接更新有很多需要注意的细节
不能dis数组保存从起点到其他点的距离，因为每次的更新没有限制，所以无法确定dis中的数据是否为最小值，比如
1->2, 1
1->2, 5
更新完成后，`dis[2]`的值是1还是5？如何确定最小与第k小？
用dis数组无法做到以上的判断，要保证A\*能更新出最小与第k小，需要把每次更新后的dis连同其他信息也保存到优先队列中，保证每次取出的元素都是（真实距离+估计距离）最小并且在前者相同的情况下，真实距离也最小。在这样的前提下，就能保证终点第i次从优先队列中取出的值是第i小的值了
```cpp
#include <iostream>
#include <queue>
#include <cstring>
#include <unordered_map>
using namespace std;

typedef pair<int, int> PII;
typedef pair<int, PII> PIII;
const int N = 1010, M = 2e4 + 10;
int h[N], hs[N], e[M], ne[M], w[M], idx;
int f[N]; bool st[N];
int cnt[N];
int n, m, start, ed, k;

void add(int h[], int x, int y, int d)
{
    e[idx] = y, ne[idx] = h[x], w[idx] = d, h[x] = idx ++ ;
}

void dijkstra()
{
    memset(f, 0x3f, sizeof(f));
    priority_queue<PII, vector<PII>, greater<PII>> q;
    q.push({0, ed});
    f[ed] = 0;
    while (q.size())
    {
        auto t = q.top();
        q.pop();
        int x = t.second;
        if (st[x]) continue;
        st[x] = true;
        for (int i = hs[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (f[y] > f[x] + w[i])
            {
                f[y] = f[x] + w[i];
                q.push({f[y], y});
            }
        }
    }
}

int bfs()
{
    if (f[start] == 0x3f3f3f3f) return -1;
    priority_queue<PIII, vector<PIII>, greater<PIII>> q;
    q.push({f[start], {0, start}});
    
    while (q.size())
    {
        auto t = q.top();
        q.pop();
        int x = t.second.second, d = t.second.first;
        cnt[x] ++ ;
        if (cnt[ed] == k) return d;
        for (int i = h[x]; i != -1; i = ne[i])
        {
            int y = e[i];
            if (cnt[y] < k) // 针对无解情况的剪枝更新
                q.push({d + w[i] + f[y], {d + w[i], y}});
        }
    }
    
    return -1;
}

int main()
{
    memset(h, -1, sizeof(h));
    memset(hs, -1, sizeof(hs));
    scanf("%d%d", &n, &m);
    int x, y, d;
    for (int i = 0; i < m; ++ i )
    {
        scanf("%d%d%d", &x, &y, &d);
        add(h, x, y, d), add(hs, y, x, d);
    }
    scanf("%d%d%d", &start, &ed, &k);
    dijkstra();
    if (start == ed) k ++ ;

    printf("%d\n", bfs());
    return 0;
}
```
debug：需要特判`start == ed`的情况，由于题目要求至少经过一条边，所以起点和终点相等时，第一小的最短路为0，需要排除这种情况，所以此时k ++ 
几天没写dijkstra，就不会写了，优先队列要保存<距离，编号>，我只保存了编号
