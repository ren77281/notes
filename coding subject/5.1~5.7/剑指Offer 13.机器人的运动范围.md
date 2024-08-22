链接：[剑指 Offer 13. 机器人的运动范围 - 力扣（LeetCode）](https://leetcode.cn/problems/ji-qi-ren-de-yun-dong-fan-wei-lcof/?envType=study-plan-v2&id=coding-interviews)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230507184251.png)

这看起来是BFS，首先起点的位数之和为0，虽然机器人每次都可以向上下左右移动，但只需要让它向右或者向下移动，这也可以遍历完所有格子。
同时需要记录哪些格子已经访问过，最后用一个引用参数，记录所有格子即可
还需要写一个函数，以判断当前坐标是否合法，以下解法用cur_k判断显然是错的，这是直接将两坐标相加了
```cpp
class Solution {
public:
    void move(size_t i, size_t j, int k, int cur_k, vector<vector<bool> >& visited, int& count)
    {
        if (i >= visited.size() || j >= visited[0].size())
            return;

        if (!visited[i][j])
        {
            if (cur_k > k)
                return;
            visited[i][j] = true;
            ++count;
        }

        move(i + 1, j, k, cur_k + 1, visited, count);
        move(i, j + 1, k, cur_k + 1, visited, count);
    }
    int movingCount(int m, int n, int k) {
        if (m == 0) 
            return 0;
        vector<vector<bool> > visited(m, vector<bool>(n, false));
        int count = 0;
        move(0, 0, k, 0, visited, count);
        for (int i = 0; i < visited.size(); ++i)
        {
            for (int j = 0; j < visited[0].size(); ++j)
                cout << visited[i][j] << ' ';
            cout << endl;
        }
        return count;
    }
};
```
这样超时
```cpp
class Solution {
public:
    int get_sum(size_t x)
    {
        int ret = 0;
        while (x)
        {
            ret += x % 10;
            x /= 10;
        }
        return ret;
    }
    
    void move(size_t i, size_t j, int k, vector<vector<bool> >& visited, int& count)
    {
        if (i >= visited.size() || j >= visited[0].size())
            return;

        if (!visited[i][j])
        {
            if (get_sum(i) + get_sum(j) > k)
                return;
                cout << i << ' ' << j << endl;
            visited[i][j] = true;
            ++count;
        }

        move(i + 1, j, k, visited, count);
        move(i, j + 1, k, visited, count);
    }
    
    int movingCount(int m, int n, int k) {
        if (m == 0) 
            return 0;
        vector<vector<bool> > visited(m, vector<bool>(n, false));
        int count = 0;
        move(0, 0, k, visited, count);
        return count;
    }
};
```
这样不超，原因是：访问过的点不用再递归，直接返回即可。总之是递归写的不够多
```cpp
    void move(size_t i, size_t j, int k, vector<vector<bool> >& visited, int& count)
    {
        if (i >= visited.size() || j >= visited[0].size() || get_sum(i) + get_sum(j) > k || visited[i][j])
            return;

        visited[i][j] = true;
        ++count;

        move(i + 1, j, k, visited, count);
        move(i, j + 1, k, visited, count);
    }
```