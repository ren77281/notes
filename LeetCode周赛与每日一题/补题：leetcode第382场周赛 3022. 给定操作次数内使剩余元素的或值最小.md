[3022. 给定操作次数内使剩余元素的或值最小 - 力扣（LeetCode）](https://leetcode.cn/problems/minimize-or-of-remaining-elements-using-operations/description/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202402062127801.png)
**拆位**
n个数进行或运算的结果最小，每次操作可以对相邻的两个数进行与运算，至多进行k次操作
n个数进行或运算，可以对每个数进行拆解，拆解成最小的bit后再进行或运算
比如，2，4，3进行或运算
2：0 1 0
4：1 0 0
3：0 1 1
本来是010 | 100 | 011，拆解后：(0 | 1 | 0) + (1 | 0 | 1) + (0 | 1 | 1)
从高到低对每个数bitwei进行或运算

回到题目，要使最后的运算结果最小，就要从高到低尽可能地使每个bit位为0
从高到低的过程中，若确定了某一位的运算结果能为0，之后的考虑便要带上可能为0的这一位

思路就是这样，具体实现比较难，有些考验代码能力
```cpp
class Solution {
public:
    int minOrAfterOperations(vector<int>& nums, int k) {
        int n = nums.size();
        vector<int> a(n);
        int ans = 0, mask = 0;
        for (int i = 31; i >= 0; -- i)
        {
            for (int j = 0; j < n; ++ j)
                a[j] = (nums[j] & mask) | (nums[j] & (1 << i));
            bool zero = false, flag = true;
            int cnt = 0;
            for (int j = 0; j < n; ++ j) 
            {
                int t = 0, cur = a[j];
                while (j < n && (cur &= a[j])) j ++ , t ++ ;
                if (j == n && t && zero == false) flag = false;
                else cnt += t;
                zero = true;
            }
            if (!flag || cnt > k) ans |= (1 << i);
            else mask |= (1 << i);
        }
        return ans;
    }
};
```