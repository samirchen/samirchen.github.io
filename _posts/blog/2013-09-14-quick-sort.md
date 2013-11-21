---
layout: post
title: 快速排序【算法与数据结构-1】
description: 快速排序的介绍及代码实现。
category: blog
---

快速排序（`Quick Sort`）是对冒泡排序的一种改进。由C. A. R. Hoare在1962年提出。它的基本思想是：通过一趟排序将要排序的数据分割成独立的两部分，其中一部分的所有数据都比另外一部分的所有数据都要小，然后再按此方法对这两部分数据分别进行快速排序，整个排序过程可以递归进行，以此达到整个数据变成有序序列。

- 时间复杂度：平均 `O(NlgN)`；最坏 `O(N^2)`
- 空间复杂度：`O(1)`
- 不稳定

C 语言代码：

[到github去获取代码](https://github.com/samirchen/algorithms/blob/master/sort/quickSort.c)

	#include<stdio.h>
	void swap(int* x, int* y) {
	     int tmp = *x;
	     *x = *y;
	     *y = tmp;
	}
	int partition(int a[], int p, int r) {
	    int x = a[r];
	    int i = p - 1;
	    for (int j = p; j <= r - 1; j++) {
	        if (a[j] <= x) {
	            i++;
	            swap(&a[i], &a[j]);
	        }
	    }
	 
	    swap(&a[i+1], &a[r]);
	 
	    return i+1;
	}
	void quickSort(int a[], int p, int r) {
	    if (p < r) {
	        int q = partition(a, p, r);
	        quickSort(a, p, q-1);
	        quickSort(a, q+1, r);
	    }
	}
	int main() {
	    int a[10] = {1, 2, 10, 9, 7, 8, 6, 5, 3, 4};
	    quickSort(a, 0, 9);
	 
	    int i = 0;
	    for (i = 0; i < 10; i++) {
	        printf("%d ", a[i]);
	    }
	    printf("\n");
	 
	    return 0;
	}



[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{page.title}})