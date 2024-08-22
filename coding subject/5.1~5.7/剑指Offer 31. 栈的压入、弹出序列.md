链接：[剑指 Offer 31. 栈的压入、弹出序列 - 力扣（LeetCode）](https://leetcode.cn/problems/zhan-de-ya-ru-dan-chu-xu-lie-lcof/?envType=study-plan-v2&id=coding-interviews)

难度中等，思路就是模拟栈的压入与弹出。
根据弹出序列，一直push压入序列中的数，直到栈顶的元素等于弹出序列的第一个数，然后判断栈顶元素是否等于弹出序列的第二个数...依此往复，若弹出序列的数不等于栈顶元素，则不断的将压入序列中的数压入，当所有压入序列的数压入后，循环结束。

如果弹出序列是正确的，那么在压入序列中的数push进栈时，栈顶元素肯定会等于弹出序列中的数，然后向后遍历弹出序列并pop栈顶元素