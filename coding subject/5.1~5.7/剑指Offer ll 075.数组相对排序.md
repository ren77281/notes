链接：[剑指 Offer II 075. 数组相对排序 - 力扣（LeetCode）](https://leetcode.cn/problems/0H97ZC/)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230504184827.png)

首先想到的就是哈希一下...
```cpp
class Solution {
public:
    vector<int> relativeSortArray(vector<int>& arr1, vector<int>& arr2) {
        map<int, int> counts;
        vector<int> ret;
        for (auto x : arr1)
            ++counts[x];
        for (auto x : arr2)
        {
            if (counts.count(x))
            {
                while (counts[x]--)
                {
                    ret.push_back(x);
                }
                counts.erase(x);
            }
        }

        for (auto it = counts.begin(); it != counts.end(); ++it)
        {
            for (int i = 0; i < it->second; ++i)
            {
                ret.push_back(it->first);
            }
        }
        return ret;
    }
};
```
看了题解，有一个新思路，自定义排序。但还是无法将空间复杂度降到O(1)
比如1，2，3...，n这是一个升序序列，我们要做的是将自定义升序序列映射到“正确”的升序序列中
这个映射用map建立，所以还是有空间的开销
```cpp
class Solution {
public:
    vector<int> relativeSortArray(vector<int>& arr1, vector<int>& arr2) {
        unordered_map<int, int> map;
        for (int i = 0; i < arr2.size(); ++i) 
            map[arr2[i]] = i + 1;
        sort(arr1.begin(), arr1.end(), [&](int left, int right){
            if (map[left] && map[right]) return map[left] < map[right];
            else if (map[left]) return true;
            else if (map[right]) return false;
            else return left < right;
        });
        return arr1;
    }
};
```