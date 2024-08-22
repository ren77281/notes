```toc
```


### 317
在有容积限制的情况下，装入背包的物品能获得的最大价值
没有容积限制的情况下，能装入所有的物品，装入物品需要付出代价，问最小的代价
***

### 312
[C - Invisible Hand (atcoder.jp)](https://atcoder.jp/contests/abc312/tasks/abc312_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230905150917.png)
卖商品的人会以大于等于数组中的价格卖出商品，买商品的人会以小于等于数组中的价格买入商品，满足以x元卖商品的人大于等于以x元买商品的人，求x的最小值
该题具有二段性，确定一个mid后，a数组小于等于mid的人将以mid价格卖出商品，b数组大于等于mid的人将以mid价格买入商品。若mid满足卖出的人大于买入的人时，大于mid的值依然满足该性质，小于mid的值可能不满足该性质，所以二分找到分界点即可
***
### 311
[C - Find it! (atcoder.jp)](https://atcoder.jp/contests/abc311/tasks/abc311_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230905150900.png)
该题定义了一个有向循环，在保证给定的图中存在有向循环时，找出该循环序列，序列的顺序任意
题目给定的图具有性质：每个点的出度都为1，即一张拓扑图，由于题目保证点的数量与边的数量相等，所以图中一定存在一个环。从任意点开始遍历一定能重新遍历到该点，此时以该点所以环的起点，重新遍历并输出该环即可
由于点的范围确定，并且每个点都只有一条出边，所以这题直接用一维数组保存图即可，遍历x的下一个点，`x = a[x]`即可
***
[D - Grid Ice Floor (atcoder.jp)](https://atcoder.jp/contests/abc311/tasks/abc311_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230905150931.png)
在矩阵中，每次选择上下左右中的一个方向前进，直到遇到障碍物停止，此时再选择一个方向前进。问能前进的最大距离？
搜索题，难点在于如何限制状态使得搜索不重不漏。先将起点入队，停止时将停下的点入队，每个点有四个方向可以前进，所有点的四个方向都是一个状态。确保队列中每个点的四个方向都被遍历即可
最开始入队的起点，状态也需要维护，ans++
***
### 309
[D - Add One Edge (atcoder.jp)](https://atcoder.jp/contests/abc309/tasks/abc309_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230905153226.png)
给定两个连通块点数分别为n1和n2，添加一条边，使得1号点到n1+n2的最近距离最远
由于边的权值为1，所以bfs分别求出距离1和n1+n2的最远点以及距离，将两个最远点连接即可
***
### 308
[C - Standings (atcoder.jp)](https://atcoder.jp/contests/abc308/tasks/abc308_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230905153840.png)
计算浮点数，根据结果进行升序排序，输出下标
由于浮点数的精度丢失，这题用long double可解，double则会wa
或者自定义排序规则，转换公式使之进行乘法运算
***
[D - Snuke Maze (atcoder.jp)](https://atcoder.jp/contests/abc308/tasks/abc308_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230905154117.png)
从左上角移动到右下角，移动产生的序列序列需要按照一定的顺序：snuke，问能否按照这样的顺序移动到右下角？
用%运算可以不断地遍历这个顺序，只要保证每次dfs的顺序正确即可
***
### 307
[D - Mismatched Parentheses (atcoder.jp)](https://atcoder.jp/contests/abc307/tasks/abc307_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230905154419.png)
删除()括号以及括号中的所有字符，括号中不能包含其他括号，输出最后的字符串
用stack模拟一遍即可
***
### 306
[D - Poisonous Full-Course (atcoder.jp)](https://atcoder.jp/contests/abc306/tasks/abc306_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230905155254.png)
每道菜选择吃与不吃，吃完后有两种状态，健康与不健康，不健康状态下不能吃有毒的菜，只能吃解毒的菜。每道菜有一个分数，问能获得的最大分数？
dp题，吃完第i道菜后两种状态，健康与不健康
如何更新`dp[i]`？若当前菜解毒，那么无法吃下这道菜更新`dp[i][1]`，`dp[i][1] = dp[i - 1][1]`，接着考虑吃掉这道菜更新`dp[i][0]`，跳过，从健康，不健康三个状态更新
若当前菜有毒，那么无法吃下这道菜更新`dp[i][0]`，`dp[i][0] = dp[i - 1][0]`，接着考虑吃掉这道菜更新`dp[i][1]`，跳过，从健康状态更新
***
### 305
[D - Sleep Log (atcoder.jp)](https://atcoder.jp/contests/abc305/tasks/abc305_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230910163907.png)
数组由连续的0和1组成，给定l和r返回区间和（连续1和0间隔出现）
由于l和r的范围很大，不能建立相同大小的前缀和数组，只能采用离散化的方式建立前缀和数组
查询之前先将l和r转换为离散化之后的下标，由于查询是大于等于的，根据查询的结果判断l和r落在连续0还是连续1的区间中，将结果+-掉这部分的值
***
### 303
[C - Dash (atcoder.jp)](https://atcoder.jp/contests/abc303/tasks/abc303_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230910165321.png)
模拟题，按照给定的序列从源点开始走，每走一步消耗一点体力，走到某个格子体力会恢复为k，判断走完给定的序列后，体力是否为正数？
用set存储能恢复体力的格子，每走一步判断是否能恢复体力，若是能则删除给格子
***
[D - Shift vs. CapsLock (atcoder.jp)](https://atcoder.jp/contests/abc303/tasks/abc303_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230910165621.png)
dp题，状态转移的情况需要考虑完全
考虑敲出第i个字符（a）后，模式为小写：
从小写直接按a，从小写切换成大写按shift+a，从大写切换成小写再按a，从大写按shift+a再切换成小写
考虑敲出第i个字符（A）后，模式为大写：
从大写直接按a，从大写切换成小写按shift+a，从小写切换成大写再按a，从小写按shift+a再切换成大写
也就是要考虑第i个字符为a还是A，完成后的模式是大写还是小写，考虑时要从小写还是大写模式转移过来。最后一种考虑最容易漏掉情况
***
### 302
[C - Almost Equal (atcoder.jp)](https://atcoder.jp/contests/abc302/tasks/abc302_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911081850.png)
重新排列每个字符串，使得相邻字符串之间只有一个字符不同
n很小，暴力搜索即可。先预处理出每个字符串相似的字符串，用二维数组存储。之后建立一个虚拟源点进行dfs
***
[D - Impartial Gift (atcoder.jp)](https://atcoder.jp/contests/abc302/tasks/abc302_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911084821.png)
upper_bound接口要求数组必须是升序，所以之前不能降序排列数组
给定两个数组，返回差值小于等于d的两个数，要求两数之和最大
二分，遍历a数组找b <= a + d，找完后需要特判b与a的差值是否合法，合法再维护ans
之前的解法：双指针贪心，降序排序两数组，一开始两个指针指向两数组中最大的元素，满足差值小于等于d的条件直接输出，否则将指向较大数的指针指向更小的一个数
***
### 300
[C - Cross (atcoder.jp)](https://atcoder.jp/contests/abc300/tasks/abc300_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911094722.png)
判断矩阵中大小为1~min(h, w)中的x数量。由于矩阵很小所以爆搜即可
***
### 299
[D - Find by Query (atcoder.jp)](https://atcoder.jp/contests/abc299/tasks/abc299_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911095045.png)
交互式题目，一个由0和1组成的序列，第一个数为0，最后一个数为1，查找ai使得ai不等于ai+1
根据题意，序列中一定存在ai=0，ai+1=1，我们的目的是搜索出ai=0的ai下标。使用二分，若mid为1，r=mid-1，若mid为0，l=mid
***
### 298
[C - Cards Query Problem (atcoder.jp)](https://atcoder.jp/contests/abc298/tasks/abc298_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911103447.png)
将卡片放入盒子中，按升序打印某个盒子中的卡片，按降序打印含有某个卡片的盒子编号
数据结构题，用升序set维护第一种打印，用降序set维护第二种打印
***
### 293
[C - Make Takahashi Happy (atcoder.jp)](https://atcoder.jp/contests/abc293/tasks/abc293_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911105016.png)
dfs即可
***
### 292
[C - Four Variables (atcoder.jp)](https://atcoder.jp/contests/abc292/tasks/abc292_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911110142.png)
分解因数
将AB和CD看成l和r，枚举l+r=n，分解l与r的因数，将两者相乘
***
[D - Unicyclic Components (atcoder.jp)](https://atcoder.jp/contests/abc292/tasks/abc292_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911134519.png)

遍历无向图，判断连通部分的点数与边数是否相等
dfs保存点数与边数即可，这题值得多刷几次，我的dfs很经常出错
***
### 291
[D - Flip Cards (atcoder.jp)](https://atcoder.jp/contests/abc291/tasks/abc291_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911140951.png)
翻转卡片，使得相邻卡片上的数字不相等，问有几种方法满足该条件
看数据范围知道这是dp题，若第i张卡片不翻转，那么满足条件的翻转方式有几种
以及第i张卡片翻转，那么满足条件的翻转方式有几种
考虑与上一张卡片的两个数字是否相同，若不相同，那么方法数等于该状态的方法数
主要是状态转移有些绕
***
### 289
[D - Step Up Robot (atcoder.jp)](https://atcoder.jp/contests/abc289/tasks/abc289_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911152249.png)
数组中某些位置不能递达，每次能走指定的步长，问是否能递达x位置
广搜即可
也可以开数组记录某个位置是否能递达，先将不能递达的位置删去，再线性遍历1~x，每次遍历时根据能走的指定步长判断该位置是否能递达，用|= 即可
***
### 288
[C - Don’t be cycle (atcoder.jp)](https://atcoder.jp/contests/abc288/tasks/abc288_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911183319.png)
输出无向图中环的数量
两种方法：并查集与dfs
由于图是无向图，所以不需要使用spfa，且spfa只能判断是否存在环不能判断环的数量
检查无向图的环的数量与生成树有关，一颗生成树中n个点，只能有n-1条边，若图中有cnt个连通块，那么需要有cnt个生成树，那么图中只能有n-cnt条边
***
### 287
[C - Path Graph? (atcoder.jp)](https://atcoder.jp/contests/abc287/tasks/abc287_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911190039.png)
判断该图是否是一张链表。
根据题意，首先图必须是一张连通图，n = m + 1，其次不能存在环，即度为1的点只能有两个，并且其他点的度必须为2
考虑图的问题时，度也是一个考虑角度
***
[D - Match or Not (atcoder.jp)](https://atcoder.jp/contests/abc287/tasks/abc287_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911195454.png)
s串的长度大于t串，考虑将s串的前缀和后缀进行拼接，使之长度等于t串的长度。输出每种拼接方式，两串是否相等
考虑直接暴力求解将进行多次的重复判断，由于前缀和后缀都是连续的，所以可以预处理出两串相等的最长前缀与后缀，之后的线性枚举直接根据长度判断即可
***
### 286
[C - Rotate and Palindrome (atcoder.jp)](https://atcoder.jp/contests/abc286/tasks/abc286_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230911202517.png)
可以将字符串进行旋转，或者直接修改字符，使字符串成为回文串，每次操作都要支付相应的费用，问最小的费用
数据范围较小，直接暴力即可
***
### 285
[D - Change Usernames (atcoder.jp)](https://atcoder.jp/contests/abc285/tasks/abc285_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230912092013.png)
替换n个字符串，满足替换后的字符串两两不相等，问是否有一个替换顺序满足该条件
分析题意后，发现题目的本质是询问无向图是否存在环，用并查集判断即可
其实是有向图判环，由于题目保证s和t两集合中的字符串两两不相等，所以可以用无向图判环
***
### 284
[D - Happy New Year 2023 (atcoder.jp)](https://atcoder.jp/contests/abc284/tasks/abc284_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230919084542.png)
题目保证给定的n一定能表示成两个质数相乘的形式
只需要枚举到n的三次根号技能找出答案，可以直接筛出1e6之间的质数，也可以从i开始直接枚举因数
***
### 283
[D - Scope (atcoder.jp)](https://atcoder.jp/contests/abc283/tasks/abc283_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230919084011.png)
水题，用栈与set结构维护一下就可以
***
### 279
[D - Freefall (atcoder.jp)](https://atcoder.jp/contests/abc279/tasks/abc279_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230919090336.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230919090238.png)

三分：不需要使用mid，反而使用midl与midr，若f(midl) < f(midr)，那么一定能确定$[midr, r]$之间函数必定是单增的，此时r = midr - 1，否则l = midl + 1
***
### 278
[D - All Assign Point Add (atcoder.jp)](https://atcoder.jp/contests/abc278/tasks/abc278_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230919092703.png)
三种不同的操作，将所有数据置为同一个值O(n)，增加某个位置的值，打印某个位置的值。关键在O(n)的时间如何优化，由于有q个操作，若每个操作都是O(n)，那么一定t
考虑数组中的每个数=打印前的最后一次赋值+之后被添加的值。由于每次的赋值操作都是统一的，所以可以用一个变量记录赋值操作的值。每个数据的增加操作都是不同的，这需要分别记录，当赋值操作发生时，将维护每个数据的增加操作的结构的值置为0
***
### 277
[C - Ladder Takahashi (atcoder.jp)](https://atcoder.jp/contests/abc277/tasks/abc277_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230919190440.png)
1所在的连通块中，编号最大的点为多少？用dfs遍历即可，维护最大编号
由于编号最大为1e9，所以用unordered_map与set存储图与标记数组
***
### 276
[D - Divide by 2 or 3 (atcoder.jp)](https://atcoder.jp/contests/abc276/tasks/abc276_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230919195247.png)
与除法有关的数论题，一般都与分解质因数有关系。将所有的尽可能地除2或者3，使这些数最后相等，那么所有数中除了$2^i$与$3^i$之外的质因数都要相等
可以先算出所有数的最大公因数，将所有数除去最大公因数，再将所有数尽可能地除以2或者3，所有次数累加就是答案
若除完后，剩下的数不是1，说明无法使这些数相等，打印-1
***
### 275
[C - Counting Squares (atcoder.jp)](https://atcoder.jp/contests/abc275/tasks/abc275_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230921192610.png)
枚举矩阵中正方形的数量，正方形的边不一定与坐标轴垂直
由于数据量较小，直接暴力枚举。枚举正方形的左下角顶点，然后枚举dx和dy，通过dx和dy确定下一个点的位置。边长与坐标轴平行的正方形，dx和dy中必然有一个为0，只需要枚举非负数的dx与dy，就能完整的枚举出矩阵中的所有正方形。将一个正方形的四个点用set保存起来，最后再用一个set进行去重
***
[D - Yet Another Recursive Function (atcoder.jp)](https://atcoder.jp/contests/abc275/tasks/abc275_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230921194905.png)
n很大，直接递归会爆栈，使用记忆化搜索

***
### 274
[C - Ameba (atcoder.jp)](https://atcoder.jp/contests/abc274/tasks/abc274_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828151009.png)
数组中第i个数，生成2i与2i+1。问2n+1中的每个数与1的距离
建个图就行了
***
### 271
[A - 484558 (atcoder.jp)](https://atcoder.jp/contests/abc271/tasks/abc271_a)

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230827100836.png)
%02x为前导零输出2位十六进制
***
[C - Manga (atcoder.jp)](https://atcoder.jp/contests/abc271/tasks/abc271_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828101952.png)

两本漫画换一本，能连续读到第几本？
***
### 270
[A - 1-2-4 Test (atcoder.jp)](https://atcoder.jp/contests/abc270/tasks/abc270_a)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828101118.png)
抓重点，没有得到A和B都没得到的分数，就是个|运算
***
[B - Hammer (atcoder.jp)](https://atcoder.jp/contests/abc270/tasks/abc270_b)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828101500.png)
要递达终点，但可能被墙堵住，用锤子能凿开墙
分类讨论即可，还是讨论的很细的
***
[C - Simple path (atcoder.jp)](https://atcoder.jp/contests/abc270/tasks/abc270_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828101638.png)

找出树中两点间的唯一路径，dfs即可
**一般的树要加双向边**
回溯写得还是不够熟练
***
### 269
[B - Rectangle Detection (atcoder.jp)](https://atcoder.jp/contests/abc269/tasks/abc269_b)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828100257.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828100506.png)
维护最大最小值
***
[C - Submask (atcoder.jp)](https://atcoder.jp/contests/abc269/tasks/abc269_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828100651.png)
对于所有的1，都有出现与没出现两种情况，可以写搜索找出所有这样的数
还能用位运算技巧：n & (n - 1)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828101007.png)
***
[D - Do use hexagon grid (atcoder.jp)](https://atcoder.jp/contests/abc269/tasks/abc269_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230921203907.png)
将某些点涂成黑色，问黑色的连通块数量
可以用并查集做，将每个黑色点转换成唯一的数字，用map维护映射关系
也可以用bfs做，遍历所有黑色点，未遍历过的黑色点进行bfs，ans++，每次bfs将未遍历过的黑色点加入队列
***
### 268
[B - Prefix? (atcoder.jp)](https://atcoder.jp/contests/abc268/tasks/abc268_b)
前缀包括本身
***
[C - Chinese Restaurant (atcoder.jp)](https://atcoder.jp/contests/abc268/tasks/abc268_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828095656.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828095714.png)

一题不错的模拟，圆盘的转动格数从1~n，正常想法是枚举每种格数有多少人开心
但代码实现的逻辑很复杂，反着思考，让每个人开心需要转动的格数，每个人都有三种不同的个数，用map存储格数与人数的关系输出最多人的格数即可
***
### 267
[B - Split? (atcoder.jp)](https://atcoder.jp/contests/abc267/tasks/abc267_b)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828093812.png)

以列为单位，只要球全倒下的列两边存在球没有倒下的列视为split
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828093952.png)
可以用条件特判，但最简单是位运算。以列为数位，球全倒下的列为0，其他为1，找到第一个1，此时的数为b，`b & b + 1`即可
0111011，b = 1011，b+1 = 0111
判断结果是否为1即可
之前没见过，`b & b + 1`这样的位运算

**&的优先级低于`==`，+-\\\*高于&**
***
[C - Index × A(Continuous ver.) (atcoder.jp)](https://atcoder.jp/contests/abc267/tasks/abc267_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828095104.png)
计算长度为m的“子段和”，暴力求出所有的子段和时，会发现计算的重复
m为窗口，每次向右滑动，都会减去子段和再加上$m * a_{i+m}$
读懂题意后，思考暴力解法，再优化掉暴力中的可优化部分，“滑动窗口”
***
### 266
[B - Modulo Number (atcoder.jp)](https://atcoder.jp/contests/abc266/tasks/abc266_b)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828092735.png)
确定x使得n-x为k的倍数，x必须是正数
这就是取模的另一种表述，注意C++的负数取模后为负数，所以n%k后为负数时，再加上k
[C - Convex Quadrilateral (atcoder.jp)](https://atcoder.jp/contests/abc266/tasks/abc266_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230828093135.png)
确定四边形的内角是否都小于180°，向量叉乘，叉乘结果大于0内角小于180°，小于等于0内角大于等于180°
$(x1, x2) * (y1, y2) = x1 * y2 - x2 * y1$
根据叉乘运算，内角为以逆时针方向，前一个向量到后一个向量的夹角
***
### 265
[D - Iroha and Haiku (New ABC Edition) (atcoder.jp)](https://atcoder.jp/contests/abc265/tasks/abc265_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230922174345.png)
若找出三个连续的区间，使得区间和为指定的数。因为区间是连续的，所以可以考虑前缀和，确定第一个点后，根据题目给定的区间和来判断剩余的三个点是否存在，用set的count进行判断
***
### 264
[C - Matrix Reducing (atcoder.jp)](https://atcoder.jp/contests/abc264/tasks/abc264_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230922192441.png)
删除矩阵a的行或者列，使之等于b
有一个技巧，将大矩阵转换成小矩阵。用二进制进行枚举，枚举出行数和列数都相同的a矩阵的行号与列号，用数组row和col存储，这样row和col数据与下标之间的映射关系就是两个矩阵的行号与列号之间的映射关系
***
[D - "redocta".swap(i,i+1) (atcoder.jp)](https://atcoder.jp/contests/abc264/tasks/abc264_d)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230922195215.png)

每次交换两相邻字符，使之顺序正确
统计逆序对数量即可
***
### 260
[C - Changing Jewels (atcoder.jp)](https://atcoder.jp/contests/abc260/tasks/abc260_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230922200742.png)
误以为是dp，但是每次的蓝宝石与红宝石的交换只有一种选择，所以这题直接递推就好了
***
### 259
[C - XX to XXX (atcoder.jp)](https://atcoder.jp/contests/abc259/tasks/abc259_c)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/20230922202510.png)
统计每个字符的出现次数，根据规则进行判断即可