---
layout: post
title: 动态规划【算法与数据结构-9】
description: 动态规划的介绍及一些示例的代码实现。
category: blog
---

##1、动态规划的思想：

动态规划通过组合子问题的解而解决整个问题。这点我们可以对比一下分治算法，分治法算法是指将问题划分成一些独立的子问题，递归地求解子问题，然后合并子问题的解而得到原问题的解。与此不同，动态规划适用于子问题不是独立的情况，也就是各子问题包含公共的子子问题。在这种情况下，若用分治法则会做许多不必要的工作，即重复地求解公共的子子问题。动态规划对每个子子问题只求解一次，将其结果保存在一张表中，从而避免每次遇到各个子问题时重新计算答案。动态规划通常用于最优化问题。

##2、动态规划的适用条件：

- 1）一个问题的最优解包含了子问题的一个最优解，这个性质叫做“最优子结构”，这是动态规划的适用条件之一。
- 2）一个递归算法在其递归的不同分支中可能会多次遇到同一个子问题，这个性质叫做“子问题重叠”，这是动态规划的另一个适用条件。

##3、动态规划算法设计的 4 个步骤：

- 1）描述最优解的结构。
- 2）递归定义最优解的值。自上向下看问题得到递归思路。
- 3）按自底向上的方式计算最优解的值。用合适的数据结构处理中间值。
- 4）由计算出的结果构造一个最优解。即构造出 DP 解法。

##4、非正式的，一个动态规划算法的运行时间依赖于：

**（子问题的总个数）*（每个子问题中有多少种选择）**

##5、用动态规划的思想来解决“最长公共子序列（LCS）”问题：

###1）最长公共子序列问题描述

子序列：一个给定序列的子序列就是该给定序列中去掉零个或者多个元素。比如，Z={B, C, D, B} 就是 X={A, B, C, B, D, A, B} 的一个子序列，相应的下标序列为 {2, 3, 5, 7}。
公共子序列：给定两个序列 X 和 Y，如果序列 Z 既是 X 的一个子序列又是 Y 的一个子序列，则称序列 Z 是 X 和 Y 的公共子序列。
最长公共子序列：长度最长的公共子序列。

###2）用动态规划来解决 LCS 问题

2.1）描述一个最优解的结构
两个序列的一个 LCS 也包含了两个序列的前缀的一个 LCS。这就说明 LCS 问题具有最优子结构性质。

通过证明下面定理可以证明 LCS 问题的最优子结构性质：

设 X = {x1, x2, …, xm} 和 Y = {y1, y2, …, yn} 为两个序列，并设 Z = {z1, z2, …, zk} 为 X 和 Y 的任意一个 LCS。

- a）如果 xm = yn，那么 zk = xm = yn 而且 Z(k-1) 是 X(m-1) 和 Y(n-1) 的一个 LCS。
- b）如果 xm != yn, 那么 zk != xm 蕴含 Z 是 X(m-1) 和 Y 的一个 LCS。
- c）如果 xm != yn, 那么 zk != yn 蕴含 Z 是 X 和 Y(n-1) 的一个 LCS。

2.2）递归定义最优解的值

定义 c[i, j] 为序列 Xi 和 Yj 的一个 LCS 的长度。由 LCS 问题的最优子结构可得递归式：
c[i, j] =
0; 如果 i = 0 或 j = 0;
c[i-1, j-1]+1; 如果 i,j > 0 和 xi = yj;
max{c[i, j-1], c[i-1, j]}; 如果i,j > 0 和 xi != yj;

2.3）计算最优解的值

递归：自上向下看问题，根据上面的递归式可以很容易地写出一个指数时间的递归算法来计算两个序列的 LCS 的长度。（计算递归算法时间复杂度的方法见《算法导论》第四章）
动态规划：自底向上来计算解。

C 语言代码：

[到github去获取代码](http://localhost/wordpress/wp-content/uploads/2013/05/LCS.c)

	// 《算法导论》 P211.
	#include
	using namespace std;
	 
	const int M = 7;
	char x[M + 1] = {' ', 'A', 'B', 'C', 'B', 'D', 'A', 'B'};
	const int N = 6;
	char y[N + 1] = {' ', 'B', 'D', 'C', 'A', 'B', 'A'};
	int c[M + 1][N + 1];
	 
	// 递归算法，时间复杂度 O((MN)^2)，空间复杂度 O(1)
	int recursionLCS(int m, int n) {
	 
	    if (m == 0 || n == 0) {
	        return 0;
	    }
	 
	    if (m > 0 && n > 0 && x[m] == y[n]) {
	        return recursionLCS(m - 1, n - 1) + 1;
	    }
	 
	    if (m > 0 && n > 0 && x[m] != y[n]) {
	        int tmp1 = recursionLCS(m, n - 1);
	        int tmp2 = recursionLCS(m - 1, n);
	        return tmp1 > tmp2 ? tmp1 : tmp2;
	    }
	 
	    return 0;
	}
	 
	// 动态规划算法，时间复杂度 O(MN)，空间复杂度O(MN)
	void dpLCS() {
	 
	    for (int i = 1; i <= M; i++) {
	        c[i][0] = 0;
	    }
	 
	    for (int j = 0; j <= N; j++) {
	        c[0][j] = 0;
	    }
	 
	    for (int i = 1; i <= M; i++) {
	        for (int j = 1; j <= N; j++) {                          
	            if (x[i] == y[j]) {                                  
	                c[i][j] = c[i - 1][j - 1] + 1;             
	            }                          
	            else if (c[i - 1][j] >= c[i][j - 1]) {
	                c[i][j] = c[i - 1][j];
	            }
	            else {
	                c[i][j] = c[i][j - 1];
	            }
	        }
	 
	    }
	 
	}
	 
	// 构造一个最优解，时间复杂度 O(M+N)
	void dpPrintLCS(int i, int j) {
	 
	    if (i == 0 || j == 0) {
	        return;
	    }
	 
	    if (x[i] == y[j]) {
	        dpPrintLCS(i - 1, j - 1);
	        cout << x[i] << " ";     
	    }          
	    else if (c[i - 1][j] >= c[i][j - 1]) {
	        dpPrintLCS(i - 1, j);
	    }
	    else {
	        dpPrintLCS(i, j - 1);
	    }
	 
	}
	 
	int main() {
	    cout << recursionLCS(M, N) << endl;
	 
	    dpLCS();
	    cout << c[M][N] << endl;
	    dpPrintLCS(M, N);
	    cout << endl;
	 
	    return 0;
	 
	}

2.4）构造一个最优解

通过 2.3） 的 dpPrintLCS(int i, int j) 方法构造出一个 LCS。


##6、用动态规划的思想来解决“数组的子数组之和的最大值”问题

###1）数组的子数组之和的最大值问题描述

子数组：数组 a[0], a[1], …, a[n-2], a[n-1] 的子数组是 a[i], a[i+1], …, a[j-1], a[j] (0 <= i <= j <= n-1，其中 a[i] 到 a[j] 的所有元素下标是连续的)。

数组的子数组之和的最大值：一个有 N 个元素的一维数组（a[0], a[1], ..., a[n-2], a[n-1]），这个数组有很多子数组，此问题即找出其中一个子数组，它的和在所有子数组中最大。

###2）用动态规划来解决数组的子数组之和的最大值问题

2.1）描述一个最优解的结构

数组 a[0], a[1], ..., a[n-2], a[n-1]；

设子数组 a[i], ..., a[n-2], a[n-1] 中包含了 a[i] （也就是从 a[i] 开始） 的和最大的一段子数组（这段子数组不一定是 a[i], ..., a[n-2], a[n-1] 中和最大的子数组）的和为 b[i]；

设 c[i] 是 a[i], …, a[n-2], a[n-1] 中和最大的子数组的和；

那么 c[i-1]=max{a[i-1], a[i-1]+b[i], c[i]}。

2.2）递归定义最优解

c[i-1] = max{a[i-1], a[i-1]+b[i], c[i]}

2.3）计算最优解的值

C 语言代码：

[到github去获取代码](http://localhost/wordpress/wp-content/uploads/2013/05/max_subarray_sum.c)

	// Find the X-element sub-array which has the max sum of a N-element integer array.
	#include <stdio.h>
	 
	#define N 10
	 
	int method(int a[]);
	 
	int main() {
	    int a[N] = {-2, 5, 3, -6, 4, -8, 6, 1, -2, -1};
	 
	    int result = method(a);
	    printf("%d\n", result);
	}
	 
	int max(int x, int y) {
	    return x > y ? x : y;
	}
	// 动态规划算法，时间复杂度O(N)，空间复杂度O(N)（空间复杂度可以再优化到O(1)）
	int method(int* a) {
	    int c[N] = {0};
	    int b[N] = {0};
	 
	    c[N-1] = a[N-1];
	    b[N-1] = a[N-1];
	 
	    int i = 0;
	    for (i = N-2; i >= 0; i--) {
	        b[i] = max(a[i], a[i]+b[i+1]);
	        c[i] = max(c[i+1], b[i]);
	    }
	    return c[0];
	}

2.4）构造最优解

略。


[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{page.title}})