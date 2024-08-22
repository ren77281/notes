## 写在最前面的复盘
感谢leetcode，丰富了我为数不多的卡常经验
2是简单思维题，但卡常
4是爆搜优化，也卡常，补题时给卡麻了

对于4，赛时只想到爆搜思路，时间不够，没得想优化。个人认为这题的字符串转换过程没法一眼dp，也可能是我经验不够多，但从爆搜优化到记忆化/dp的过程是非常值得学习的
然后就是一个全新的知识点，对于前缀相同的子字符串截取，使用Trie树查找的效率很高，这时就不要使用`unordered_map<string, int>`了
还有就是Flody的剪枝，对于稀疏图来说，这个剪枝的效果非常好
最后就是卡常，leetcode好像不能用全局变量，所以这题用二维array优化了一个地方，也是第一次用array，为了不被卡常，以后能用array就尽量用

## 2977. 转换字符串的最小成本 II（Flody 爆搜优化->dp）
[2977. 转换字符串的最小成本 II - 力扣（LeetCode）](https://leetcode.cn/problems/minimum-cost-to-convert-string-ii/description/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202312261857975.png)

根据上题的经验，首先建有向图，用Flody求最短距离，但是需要将original和changed中的字符串映射为唯一编号
这里的映射不能使用`unordered_map<string, int>`，原因在于后续的求解中，C++的substr太慢

最大的问题是：要替换哪些子字符串？对于每个字符，都有替换和不替换两种情况，若替换，则要在source和target中，截取该字符往后的子字符串，若能替换，则从子字符串的后一个字符开始搜索
这里有一个优化，使用substr截取字符串吗？截取后通过`unordered_map<string, int>`索引唯一编号吗？这种做法时间开销巨大，考虑到每次截取字符串的特征：以相同字符开头。若当前截取的字符串不是图中的点，那么后续的字符串也一定不是图中的点，因为后续字符串的前缀肯定包含当前字符串。根据**前缀**的特性，可以使用Trie树保存字符串，并且保存每个字符串的唯一下标

以上思路为爆搜，时间复杂度$2^{1000}$，无法通过。考虑优化，若当前搜索到第i个字符，并且替换了长度为j的子字符串，那么下一次的搜索要从第i+j个字符开始，可以预测，这个将被多次搜索，所以想到记忆化搜索。或者，考虑到替换操作的向后依赖性，是否能先确定当前位置向后的状态呢？
当前状态`f[i]`为：将source的第i个字符以及向后的所有字符替换的最小代价，i从后往前遍历，这样每次搜索依赖的状态就是确定的
```cpp
class Solution {
public:
    long long minimumCost(string source, string target, vector<string>& original, vector<string>& changed, vector<int>& cost) {
        // vector<vector<int>> son(10010, vector<int>(30, 0));
        array<array<int, 30>, 10010> son; 
        for (int i = 0; i < 10010; ++ i) son[i].fill(0);
        vector<int> idx(10010, 0);
        int cnt = 1, n = 0;
        auto insert = [&](string& s){
            int p = 0;
            for (auto t : s)
            {
                int v = t - 'a';
                if (!son[p][v]) son[p][v] = cnt ++ ;
                p = son[p][v];
            }
            if (!idx[p]) idx[p] = ++ n;
            return idx[p];
        };
        for (auto& s : original) insert(s);
        for (auto& s : changed) insert(s);
        long long INF = 1e17;
        vector<vector<long long>> g(n + 10, vector<long long>(n + 10, INF));
        for (int i = 1; i <= n; ++ i) g[i][i] = 0;
        for (int i = 0; i < original.size(); ++ i)
        {
            int x = insert(original[i]), y = insert(changed[i]);
            g[x][y] = min(g[x][y], (long long)cost[i]);
        }
        for (int k = 1; k <= n; ++ k)
            for (int i = 1; i <= n; ++ i)
            {
                if (g[i][k] == INF) continue;
                for (int j = 1; j <= n; ++ j)
                    g[i][j] = min(g[i][j], g[i][k] + g[k][j]);
            }

        int m = source.size();
        vector<long long> d(1010, INF); d[m] = 0;
        for (int i = m - 1; i >= 0; -- i)
        {
            if (source[i] == target[i]) d[i] = d[i + 1];
            int s = 0, t = 0;
            for (int j = 0; i + j < m; ++ j)
            {
                s = son[s][source[i + j] - 'a'], t = son[t][target[i + j] - 'a'];
                if (s == 0 || t == 0) break;
                int x = idx[s], y = idx[t];
                if (x == 0 || y == 0) continue;
                d[i] = min(d[i], d[i + j + 1] + g[x][y]);
            }
        }
        if (d[0] >= INF / 2) return -1;
        else return d[0];
    }
};
```