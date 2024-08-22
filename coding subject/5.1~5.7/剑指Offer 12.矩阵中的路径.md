链接：[剑指 Offer 12. 矩阵中的路径 - 力扣（LeetCode）](https://leetcode.cn/problems/ju-zhen-zhong-de-lu-jing-lcof/?envType=study-plan-v2&id=coding-interviews)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230507085423.png)
首先矩阵中的单词要按顺序排列，那就抽象成两个模块，一个模块找到单词的首字符，一个模块用来匹配单词
找首字符：直接暴力遍历即可
匹配单词：
- 递归，将上下左右四个位置的字符进行递归。递归时需要判断坐标是否合法，若合法且字符匹配，则继续往四个方向递归。
- 直到word被遍历完，此时修改一个引用遍历，使其他递归分支停止，进而返回
若所有字符遍历完还没有匹配上，则返回false

现在遇到一个问题：递归时，哪些判断要先执行？
```cpp
class Solution {
public:
    bool match_word(size_t i, size_t j, const vector<vector<char>>& board,\
            const string& word, size_t word_i, bool& can_return)
    {
        if (word_i == word.size())
            can_return = true;

        if (can_return)
            return true;
        // 合法性判断
        if (i >= board.size() || j >= board[0].size())
            return false;

        if (board[i][j] == word[word_i])    
        {
            cout << i << ' ' << j << endl;
            return match_word(i - 1, j, board, word, word_i + 1, can_return) ||
            match_word(i + 1, j, board, word, word_i + 1, can_return) ||
            match_word(i, j - 1, board, word, word_i + 1, can_return) ||
            match_word(i, j + 1, board, word, word_i + 1, can_return);
        }
        return false;
    }

    bool exist(vector<vector<char>>& board, string word) {
        if (board.size() == 0) return false;

        bool can_return = false;
        
        for (size_t i = 0; i < board.size(); ++i)
        {
            for (size_t j = 0; j < board[i].size(); ++j)
            {
                if (board[i][j] == word[0] && match_word(i, j, board, word, 0, can_return))
                    return true;
            }
        }
        return false;
    }
};
```
再梳理一下思路，递归函数要返回的是can_return，当word遍历完成时，说明匹配到了word，修改can_return，然后最开始的递归调用会返回can_return的true。
要进行的判断：合法性、can_return、word_i == word.size()

哦，不是这个问题，问题是：遍历过的字符不能再遍历。现在卡在这了，下午再说吧

其实可以不用can_return变量，用逻辑运算符&&或者||就行
还有访问数组的问题，没有必要再为访问数组额外开辟空间，将当前遍历的字符临时置一个数即可，四个方向遍历完，该字符就会恢复，这很巧妙
```cpp
class Solution {
public:
    bool match_word(size_t i, size_t j, vector<vector<char>>& board,\
            const string& word, size_t word_i)
    {
        if (word_i == word.size())
            return true;
        // 合法性判断
        if (i >= board.size() || j >= board[0].size())
            return false;

        if (board[i][j] != word[word_i])
            return false;

        board[i][j] = '\0';
        bool ret = match_word(i - 1, j, board, word, word_i + 1) ||
        match_word(i + 1, j, board, word, word_i + 1) ||
        match_word(i, j - 1, board, word, word_i + 1) ||
        match_word(i, j + 1, board, word, word_i + 1);
        board[i][j] = word[word_i];

        return ret;
    }

    bool exist(vector<vector<char>>& board, string word) {
        if (board.size() == 0) return false;

        for (size_t i = 0; i < board.size(); ++i)
        {
            for (size_t j = 0; j < board[i].size(); ++j)
            {
                if (match_word(i, j, board, word, 0))
                    return true;
            }
        }
        return false;
    }
};
```
为什么要叫回溯？这不就是递归，回溯不就是归的过程吗？
要注意的点：
- 使用逻辑运算符，不要忽略递归函数的返回值
- 为board临时赋值，可以省下访问数组的空间

看了官方题解，使用了访问数组，思路也是一样的，四个方向回溯完，需要恢复标记位。比较一下，明显是临时修改原数组的值来的巧妙