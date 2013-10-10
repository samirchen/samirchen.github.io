---
layout: post
title: （算法与数据结构-8）二叉树的各种遍历算法
description: 二叉树的各种遍历算法的介绍及代码实现。
category: blog
---

- 前序遍历：先访问根节点；再遍历左子树；最后遍历右子树。
- 中序遍历：先遍历左子树；再访问根节点；最后遍历右子树。
- 后序遍历：先遍历左子树；再遍历右子树；最后访问根节点。
- 层次遍历：按照树的高度，从左到右依次访问每一个节点。

C 语言代码

[到github去获取源代码](https://github.com/samirchen/algorithms/blob/master/tree/binaryTree.c)

	#include <stdio.h>
	#include <stdlib.h>
	 
	typedef struct node Node;
	 
	struct node {
	        int value;
	        Node* left;
	        Node* right;
	        Node* parent;
	};
	 
	Node* nodeAlloc() {
	        return (Node*) malloc(sizeof(Node));
	}
	 
	// Preorder walk.
	void recursionPreorderTreeWalk(Node* root) {
	        if (root != NULL) {
	                printf("%d ", root->value); // Visit.
	                recursionPreorderTreeWalk(root->left);
	                recursionPreorderTreeWalk(root->right);
	        }
	} 
	// Preorder walk. Use stack.
	void nonrecursionPreorderTreeWalk(Node* root) {
	        if (root != NULL) {
	                Node* stack[10] = {NULL}; // A simple stack.
	                int i = 0; // Top of stack. Store nothing in stack[i]. Store nodes in stack[0]~stack[i-1].
	 
	                stack[i++] = root; // Push
	                while (i > 0) {
	                        Node* node = stack[--i]; // Pop
	                        printf("%d ", node->value); // Visit.
	                        if (node->right != NULL) {
	                                stack[i++] = node->right; // Push
	                        }
	                        if (node->left != NULL) {
	                                stack[i++] = node->left; // Push
	                        }
	                }
	 
	        }
	}
	 
	 
	// Inorder walk.
	void recursionInoderTreeWalk(Node* root) {
	        if (root != NULL) {
	                recursionInoderTreeWalk(root->left);
	                printf("%d ", root->value); // Visit.
	                recursionInoderTreeWalk(root->right);
	        }
	}
	// Inorder walk. Use stack.
	void nonrecursionInorderTreeWalk(Node* root) {
	        if (root != NULL) {
	                Node* stack[10] = {NULL}; // A simple stack.
	                int i = 0; // Top of stack.
	 
	                Node* node = root;
	                while (i > 0 || node != NULL) {
	                        if(node != NULL) {
	                                stack[i++] = node; // Push
	                                node = node->left;
	                        }
	                        else {
	                                node = stack[--i]; // Pop
	                                printf("%d ", node->value); // Visit.
	                                node = node->right;
	                        }
	                }
	        }
	}
	 
	// Postorder walk.
	void recursionPostorderTreeWalk(Node* root) {
	        if (root != NULL) {
	                recursionPostorderTreeWalk(root->left);
	                recursionPostorderTreeWalk(root->right);
	                printf("%d ", root->value); // Visit.
	        }
	}
	// Postorder walk. Use stack.
	void nonrecursionPostorderTreeWalk(Node* root) {
	        if (root != NULL) {
	                Node* stack[10] = {NULL}; // A simple stack.
	                int i = 0; // Top of stack.
	 
	                Node* preVisitedNode = NULL;
	                Node* curNode = NULL;
	                stack[i++] = root; // Push
	                while (i > 0) {
	                        curNode = stack[i-1]; // Top
	                        if ((curNode->left == NULL && curNode->right == NULL) || (preVisitedNode != NULL && (preVisitedNode == curNode->left || preVisitedNode == curNode->right))) {
	                                printf("%d ", curNode->value); // Visit.
	                                i--; // Pop
	                                preVisitedNode = curNode;
	                        }
	                        else {
	                                if (curNode->right != NULL) {
	                                        stack[i++] = curNode->right; // Push
	                                }
	                                if (curNode->left != NULL) {
	                                        stack[i++] = curNode->left; // Push
	                                }
	                        }
	                }
	 
	        }
	}
	 
	// Levelorder walk. Use queue.
	void levelorderTreeWalk(Node* root) {
	        if (root != NULL) {
	                Node* queue[10] = {NULL}; // A simple queue.
	                int i = 0; // Start of queue.
	                int j = 0; // End of queue.
	 
	                queue[j++] = root; // Push
	 
	                while (i != j) {
	                        Node* node = queue[i++]; // Pop
	                        printf("%d ", node->value); // Visit.
	 
	                        if (node->left != NULL) {
	                                queue[j++] = node->left; // Push
	                        }
	 
	                        if (node->right != NULL) {
	                                queue[j++] = node->right; // Push
	                        }
	                }
	        }
	 
	}
	 
	int main() {
	        // Create tree.
	        Node* n4 = nodeAlloc();
	        n4->value = 4;
	        n4->left = NULL;
	        n4->right = NULL;
	 
	        Node* n5 = nodeAlloc();
	        n5->value = 5;
	        n5->left = NULL;
	        n5->right = NULL;
	 
	        Node* n2 = nodeAlloc();
	        n2->value = 2;
	        n2->left = n4;
	        n2->right = n5;
	 
	        Node* n6 = nodeAlloc();
	        n6->value = 6;
	        n6->left = NULL;
	        n6->right = NULL;
	 
	        Node* n3 = nodeAlloc();
	        n3->value = 3;
	        n3->left = NULL;
	        n3->right = n6;
	 
	        Node* root = nodeAlloc();
	        root->value = 1;
	        root->left = n2;
	        root->right = n3;
	 
	        root->parent = NULL;
	        n2->parent = root;
	        n3->parent = root;
	        n4->parent = n2;
	        n5->parent = n2;
	        n6->parent = n3;
	 
	        // Preorder.
	        recursionPreorderTreeWalk(root);
	        printf("\n");
	        nonrecursionPreorderTreeWalk(root);
	        printf("\n");
	 
	        // Inorder.
	        recursionInoderTreeWalk(root);
	        printf("\n");
	        nonrecursionInorderTreeWalk(root);
	        printf("\n");
	 
	        // Postorder.
	        recursionPostorderTreeWalk(root);
	        printf("\n");
	        nonrecursionPostorderTreeWalk(root);
	        printf("\n");
	 
	        // Levelorder
	        levelorderTreeWalk(root);
	        printf("\n");
	 
	        free(root);
	        free(n2);
	        free(n3);
	        free(n4);
	        free(n5);
	        free(n6);
	 
	        return 0;
	}