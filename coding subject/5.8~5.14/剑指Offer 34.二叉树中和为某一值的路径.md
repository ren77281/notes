链接：[剑指 Offer 34. 二叉树中和为某一值的路径 - 力扣（LeetCode）](https://leetcode.cn/problems/er-cha-shu-zhong-he-wei-mou-yi-zhi-de-lu-jing-lcof/?envType=study-plan-v2&id=coding-interviews)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230508181509.png)

这题应该是深搜，从根节点开始遍历，直到相加的值大于或等于目标值，搜索停止
用来记录当前路径的变量不能使用引用
```cpp
class Solution {
public:
    void check(TreeNode* node, int target, int cur_val, vector<int>& cur_path, vector<vector<int> >& ret)
    {
        if (node == nullptr)   
            return;

        cur_path.push_back(node->val);
        cur_val += node->val;
        
        if (cur_val == target && node->left == nullptr && node->right == nullptr)
            ret.push_back(cur_path);

        check(node->left, target, cur_val, cur_path, ret);
        check(node->right, target, cur_val, cur_path, ret);
        cur_path.pop_back();
    }
    vector<vector<int>> pathSum(TreeNode* root, int target) {
        vector<int> cur_path;
        vector<vector<int>> ret;
        check(root, target, 0, cur_path, ret);
        return ret;
    }
};
```
题目需要记录符合条件的路径，这里用了一个vector，当访问一个节点时将其尾插，当节点的左右节点遍历完，将其尾删。由于节点的值不只是正数，所以不能提前结束搜索，需要将所有节点遍历过去。
当然了，得到符合条件的路径可以提前返回，但因为符合条件的路径从根直到叶节点，提前返回也不会快多少，所以这里就直接遍历所有节点了。
题解用：target -= val来减少记录当前值的空间消耗，可以学一下