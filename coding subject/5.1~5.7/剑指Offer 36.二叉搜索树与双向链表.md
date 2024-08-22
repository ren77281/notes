一颗二叉搜索树的中序可以得到有序数列
知道中序遍历， 但是想到的却是将当前节点和前一节点与后一节点同时建立关系：将当前节点指向前一节点，再指向后一节点，所以我会想用两个变量保存前一节点和后一节点
但是如何保存，没有具体的思路，因为肯定要使用中序遍历，所以只要保存被中序遍历到的前一个节点就行，这是没有想到的一点
然后就是不应该同时建立前一节点和后一节点的关系，因为后一节点需要再进行一次中序遍历，这就会导致前一节点的丢失，或者你可以再保存一次节点？这样做未免太臃肿了
所以只要将当前节点和前一节点建立关系即可（当前节点的prev，和前一节点的next），刚开始保存前一节点的变量为空，所以只有该变量不为空时，才可以与之建立关系
当然了，第一个节点的前一节点，最后一个节点的后一节点的关系没有建立，这需要特殊的进行维护
```cpp
class Solution {
public:
    void inorder(Node* cur, Node*& prev, Node*& first)
    {
        if (cur == nullptr) return;
        inorder(cur->left, prev, first);
        if (prev == nullptr) 
            first = cur;
        else 
            prev->right = cur;
        
        cur->left = prev;   
        prev = cur;

        inorder(cur->right, prev, first);
    }
    Node* treeToDoublyList(Node* root) {
        if (root == nullptr) return nullptr;
        Node* prev = nullptr;
        Node* first = nullptr;
        inorder(root, prev, first);
        first->left = prev;
        prev->right = first;
        return first;
    }
};
```