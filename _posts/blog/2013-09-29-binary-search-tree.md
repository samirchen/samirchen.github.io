---
layout: post
title: （算法与数据结构-7）二叉查找树
description: 二叉查找树的介绍及代码实现。
category: blog
---

查找树是一种支持多动动态集合操作的数据结构，包括 Search、Minimum、Maximum、Predecessor、Successor、Insert、Delete等。

二叉查找树(Binary Search Tree)。 它或者是一棵空树；或者是具有下列性质的二叉树：

- （1）若左子树不空，则左子树上所有结点的值均不大于它的根结点的值；
- （2）若右子树不空，则右子树上所有结点的值均不小于它的根结点的值；
- （3）左、右子树也分别为二叉排序树；

在二叉查找树上执行基本操作的时间与树的高度成正比，对一个包含 N 个结点的完全二叉树，它的高度 h 为 `O(lgN)`，这些操作的最坏时间为 `O(lgN)`，但是如果是 N 个结点的线性链，则树的高度为 `O(N)`，这些操作的最坏时间为 `O(N)`。

一棵随机构造的二叉查找树期望高度为 `O(lgN)`，所以在二叉查找树上基本动态集合操作的平均时间为 `O(lgN)`。

C语言代码：

[到github去获取源代码](https://github.com/samirchen/algorithms/blob/master/tree/binarySearchTree.c)

	#include <stdio.h>
	#include <stdlib.h>
	 
	typedef struct node Node;
	 
	struct node {
	    int value;
	    Node* parent;
	    Node* left;
	    Node* right;
	};
	 
	Node* nodeAlloc() {
	    return (Node*) malloc(sizeof(Node));
	}
	 
	// 中序遍历二叉查找树即可按从小到大序列输出所有结点。
	void inorderTreeWalk(Node* root) {
	    if (root != NULL) {
	        inorderTreeWalk(root->left);
	        printf("%d ", root->value);
	        inorderTreeWalk(root->right);
	    }
	}
	 
	// 递归方式查找。时间复杂度 O(h)，空间复杂度 O(1)。
	Node* recursionSearch(Node* root, int key) {
	    if (root == NULL || key == root->value) {
	        return root;
	    }
	    else if (key < root->value) {
	        return recursionSearch(root->left, key);
	    }
	    else {
	        return recursionSearch(root->right, key);
	    }
	}
	// 非递归方式查找。时间复杂度 O(h)，空间复杂度 O(1)。
	Node* nonrecursionSearch(Node* root, int key) {
	    while (root != NULL && key != root->value) {
	        if (key < root->value) {
	            root = root->left;
	        }
	        else {
	            root = root->right;
	        }
	    }
	 
	    return root;
	}
	 
	// 查找以 x 结点为根的子树中的最小值。时间复杂度 O(h)，空间复杂度 O(1)。
	Node* treeMinimum(Node* x) {
	    while (x->left != NULL) {
	        x = x->left;
	    }
	 
	    return x;
	}
	// 查找以 x 结点为根的子树中的最大值。时间复杂度 O(h)，空间复杂度 O(1)。
	Node* treeMaximum(Node* x) {
	    while (x->right != NULL) {
	        x = x->right;
	    }
	 
	    return x;
	}
	 
	// 查找在有序序列中，x 结点的下一个结点。时间复杂度 O(h)，空间复杂度 O(1)。
	Node* treeSuccessor(Node* x) {
	    if (x->right != NULL) { // 当 x 有右子树，x 的后继即其右子树中的最小结点。
	        return treeMinimum(x->right);
	    }
	    else { // 当 x 没有右子树，y 是 x 的后继需满足：1）y 是 x 的祖先；2）y 的左儿子是 x 高度最近的祖先或者就是 x。
	        // 下面 x、y 都作为游标，y 始终是 x 的 parent。
	        Node* y = x->parent;
	        while (y != NULL && x != y->left) {
	            x = y;
	            y = y->parent;
	        }
	 
	        return y;
	    }
	}
	// 查找在有序序列中，x 结点的上一个结点。时间复杂度 O(h)，空间复杂度 O(1)。
	Node* treePredecessor(Node* x) {
	    if (x->left != NULL) { // 当 x 有左子树，x 的前驱即其左子树中的最大结点。
	        return treeMaximum(x->left);
	    }
	    else {// 当 x 没有左子树，y 是 x 的前驱需满足：1）y 是 x 的祖先；2）y 的右儿子是 x 高度最近的祖先或者就是 x。       
	        // 下面 x、y 都作为游标，y 始终是 x 的 parent。
	        Node* y = x->parent;
	        while (y != NULL && x != y->right) {
	            x = y;
	            y = y->parent;
	        }
	 
	        return y;
	    }
	}
	 
	// 插入一个结点。时间复杂度 O(h)，空间复杂度 O(1)。
	// 思路：新插入的结点一定可以作为一个叶子结点，插入过程就是找到一个合适的位置。从根结点开始并沿树下降。指针 x 跟踪了这条路径，而 y 则始终指向 x 的父结点。使这两个指针沿树下降，根据 z->value 和 x->value 的比较结果来决定向左或向右转，直到 x 的值为 NULL 为止，这时这个 NULL 所占的位置即新结点 z 应该插入的地方。
	void treeInsert(Node* root, Node* z) {
	    Node* y = NULL;
	    Node* x = root;
	    while (x != NULL) {
	        y = x;
	        if (z->value < x->value) {
	            x = x->left;
	        }
	        else {
	            x = x->right;
	        }
	    }
	 
	    z->parent = y;
	 
	    if (y == NULL) {
	        root = z;
	    }
	    else {
	        if (z->value < y->value) {
	            y->left = z;
	        }
	        else {
	            y->right = z;
	        }
	    }
	}
	// 删除一个结点。时间复杂度 O(h)，空间复杂度 O(1)。
	// 思路：1）如果 z 没有子女，则修改其父结点，使 NULL 为其子女；2）如果 z 只有一个子女，则可以通过在其子结点与父结点间建立一条链接来删除 z；3）如果 z 有两个子女，则先删除 z 的后继 y（y 没有左子女，如果 y 有左子女，则 z 的后继应该是 y 的左子女而不是 y），再用 y 的内容来替代 z 的内容。
	Node* treeDelete(Node* root, Node* z) {
	    Node* x = NULL;
	    Node* y = NULL;
	    if (z->left == NULL || z->right == NULL) {
	        y = z;
	    }
	    else {
	        y = treeSuccessor(z);
	    }
	 
	    if (y->left != NULL) {
	        x = y->left;
	    }
	    else {
	        x = y->right;
	    }
	 
	    if (x != NULL) {
	        x->parent = y->parent;
	    }
	 
	    if (y->parent == NULL) {
	        root = x;
	    }
	    else {
	        if (y == y->parent->left) {
	            y->parent->left = x;
	        }
	        else {
	            y->parent->right = y;
	        }
	    }
	 
	    if (y != z) {
	        z->value = y->value;
	    }
	 
	    return y;
	}
	 
	int main() {
	    // Create tree.
	    Node* n4 = nodeAlloc();
	    n4->value = 2;
	    n4->left = NULL;
	    n4->right = NULL;
	 
	    Node* n5 = nodeAlloc();
	    n5->value = 4;
	    n5->left = NULL;
	    n5->right = NULL;
	 
	    Node* n2 = nodeAlloc();
	    n2->value = 3;
	    n2->left = n4;
	    n2->right = n5;
	 
	    Node* n6 = nodeAlloc();
	    n6->value = 8;
	    n6->left = NULL;
	    n6->right = NULL;
	 
	    Node* n3 = nodeAlloc();
	    n3->value = 7;
	    n3->left = NULL;
	    n3->right = n6;
	 
	    Node* root = nodeAlloc();
	    root->value = 5;
	    root->left = n2;
	    root->right = n3;
	 
	    root->parent = NULL;
	    n2->parent = root;
	    n3->parent = root;
	    n4->parent = n2;
	    n5->parent = n2;
	    n6->parent = n3;
	 
	 
	    // Walk tree.
	    inorderTreeWalk(root);
	    printf("\n");
	 
	    // Recursion search.
	    Node* resultNode = recursionSearch(root, 8);
	    printf("Recursion search result: %d\n", resultNode->value);
	    // Nonrecursion search.
	    resultNode = nonrecursionSearch(root, 8);
	    printf("Nonrecursion search result: %d\n", resultNode->value);
	 
	    // Min value.
	    resultNode = treeMinimum(root);
	    printf("Minimum: %d\n", resultNode->value);
	    // Max value.
	    resultNode = treeMaximum(root);
	    printf("Maximum: %d\n", resultNode->value);
	     
	    // Tree successor.
	    resultNode = treeSuccessor(n3); 
	    printf("Successor of %d is %d\n", n3->value, resultNode->value);
	    // Tree predecessor.
	    resultNode = treePredecessor(n3);
	    printf("Predecessor of %d is %d\n", n3->value, resultNode->value);
	 
	    // Insert node.
	    Node* n7 = nodeAlloc();
	    n7->value = 6;
	    treeInsert(root, n7);
	    inorderTreeWalk(root);
	    printf("\n");
	 
	    // Delete node.
	    treeDelete(root, n7);
	    inorderTreeWalk(root);
	    printf("\n");
	    free(n7);
	 
	 
	    free(root);
	    free(n2);
	    free(n3);
	    free(n4);
	    free(n5);
	    free(n6);   
	 
	    return 0;
	}


[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{page.title}})