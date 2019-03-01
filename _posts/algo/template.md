---
layout: post
title: 寻找数组中第K大的元素
categories: [算法]
description: 寻找数组中第K大的元素
keywords: quickSort, quickSelect
---

## 问题

```
Find the kth largest element in an unsorted array. Note that it is the kth largest element in the sorted order, not the kth distinct element.

Example 1:

Input: [3,2,1,5,6,4] and k = 2
Output: 5
Example 2:

Input: [3,2,3,1,2,4,5,5,6] and k = 4
Output: 4
Note: 
You may assume k is always valid, 1 ≤ k ≤ array's length.
```

**tag: Medium**

## 分析
这题最简单的做法是将数组排序，然后直接返回第K大的元素。复杂度为：O(NlogN)。但是，很明显，出题者并不想让我们这么做。

如果对数组排序，算法的复杂度起码是 O(NlogN)。那么如果我们不排序，能不能求出第K大元素呢？答案是可以的，我们知道快速排序中有一个步骤是快速选择（quickSelect）。它选择一个枢纽元素（pivot），将所有小于枢纽的元素放到枢纽的左边，将所有大于枢纽的元素放到枢纽的右边。然后返回的是枢纽在数组中的位置。