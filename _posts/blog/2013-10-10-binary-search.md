---
layout: post
title: （算法与数据结构-6）二分查找（折半查找）
description: 二分查找的介绍及代码实现。
category: blog
---

二分查找(`Binary Search`)又称折半查找，优点是比较次数少，查找速度快，平均性能好；其缺点是要求待查表为有序表，且插入删除困难。因此，折半查找方法适用于不经常变动而查找频繁的有序列表。首先，假设表中元素是按升序排列，将表中间位置记录的关键字与查找关键字比较，如果两者相等，则查找成功；否则利用中间位置记录将表分成前、后两个子表，如果中间位置记录的关键字大于查找关键字，则进一步查找前一子表，否则进一步查找后一子表。重复以上过程，直到找到满足条件的记录，使查找成功，或直到子表不存在为止，此时查找不成功。

- 时间复杂度：`O(lgN)`
- 空间复杂度：`O(1)`

C 语言代码

[到github去获取源代码](https://github.com/samirchen/algorithms/blob/master/search/binarySearch.c)

	#include <stdio.h>
	int binarySearch(int a[], int N, int key) {
	    int low = 0;
	    int high = N - 1;
	    int mid = (high+low)/2;
	    while (low <= high) {
	        mid = (high+low)/2;
	        if (a[mid] == key) {
	            return mid;
	        }
	        else if (a[mid] > key) {
	            high = mid -1;
	        }
	        else {
	            low = mid + 1;
	        }
	 
	    }
	 
	    return -1;
	}
	int main() {
	     int a[10] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
	     int index = binarySearch(a, 3, 10);
	     printf("Index of search result: %d\n", index);
	 
	     return 0;
	}