---
layout: post
title: （算法与数据结构-10）红黑树
description: 红黑树的介绍及代码实现。
category: blog
---

查找树是一种支持多动动态集合操作的数据结构，包括 `Search`、`Minimum`、`Maximum`、`Predecessor`、`Successor`、`Insert`、`Delete`等。

`红黑树（red-black tree）`是一种“平衡的”查找树，同时也是一种特殊的二叉查找树。红黑树中的每个结点包含五个域：`color`，`key`，`left`，`right`，`parent`。如果某结点的子结点（`left` 或 `right`）或父结点（`parent`）为空，则把这些为空的结点视为 NIL 结点，把 NIL 结点看做是指向二叉查找树的外结点（叶子）的指针，而把带关键字的结点视为树的内结点。

红黑树是具有下列性质的二叉查找树：

- （1）每个节点或是红的，或是黑的；
- （2）根结点是黑的；
- （3）每个叶结点（NIL）是黑的；
- （4）如果一个结点是红的，则它的两个儿子都是黑的；
- （5）对每个结点，从该结点到其子孙结点的所有路径包含相同数目的黑结点。

红黑树通过对任何一条从根到叶子的路径上各个结点着色方式的限制来确保没有一条路径会比其他路径长出两倍。一棵有 N 个内结点的红黑树的高度至多为 `2lg(N+1)`。红黑树能保证即使在最坏的情况下，基本动态集合的操作的时间为 `O(lgN)` 。

红黑树的基本动态集合操作 Search、Minimum、Maximum、Predecessor、Successor 和二叉查找树基本一致，但是 Insert、Delete 操作有一些不同，因为在插入或删除红黑树中的结点后，可能破坏了红黑树的上述 5 点性质中的某一条或几条，需要在插入或删除后进行修正以维持其红黑树性质不变从而保证树的高度在 O(lgN) 级别。

C 语言代码：

[到github去获取源代码](https://github.com/samirchen/algorithms/blob/master/tree/redBlackTree.c)

	#include <stdio.h>
	#include <stdlib.h>
	#include <time.h>
	#include <string.h>
	 
	typedef enum Color {
	    RED = 0,
	    BLACK = 1
	}Color;
	 
	typedef struct node Node;
	struct node {
	    Node *left, *right, *parent;
	    int key;
	    int data;
	    Color color;
	};
	 
	Node* RBNodeAlloc(int key, int data) {
	    Node *node = (Node*)malloc(sizeof(Node));
	    if (!node) {
	        printf("malloc error!\n");
	        exit(-1);
	    }
	    node->key = key, node->data = data;
	 
	    return node;
	}
	 
	void inorderTreeWalk(Node* root) {
	    if (root != NULL) {
	        inorderTreeWalk(root->left);
	        printf("[%d %d] ", root->key, root->color);
	        inorderTreeWalk(root->right);
	    }
	}
	 
	Node* RBTreeSearch(Node* root, int value) {
	    while (root != NULL && value != root->key) {
	        if (value < root->key) {
	            root = root->left;
	        }
	        else {
	            root = root->right;
	        }
	    }
	 
	    return root;
	}
	 
	Node* treeMinimum(Node* x) {
	    while (x->left != NULL) {
	        x = x->left;
	    }
	 
	    return x;
	}
	 
	Node* treeMaximum(Node* x) {
	    while (x->right != NULL) {
	        x = x->right;
	    }
	 
	    return x;
	}
	 
	Node* treeSuccessor(Node* x) {
	    if (x->right != NULL) {
	        return treeMinimum(x->right);
	    }
	    else {
	        Node* y = x->parent;
	        while (y != NULL && x == y->right) {
	            x = y;
	            y = y->parent;
	        }
	 
	        return y;
	    }
	}
	 
	Node* treePredecessor(Node* x) {
	    if (x->left != NULL) {
	        return treeMaximum(x->left);
	    }
	    else {
	        Node* y = x->parent;
	        while (y != NULL && x != y->right) {
	            x = y;
	            y = y->parent;
	        }
	 
	        return y;
	    }
	}
	 
	 
	/*
	 * -----------------------------------------------------------
	 * |   x                 y
	 * |  / \     ==>       / \
	 * | a   y             x   c
	 * |    / \           / \
	 * |   b   c         a   b  
	 * -----------------------------------------------------------
	*/
	Node* leftRotate(Node* root, Node* x) {
	    // Set y.
	    Node* y = x->right;
	 
	    // Turn y's left subtree into x's right subtree.
	    x->right = y->left;
	    if (y->left != NULL) {
	        y->left->parent = x;
	    }
	 
	    y->parent = x->parent;
	 
	    // Link x's parent to y.
	    if (x->parent == NULL) { // When x is root.
	        root = y;
	    }
	    else if (x == x->parent->left) { // When x is its parent's left child.
	        x->parent->left = y;
	    }
	    else { // When x is its parent's right child.
	        x->parent->right = y;
	    }
	 
	    // Put x on y's left.
	    y->left = x;
	    x->parent = y;
	 
	    return root;
	}
	 
	/*
	* -----------------------------------------------------------
	* |      x              y
	* |     / \    ==>     / \
	* |    y   c          a   x
	* |   / \                / \
	* |  a   b              b   c
	* -----------------------------------------------------------
	*/
	Node* rightRotate(Node* root, Node* x) {
	    // Set y.
	    Node* y = x->left;
	 
	    // Turn y's right subtree into x's left subtree.
	    x->left = y->right;
	    if (y->right != NULL) {
	        y->right->parent = x;
	    }
	 
	    y->parent = x->parent;
	    // Link x's parent to y.
	    if (x->parent == NULL) { // When x is root.
	        root = y;
	    }
	    else if (x == x->parent->left) { // When x is its parent's left child.
	        x->parent->left = y;
	    }
	    else { // When x is its parent's right child.
	        x->parent->right = y;
	    }
	 
	    // Put x on y's right.
	    y->right = x;
	    x->parent = y;
	 
	    return root;
	}
	 
	/*
	由于插入的新结点总会放到某一个叶子结点的位置并被涂为红色，所以插入后只可能破坏红黑树性质2）或性质4）。
	while 循环中要考虑 6 种情况，其中 z 的父节点 zp 是 z 的祖父结点的左孩子和右孩子各有 3 种情况，它们的代码是对称的。
	分析时，根据三种情况来：
	1）z 的叔叔 zu 是红色；
	2）z 的叔叔 zu 是黑色，而且 z 是右孩子；
	3）z 的叔叔 zu 是黑色，而且 z 是左孩子；
	*/
	Node* RBTreeInsertFixup(Node* root, Node* z) {
	    Node *zp, *zgp, *zu, *tmp; // z's parent, grandparent, uncle.
	    while ((zp = z->parent) && zp->color == RED) {
	        zgp = zp->parent;
	        if (zp == zgp->left) {
	            zu = zgp->right;
	            if (zu && zu->color == RED) {
	                zu->color = BLACK; // Case 1.
	                zp->color = BLACK; // Case 1.
	                zgp->color = RED; // Case 1.
	                z = zgp; // Case 1.
	            }
	            else {
	                if (zp->right == z) {
	                    root = leftRotate(root, zp); // Case 2.
	                    tmp = zp; // Case 2.
	                    zp = z; // Case 2.
	                    z = tmp; // Case 2.
	                }
	                zp->color = BLACK; // Case 3.
	                zgp->color = RED; // Case 3.
	                root = rightRotate(root, zgp); // Case 3.
	            } 
	        }
	        else
	{
	         
	            zu = zgp->left;
	            if (zu && zu->color == RED) {
	                zu->color = BLACK;
	                zp->color = BLACK;
	                zgp->color = RED;
	                z = zgp;
	            }
	            else {
	                if (zp->left == z) {
	                    root = rightRotate(root, zp);
	                    tmp = zp;
	                    zp = z;
	                    z = tmp;
	                }
	                zp->color = BLACK;
	                zgp->color = RED;
	                root = leftRotate(root, zgp);
	            }
	        }
	    }
	    root->color = BLACK;
	 
	    return root;
	}
	/*
	插入结点的过程类似二叉查找树的插入过程。只是在插入完成后需要调用 RBTreeInsertFixup() 来修复插入新结点后对红黑树性质的破坏。
	*/
	Node* RBTreeInsert(Node* root, Node* z) {
	    Node* y = NULL;
	    Node* x = root;
	    while (x != NULL) {
	        y = x;
	        if (z->key < x->key) {
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
	    else if (z->key < y->key) {
	        y->left = z;
	    }
	    else {
	        y->right = z;
	    }
	 
	 
	    z->left = NULL;
	    z->right = NULL;
	    z->color = RED;
	 
	    return RBTreeInsertFixup(root, z);
	 
	}
	 
	/*
	如果删除结点是红色，则不会影响红黑性质，删除后不需要调整。如果删除结点是黑色，则可能破坏红黑性质2）、性质4）、性质5）。
	while 循环中要考虑 8 种情况，其中 node 结点是左孩子和右孩子各有 4 种情况，它们的代码是对称的。
	分析时，根据四种情况来：
	1）node 的兄弟 other 是红色；
	2）node 的兄弟 other 是黑色，而且 other 的两个孩子都是黑色；
	3）node 的兄弟 other 是黑色，而且 other 的左孩子是红色，右孩子是黑色；
	4）node 的兄弟 other 是黑色，而且 other 的右孩子是红色；
	*/
	Node* RBTreeDeleteFixup(Node* root, Node* node, Node* p) {
	    Node *other, *oleft, *oright;
	    while ((!node || node->color == BLACK) && node != root) {
	        if (p->left == node) {
	            other = p->right;
	            if (other->color == RED) {
	                other->color = BLACK; // Case 1.
	                p->color = RED; // Case 1.
	                root = leftRotate(root, p); // Case 1.
	                other = p->right; // Case 1.
	            }
	            if ((!other->left || other->left->color == BLACK) && (!other->right || other->right->color == BLACK)) {
	                other->color = RED; // Case 2.
	                node = p; // Case 2.
	                p = node->parent; // Case 2.
	            }
	            else {
	                if (!other->right || other->right->color == BLACK) {
	                    if ((oleft = other->left)) { 
	                        oleft->color = BLACK; // Case 3.
	                    }
	                    other->color = RED; // Case 3.
	                    root = rightRotate(root, other); // Case 3.
	                    other = p->right; // Case 3.
	                }
	                other->color = p->color; // Case 4.
	                p->color = BLACK; // Case 4.
	                if (other->right) {
	                    other->right->color = BLACK; // Case 4.
	                }
	                root = leftRotate(root, p); // Case 4.
	                node = root; // Case 4.
	                break;
	            }
	        }
	        else
	        {
	            other = p->left;
	            if (other->color == RED) {
	                other->color = BLACK;
	                p->color = RED;
	                root = rightRotate(root, p);
	                other = p->left;
	            }
	            if ((!other->left || other->left->color == BLACK) && (!other->right || other->right->color == BLACK)) {
	                other->color = RED;
	                node = p;
	                p = node->parent;
	            }
	            else {
	                if (!other->left || other->left->color == BLACK) {
	                    if ((oright = other->right)) {
	                        oright->color = BLACK;
	                    }
	                    other->color = RED;
	                    root = leftRotate(root, other);
	                    other = p->left;
	                }
	 
	                other->color = p->color;
	                p->color = BLACK;
	                if (other->left) {
	                    other->left->color = BLACK;
	                }
	                root = rightRotate(root, p);
	                node = root;
	                break;
	            }
	        }
	    }
	 
	    if (node) {
	        node->color = BLACK;
	    }
	 
	    return root;
	}
	/*
	删除结点的过程类似二叉查找树的删除过程。只是当被删除的结点是黑色时要在删除结点后调用 RBTreeDeleteFixup() 来修复删除结点后对红黑树性质的破坏。
	*/
	Node* RBTreeDelete(Node* root, Node* z) {
	    Node *zc, *zp, *zl; // z's child, parent, left.
	    Node* old;
	    Color color;
	 
	    old = z;
	 
	    if (z->left && z->right) {
	        z = z->right;
	        while ((zl = z->left) != NULL) {
	            z = zl;
	        }
	        zc = z->right;
	        zp = z->parent;
	        color = z->color;
	        if (zc) {
	            zc->parent = zp;
	        }
	        if (zp) {
	            if (zp->left == z) {
	                zp->left = zc;
	            }
	            else {
	                zp->right = zc;
	            }
	        }
	        else {
	            root = zc;
	        }
	        if (z->parent == old) {
	            zp = z;
	        }
	        z->parent = old->parent;
	        z->color = old->color;
	        z->right = old->right;
	        z->left = old->left;
	        if (old->parent) {
	            if (old->parent->left == old) {
	                old->parent->left = z;
	            }
	            else {
	                old->parent->right = z;
	            }
	        }
	        else {
	            root = z;
	        }
	        old->left->parent = z;
	        if (old->right) {
	            old->right->parent = z;
	        }
	    }
	    else {
	        if (!z->left) {
	            zc = z->right;
	        }
	        else if (!z->right) {
	            zc = z->left;
	        }
	        zp = z->parent;
	        color = z->color;
	        if (zc) {
	            zc->parent = zp;
	        }
	        if (zp) {
	            if (zp->left == z) {
	                zp->left = zc;
	            }
	            else {
	                zp->right = zc;
	            }
	        }
	        else {
	            root = zc;
	        }
	    }
	    free(old);
	 
	    if (color == BLACK) {
	        root = RBTreeDeleteFixup(root, zc, zp);
	    }
	 
	    return root;
	}
	 
	 
	 
	int main() {
	 
	    int i, count = 20;
	    int key;
	    Node* root = NULL, *node = NULL;
	    srand(time(NULL));
	    for (i = 1; i < count; ++i) {
	        key = rand() % count;
	        Node* n = RBNodeAlloc(key, i);
	        if ((root = RBTreeInsert(root, n))) {
	            printf("[i = %d] insert key %d success!\n", i, key);
	        }
	        else {
	            printf("[i = %d] insert key %d error!\n", i, key);
	            exit(-1);
	        }
	 
	        inorderTreeWalk(root);
	        printf("\n");
	 
	        if ((node = RBTreeSearch(root, key))) {
	            printf("[i = %d] search key %d success!\n", i, key);
	        }
	        else {
	            printf("[i = %d] search key %d error!\n", i, key);
	            exit(-1);
	        }
	 
	        if (!(i % 10)) {
	            if ((root = RBTreeDelete(root, n))) {
	                printf("[i = %d] delete key %d success\n", i, key);
	            }
	            else {
	                printf("[i = %d] delete key %d error\n", i, key);
	            }
	        }
	 
	        inorderTreeWalk(root);
	        printf("\n");
	    }
	 
	    return 0;
	}


[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{page.title}})