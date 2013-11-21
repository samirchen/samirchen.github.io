---
layout: post
title: 基数排序【算法与数据结构-5】
description: 基数排序的介绍及代码实现。
category: blog
---

基数排序假设对 N 个数据元素进行排序，每个元素的排序关键字都有 d 位，则可以对这 N 个数据元素从高位到低位或者从低位到高位依次内排序并最终实现对这 N 个整数的排序。假设 a 是长度为 N 的数组，a 中每个数据元素排序关键字都有 d 位，其中第 1 位是最低位，第 d 位是最高位。

基数排序对其子排序的要求是：稳定排序。这一点是很必要的，举个例子来看就明白了：
对 29, 23, 24 这三个数从低位到高位来进行基数排序，并且子排序不稳定。对个位进行排序的时候，三个数在个位上的值各不相同，排序过程是没有问题的。在对十位进行排序时，三个数在十位上的值都是一样的，如果子排序不稳定，就会造成它们的相对顺序可能发生变化，那么就可能导致最终的排序结果不正确。

<div>
<table width="30%" border="1" cellspacing="0" cellpadding="2">
<tbody>
<tr>
<td valign="top">29</td>
<td valign="top">2<span style="color: #e30000;">3</span></td>
<td valign="top"><span style="color: #e30000;">2</span>4</td>
</tr>
<tr>
<td valign="top">23</td>
<td valign="top">2<span style="color: #e30000;">4</span></td>
<td valign="top"><span style="color: #e30000;">2</span>3</td>
</tr>
<tr>
<td valign="top">24</td>
<td valign="top">2<span style="color: #e30000;">9</span></td>
<td valign="top"><span style="color: #e30000;">2</span>9</td>
</tr>
</tbody>
</table>
</div>

基数排序通常都是按照从低位到高位的顺序来排序，因为高位的权重大于低位，这样在对高位排序的过程中就不必考虑和记录低位的信息。而从高位到低位来排序就不行了，举个例子看看：
对 94, 19, 83 这三个数从高位到低位来进行基数排序，并且子排序稳定。对十位进行排序的时候，并没有什么问题。对个位进行排序的时候，如果没有考虑和记录十位的信息，那么排序结果就如下面所列，是不正确的。

<div>
<table width="30%" border="1" cellspacing="0" cellpadding="2">
<tbody>
<tr>
<td valign="top">94</td>
<td valign="top"><span style="color: #e30000;">1</span>9</td>
<td valign="top">8<span style="color: #e30000;">3</span></td>
</tr>
<tr>
<td valign="top">19</td>
<td valign="top"><span style="color: #e30000;">8</span>3</td>
<td valign="top">9<span style="color: #e30000;">4</span></td>
</tr>
<tr>
<td valign="top">83</td>
<td valign="top"><span style="color: #e30000;">9</span>4</td>
<td valign="top">1<span style="color: #e30000;">9</span></td>
</tr>
</tbody>
</table>
</div>

- 时间复杂度：依赖于子排序算法的时间复杂度
- 空间复杂度：依赖于子排序算法的空间复杂度
- 稳定

伪代码
	
	radixSort(A, d)
	     for i from 1 to d
	           do use a stable sort to sort array A on digit i

通常结合基数排序和计数排序，把计数排序作为基数排序的子排序，是一种不错的选择。这样可以用来对时间、数字等等数据进行排序。可以实现时间复杂度为 `O(N)`，空间复杂度为 `O(N)` 的排序算法。


[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{page.title}})