---
layout: post
title: 归并排序【算法与数据结构-2】
description: 归并排序的介绍及代码实现。
category: blog
---

归并排序（`Merge Sort`）是将两个（或两个以上）有序表合并成一个新的有序表，即把待排序序列分为若干个子序列，每个子序列是有序的。然后再把有序子序列合并为整体有序序列。

- 时间复杂度：`O(NlgN)`
- 空间复杂度：`O(N)`
- 稳定

C 语言代码：

[到github去获取源代码](https://github.com/samirchen/algorithms/blob/master/sort/mergeSort.c)

	#include <stdio.h>
	#include <stdlib.h>
	void merge(int* a, int p, int q, int r) {
	    int n1 = q - p + 1;
	    int n2 = r - q;
	 
	    int* L = (int*) malloc(n1 * sizeof(int)); // 'malloc' requires 'stdlib.h'.
	    int* R = (int*) malloc(n2 * sizeof(int));
	 
	    int i = 0;
	    int j = 0;
	    for (i = 0; i < n1; i++) {
	        L[i] = a[p + i];
	    }
	    for (j = 0; j < n2; j++) {
	        R[j] = a[q + 1 + j];
	    }
	 
	    i = j = 0;
	    int k = p;
	    for (k = p; k <= r; k++) {
	 
	        if (i >= n1 && j < n2) { // L 已经遍历完了，R 还没遍历完。
	            a[k] = R[j];
	            j++;
	        }
	 
	        if (j >= n2 && i < n1) { // R 已经遍历完了，L 还没遍历完。
	            a[k] = L[i];
	            i++;
	        }
	 
	        if (i < n1 && j < n2) { // R，L 都没遍历完。
	            if (L[i] <= R[j]) {
	                a[k] = L[i];
	                i++;
	            }
	            else {
	                a[k] = R[j];
	                j++;
	            }
	        }
	    }
	 
	    free(L);
	    free(R);
	 
	}
	void mergeSort(int* a, int p, int r) {
	   if (p < r) {
	        int q = (p + r) / 2;
	        mergeSort(a, p, q);
	        mergeSort(a, q + 1, r);
	        merge(a, p, q, r);
	    }
	}
	int main() {
	    int a[10] = {1, 2, 10, 9, 7, 8, 6, 5, 3, 4};
	    mergeSort(a, 0, 9);
	 
	    int i = 0;
	    for (i = 0; i < 10; i++) {
	        printf("%d ", a[i]);
	    }
	    printf("\n");
	 
	    return 0;
	}


[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{page.title}})