链接：[力扣 (leetcode.cn)](https://leetcode.cn/problems/zi-fu-chuan-de-pai-lie-lcof/?envType=study-plan-v2&id=coding-interviews)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230509125231.png)

这是一道回溯的经典题型，这几天遇到了好几题回溯，这里区分一下回溯的思路：其基本思想就是“一步步的试探”，通过枚举所有可能的解，逐个尝试，并在尝试的过程中回溯到之前的状态，直到找到一个可行解或者所有解都被尝试过为止。感觉“一步步尝试”有些暴力的意思...

它包括几个步骤：
1. 选择下一个要尝试的元素
2. 递归地尝试所有可能的元素，直到找到一个可行解或者递归无法进行下去
3. 如果当前选择无法得到可行解，则回溯之前状态，尝试其他选择


```cpp
class Solution {
public:
    void back_track(string& s, size_t cur_i, vector<string>& ret, string& cur_str)
    {
        if (cur_i == s.size())
        {
            ret.push_back(cur_str);
            return;
        }

        for (size_t i = 0; i < s.size(); ++i)
        {
            if (!s[i] || i > 0 && s[i - 1] && s[i] == s[i - 1])
                continue;

            cur_str.push_back(s[i]);
            s[i] = 0;

            back_track(s, cur_i + 1, ret, cur_str);
            
            s[i] = cur_str.back();
            cur_str.pop_back();
        }
    }
    vector<string> permutation(string s) {
        vector<string> ret;
        string cur_str;
        sort(s.begin(), s.end());
        back_track(s, 0, ret, cur_str);
        return ret;
    }
};
```

这题的重点是：原串中有重复字符时，生成的全排列中，会包含同样的排列。所以如何生成不重复的全排列，这是关键。
```cpp
if (!s[i] || i > 0 && s[i - 1] && s[i] == s[i - 1])
    continue;
```
先对原串进行排序，保证所有相同的字符是挨在一起的
选择字符时，若上一个字符没有选择过，且与当前字符相同，则不需要选择当前字符。
所以一串相同的字符只会选择第一个字符