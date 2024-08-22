链接：[剑指 Offer 67. 把字符串转换成整数 - 力扣（LeetCode）](https://leetcode.cn/problems/ba-zi-fu-chuan-zhuan-huan-cheng-zheng-shu-lcof/?envType=study-plan-v2&id=coding-interviews)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230506133302.png)
首先要跳过前面的空格，然后判断下一个字符是否是-或者+，根据这个修改sign的值
```cpp
class Solution {
public:
    int strToInt(string str) {
        int start = 0;
        int sign = 1;
        size_t length = str.size();
        size_t res = 0;
        while (start < length && str[start] == ' ') 
            ++start;    
        
        if (start < length && !isdigit(str[start]) && str[start] == '-' || str[start] == '+')
        {
            if (str[start] == '-')
                sign = -1;
            
            ++start;
        }
        
        while (start < length && isdigit(str[start]))
        {
            res = res * 10 + (str[start] - '0') * sign;
            if (res != 0)
            {
                if (sign == 1 && res > INT_MAX) return INT_MAX;
                else if (sign == -1 && res < INT_MIN) return INT_MIN;
            }
            ++start;
        }

        return res;
    }
};
```
这道题中，用res = res \* 10 + x这样的计算，可以正向地把一个字符串转换成整数，否则就要逆置或者用字符串转整数了，所以这还是值得学习的