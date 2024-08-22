## E
判断两个字符串是否相等：统计不同字符(`str1[i] != str2[i]`)的个数cnt，cnt为0说明两字符串相同
这样的判断方式通常用于：对字符串的累计操作，比如先修改一个字符，再交换两个字符，每次操作维护好cnt就能$O(1)$地知道两字符串是否相等

交换字符时，如何维护cnt？可以是str1中的两字符交换，可以是str2中的两字符交换，也可以是分别来自str1和str2的两字符交换。分类讨论以上情况的话，代码逻辑过于复杂，甚至会漏掉一些情况，灵神给出的逻辑却是十分简洁

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230826105819.png)

分成两部分讨论：交换前与交换后。题目给出两字符具体的下标pos1与pos2，假设两个字符分别来自str1和str2
交换前：若`str1[pos1] != str2[pos1]`，交换`str1[pos1]`与`str2[pos2]`后，可能`str1[pos1] == str2[pos1]`，假设交换后pos1的两字符一定相等，那么cnt -- 
在`pos2 != pos1`的情况下，也进行以上判断，若`str1[pos2] != str2[pos2]`，cnt -- 
交换后：由于交换前的cnt -- 的前提是一个假设，若假设不成立，就要cnt ++ 
所以真正`swap(str1[pos1], str2[pos2])`后，若pos1位置上的两字符依然不相等，假设不成立，撤销之前的操作，cnt ++ 。在`pos2 != pos1`的情况下，若pos2位置上的两字符依然不相等，假设不成立，cnt ++ 

以上逻辑只考虑了交换的两字符来自不同字符串的情况，带入以上逻辑，同样能用于交换的两字符来自相同字符串的情况

回到这道题，三种操作，1. 标记字符串相同位置的字符t秒，2. 交换两字符，3. 询问除了标记字符之外，两字符串是否相等

执行第一个操作时，判断`str1[pos] != str2[pos]`，若成立，cnt -- 。每秒执行一个操作，若当前为第i秒，将pair`{ pos, i + t }`入队。执行第i秒操作时，将所有`second == i`的元素出队，撤销标记，若`str1[pos] != str2[pos]`，cnt ++ 
```cpp
#include <iostream>
#include <string>
#include <queue>
using namespace std;

typedef pair<int, int> PII;
typedef long long LL;
string str[3];
int t, Q;

void solve()
{
    cin >> str[1] >> str[2];
    int cnt = 0;
    for (int i = 0 ; i < str[1].size(); ++ i )
        if (str[1][i] != str[2][i]) cnt ++ ;
    
    queue<PII> q;
    cin >> t >> Q;
    for (int i = 1; i <= Q; ++ i )
    {
        while (q.size() && q.front().second == i)
        {
            int pos = q.front().first;
            q.pop();
            if (str[1][pos] != str[2][pos]) cnt ++ ;
        }
        
        int op; cin >> op;
        if (op == 1) 
        {
            int pos; cin >> pos; pos -- ;
            q.push({pos, i + t});
            if (str[1][pos] != str[2][pos]) cnt -- ;
        }
        else if (op == 2)
        {
            int pos1, pos2, id1, id2;
            cin >> id1 >> pos1 >> id2 >> pos2;
            pos1 -- , pos2 -- ;
            if (str[1][pos1] != str[2][pos1]) cnt -- ;
            if (pos1 != pos2 && str[1][pos2] != str[2][pos2]) cnt -- ;
            swap(str[id1][pos1], str[id2][pos2]);
            if (str[1][pos1] != str[2][pos1]) cnt ++ ;
            if (pos1 != pos2 && str[1][pos2] != str[2][pos2]) cnt ++ ;
        }
        else cout << (cnt == 0 ? "YES\n" : "NO\n");
    }
}

int main()
{
    ios::sync_with_stdio(false), cin.tie(0),cout.tie(0);
    int T; cin >> T;
    while ( T -- )
        solve();
    return 0;
}
```
debug：多组测试数据，每次测试时，数据都应该重置。以及测试数据中的下标与字符串的下标存在差1