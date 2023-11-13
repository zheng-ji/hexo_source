---
layout: post
title: 二叉树相关的递归算法题
date: 2021-10-14 15:30:42
tags: Algorithm
categories: Algorithm
---

#### 求二叉树的最大深度

- 递归

```
int find_max_depth_V1(TreeNode* root)
{
    int depth = 0;
    if (!root)
    {
        return 0;
    }
    depth = 1 + max(find_max_depth_V1(root->left), find_max_depth_V1(root->right));
    return depth;
}
```

- 层遍历

```
int find_max_depth_V2(TreeNode* root)
{
    if (!root) { return 0; }
    int depth = 0;
    queue<TreeNode*> Q;
    Q.push(root);
    while (!Q.empty())
    {
        depth++;
        for (int i = 0; i < Q.size(); i++)
        {
            TreeNode* t = Q.front();
            Q.pop();
            if (t->left)
            {
                Q.push(t->left);
            }
            if (t->right)
            {
                Q.push(t->right);
            }
        }
    }
    return depth;
}
```

#### 求二叉树的最小深度


- 递归

```
int find_min_depth_V1(TreeNode* root)
{
    int depth = 0;
    if (!root)
    {
        return 0;
    }
    if (!root->left)
    {
        return 1 + find_min_depth_V1(root->right);
    }
    if (!root->right)
    {
        return 1 + find_min_depth_V1(root->left);
    }
    depth = 1 + min(find_max_depth_V1(root->left), find_max_depth_V1(root->right));
    return depth;
}
```

####  求二叉树中节点的个数

```
int count_node(TreeNode* root)
{
    if (!root) { return 0; }
    int left = count_node(root->left);
    int right = count_node(root->right);
    return left + right + 1;
}
```

#### 求二叉树中叶子节点的个数

```
int count_leaf_node(TreeNode* root)
{
    if (!root) return 0;
    if (!root->left && !root->right) return 1;
    return count_leaf_node(root->left) + count_leaf_node(root->right);
}
```

#### 求二叉树中第 k 层节点的个数

```
int num_of_k_node(TreeNode* root, int k)
{
    if (!root || k<1) { return 0; }
	if (k == 1) { return 1; }
    int num_left = num_of_k_node(root->left, k - 1);
    int num_right = num_of_k_node(root->right, k - 1);
    return num_left + num_right;
}
```

#### 判断二叉树是否是平衡二叉树

```
int max_depth(TreeNode* root)
{
    if (!root)
    {
        return 0;
    }
    int left = max_depth(root->left);
    int right = max_depth(root->right);
    if (left==-1 || right == -1 || fabs(left-right)>1) { return -1; }
    return max(left,right) + 1;
}

bool isBanlanced(TreeNode* root)
{
    return max_depth(root) != -1;
}
```

#### 两个二叉树是否互为镜像

```
bool isMirror(TreeNode* root1, TreeNode* root2)
{
    if (!root1 && !root2) { return true; }
    if (!root1 || !root2) { return false; }
    if (root1->val != root2->val) { return false; }
    return isMirror(root1->left, root2->right) && isMirror(root1->right, root2->left);
}
```

####  翻转二叉树 

```
TreeNode* mirrorTreeNode(TreeNode* root)
{
    if (!root)
    {
        return NULL;
    }
    TreeNode* left = mirrorTreeNode(root->left);
    TreeNode* right = mirrorTreeNode(root->right);
    root->left = right;
    root->right = left;
    return root;
}
```

#### 求两个二叉树的最低公共祖先节点

```
bool findNode(TreeNode* root, TreeNode* node)
{
    if (!root || !node) {
	 	return false;
	}
    if (root->val==node->val && root->left == node->left && root->right == node->right) {
		 return true; 
	}
    bool found = findNode(root->left, node);
    if (!found) {
		 found = findNode(root->right, node);
	}
    return found;
}

TreeNode* getLastCommonParent(TreeNode* root, TreeNode* node1, TreeNode* node2)
{
    if (findNode(root->left,node1))
    {
        if (findNode(root->right, node2)) {
            return root;
        }
        else {
            return getLastCommonParent(root->left, node1, node2);
        }
    }
    else
    {
        if (findNode(root->left,node2))
        {
            return root;
        }
        else
        {
            return getLastCommonParent(root->right, node1, node2);
        }
    }
}
```

#### 二叉树的前序遍历

```
void preOrder2(TreeNode* root, vector<int> &result)
{
    if (!root) { return; }
    result.push_back(root->val);
    preOrder2(root->left, result);
    preOrder2(root->right, result);
}

vector<int> preOrderReverse(TreeNode* root)
{
    vector<int> result;
    preOrder2(root, result);
    return result;
}
```

#### 二叉树的中序遍历

```
void midOrder2(TreeNode* root, vector<int> &result)
{
    if (!root)
    {
        return;
    }
    midOrder2(root->left, result);
    result.push_back(root->val);
    midOrder2(root->right, result);
}

vector<int> midOrderReverse(TreeNode* root)
{
    vector<int> result;
    midOrder2(root, result);
    return result;
}
```

#### 二叉树的后序遍历（递归）

```
void posOrder2(TreeNode* root, vector<int> &result)
{
    if (!root)
    {
        return;
    }
    posOrder2(root->left, result);
    posOrder2(root->right, result);
    result.push_back(root->val);
}

vector<int> midOrderReverse(TreeNode* root)
{
    vector<int> result;
    posOrder2(root, result);
    return result;
}
```

#### 输入二叉树和整数，打印出二叉树节点值等于输入整数的所有路径

```
void helper(TreeNode* root, int target, vector<int> s, int currentSum)
{
    currentSum += root->val;
    s.push_back(root->val);
    if (!root->left && !root->right)
    {
        if (currentSum == target) {
            for (auto i:s)
            {
                cout << i <<" ";
            }
            cout << endl;
        }
    }
    if (root->left)
    {
        helper(root->left, target, s, currentSum);
    }
    if (root->right)
    {
        helper(root->right, target, s, currentSum);
    }
    s.pop_back();
}

void findPath(TreeNode* root, int target)
{
    if (!root)
    {
        return;
    }
    vector<int> s;
    int currentSum = 0;
    helper(root, target, s, currentSum);
}
```

