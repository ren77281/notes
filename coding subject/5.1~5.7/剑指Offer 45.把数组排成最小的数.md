链接：[剑指 Offer 45. 把数组排成最小的数 - 力扣（LeetCode）](https://leetcode.cn/problems/ba-shu-zu-pai-cheng-zui-xiao-de-shu-lcof/)

个人认为这道题的重点在于两个数的比较，也就是自定义仿函数的问题。
比如4和41，这两个的大小如何比较？先将4和41的第一位比较，因为相等，再将4和41的第二位比较，4大于1，所以4是大于41的。当然这个比较成立的前置条件是：更小的数将占据更大的权值，将更小的数放在一个数的高位，所以41小于4。
假设数组只有两个数，就是需要比较的两个数，然后有两种排列方式414和441，一种是41占有更高的权值，一种是4占有更高的权值。显然是414更小。
同时，这题最大的收获是：将字符串拼接，利用字符串的比较，C++对字符串的比较是按字典顺序比较的。先判断两字符串的第一位字典大小，若相同再判断下一位。
实在是巧妙
```cpp
class Solution {
public:
    string minNumber(vector<int>& nums) {
        vector<string> string_nums;
        string ret;
        for (auto x : nums) string_nums.push_back(to_string(x));
        sort(string_nums.begin(), string_nums.end(), [](const string& s1, const string& s2){
            return s1 + s2 < s2 + s1;
        });
        for (auto& x : string_nums) ret += x;
        return ret;
    }
};
```