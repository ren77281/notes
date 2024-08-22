![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202312182230930.png)
wa穿了第3题，赛时其实想到了思路：中位数贪心，从中位数开始，用左右指针找到第一个回文数，与该回文数的代价就是答案。但是没有考虑到左右指针同时找到回文数的情况，wa了一发之后开始改。用一个vector保存代价，只要数组长度大于2就返回其中的较小值。但是没有注意到自己的算法是左右指针同时找，可能出现同一个指针找到两次回文数的情况，此时就不是左右指针分别找到一次回文数。后面改成：数组长度大于5就返回最小值才ac，赛后重写用第一次的思路写了一遍，很快就ac了
现在想想，自己能很快想到正解，但是算法实现的却不是正解，而且赛时还没发现，甚至以为是思路错了，还往平均数那块想了会。只能说，自己在算法实现这块考虑得不仔细吧，下次别着急，想慢点
至于说第4题，赛时完全没有想到中位数贪心，想到哪了呢？我在考虑数组中数的出现次数，出现次数更多的数，最后的代价是否会小于出现次数更小的数。甚至想到：若一个数出现次数超过一半，那么把所有数变为这个数，此时的频率是否最大？接着就对着这个贪心结论证啊证，最后不了了之。但题目的关键点是：操作次数有限，那么就要考虑如何操作能尽可能少的使用操作次数，这样就能想到中位数了

周末这几场打下来，发现自己最大的问题就是：题意的理解。一是读假题，连题目在说什么都没搞清楚，甚至是自以为搞清楚，然后自欺欺人地想算法去了，如abc的E题，小白赛的E题。二是没有抓住题意的重点，只要是稍有难度的题，都需要抓住关键点不断地分析，如这次的第4题，小白赛的F题

问题反而是出现在阅读理解上

## 2967. 使数组成为等数数组的最小代价（中位数贪心 回文数判断）
[2967. 使数组成为等数数组的最小代价 - 力扣（LeetCode）](https://leetcode.cn/problems/minimum-cost-to-make-array-equalindromic/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202312182230228.png)
根据中位数贪心，将数组排序后，从中位数开始，分别向左和向右找到第一个回文数并计算代价，数组两个代价中较小的即可
```cpp
class Solution {
public:
    bool f(string s)
    {
        int l = 0, r = s.size() - 1;
        while (l < r)
        {
            if (s[l] != s[r]) return false;
            l ++ , r -- ;
        }
        return true;
    }
    long long minimumCost(vector<int>& nums) {
        sort(nums.begin(), nums.end());
        int mid;
        
        if (nums.size() & 1) mid = nums[nums.size() / 2];
        else mid = (nums[nums.size() / 2] + nums[nums.size() / 2 - 1]) / 2;
        long long ans1 = 4e18, ans2 = 4e18;
        int l = mid, r = mid;
        while (l >= 0)
        {
            if (f(to_string(l)))
            {
                long long t = 0;
                for (int i = 0; i < nums.size(); ++ i) t += abs(nums[i] - l);
                ans1 = t;
                break;
            }
            l -- ;
        }
        while (r < 2e9)
        {
            if (f(to_string(r)))
            {
                long long t = 0;
                for (int i = 0; i < nums.size(); ++ i) t += abs(nums[i] - r);
                ans2 = t;
                break;
            }
            r ++ ;
        }
        return min(ans1, ans2);
    }
};
```
***
## 2968. 执行操作使频率分数最大（中位数贪心 前缀和 滑窗）
[2968. 执行操作使频率分数最大 - 力扣（LeetCode）](https://leetcode.cn/problems/apply-operations-to-maximize-frequency-score/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202312190842093.png)
要使将数组中的某些数变成同一个数的代价最小，依然是中位数贪心
同时这个序列必须是原序列的一段连续区间。比如原数组为1，2，3，4，将1，2，3变为2的代价一定比1，2，4变为2的代价小
题目要返回代价小于等于k的情况下，最长的连续区间，对于连续区间问题，自然想到滑动窗口
那么接下来要考虑的是窗口滑动时的代价变化，除了$O(n)$暴力求代价，还能使用前缀和进行预处理，$O(1)$地求代价
```cpp
class Solution {
public:
    int maxFrequencyScore(vector<int>& nums, long long k) {
        sort(nums.begin(), nums.end());
        int n = nums.size();
        vector<long long> a(n + 1), s(n + 1);
        for (int i = 0; i < n; ++ i) a[i + 1] = nums[i], s[i + 1] = s[i] + a[i + 1];
        auto f = [&](int l, int mid, int r) -> long long{
            long long left = (mid - l) * a[mid] - (s[mid - 1] - s[l - 1]);
            long long right = (s[r] - s[mid - 1]) - (r - mid + 1) * a[mid];
            return left + right;
        };
        int l = 1, r = 1;
        int ans = 0;
        while (r <= n)
        {
            while (f(l, (l + r) / 2, r) > k) l ++ ;
            ans = max(ans, r - l + 1);
            r ++ ;
        }
        return ans;
    }
};
```