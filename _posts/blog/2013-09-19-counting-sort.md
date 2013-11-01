---
layout: post
title: （算法与数据结构-4）计数排序
description: 计数排序的介绍及代码实现。
category: blog
---

计数排序通常假设 N 个输入数据元素的排序关键字中的每一个都是介于 0 到 k 之间的整数，此处 k 为某个整数。计数排序的基本思想是对每一个输入元素 x，确定出小于 x 的元素个数，这样就可以把 x 直接放到它在最终输出数组中的位置上。例如，有 5 个元素小于 x，那么 x 就应该排在第 6 个位置上输出。假设 a 是长度为 N 的数组；b 是长度为 N 存放排序结果的数组；c 是长度为 k+1 存放计数统计的数组。

- 时间复杂度：`O(k+N)`，通常 k 是常数，就是 `O(N)`
- 空间复杂度：`O(k+N)`，通常 k 是常数，就是 `O(N)`
- 稳定

C 语言代码

[到github去获取源代码](https://github.com/samirchen/algorithms/blob/master/sort/countingSort.c)

	#include <stdio.h>
	#include <stdlib.h>
	 
	void countingSort(int* a, int* b, int k, int N) {
	    int* c = (int*) malloc((k+1) * sizeof(int));
	    int i = 0;
	    for (i = 0; i <= k; i++) {
	        c[i] = 0;
	    }
	 
	    for (i = 0; i < N; i++) {
	        c[a[i]] = c[a[i]] + 1;
	    }
	 
	    for (i = 1; i <= k; i++) {
	        c[i] = c[i-1] + c[i];
	    }
	 
	    for (i = N-1; i >= 0; i--) {
	        b[ c[a[i]]-1] = a[i]; // Index should be "c[a[i]]-1", index of b begin from 0, count of a[i] begin from 1.
	        c[a[i]] = c[a[i]] - 1;
	    }
	}
	 
	int main() {
	 
	    // All elements are in [0, 9].
	    int a[10] = {9, 8, 1, 2, 3, 9, 2, 5, 2, 4};
	    int b[10] = {0};
	 
	    countingSort(a, b, 9, 10);
	 
	    int i = 0;
	    for (i = 0; i < 10; i++) {
	        printf("%d ", b[i]);
	    }
	    printf("\n");
	 
	    return 0;
	}

[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{page.title}})