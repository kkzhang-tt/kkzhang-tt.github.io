---

layout: post
title: "编程珠玑: 算法设计技术"
author: "kkzhang"
category: algrithom

---

## 问题描述

**输入**：具有 `n` 个浮点数的向量 `x`。

**输出**：输入向量的任意连续子向量的最大和。

**示例**：输入向量 `x` 为 [31, -41, 59, 26, -53, 58, 97, -93, -23, 84]，那么该程序输出为 x[2...6] 的总和。

## 分析

当所有数都是正数时，问题比较简单，此时最大子向量就是整个输入向量。

当输入向量中含有负数时，是否应该包含某个负数并期望旁边的正数会弥补它呢？

为了使问题定义更加完整，假设所有的输入都是负数时，最大子向量为空向量，总和为 0.

## O(n^3) 算法

该问题最直接的解法是对所有满足 `0 ≤ i ≤ j < n` 的 `(i, j)` 整数对进行迭代。对每个整数对，都需要计算 `x[i, j]` 的总和，并判断此时该和是否是迄今为止最大的和。

代码比较简单，但是运行速度很慢。

```c
maxsofar = 0
for i = [0, n)
	for j = [i, n)
		sum = 0 // sum is sum of x[i...j]
		for k = [i, j]
			sum += x[k]
		maxsofar = max(maxsofar, sum)
```

## O(n^2) 算法

1. 我们注意到，`x[i...j]` 的总和与前面已经计算出的总和 `x[i...j-1]` 相关，利用该关系可以得到第一个平方算法。
    
    ```c
    maxsofar = 0
    for i = [0, n)
    	sum = 0 // sum is sum of x[i...j]
    	for j = [i, n)
    		sum += x[j]
    		maxsofar = max(max, maxsofar)
    ```
    
2. 另一种平方算法是提前计算 `x[0...i]` 各个数累加的和。`cumarr[i]` 表示 `x[0...i]` 中各个数的累加和，则 `x[i..j]` 的累加和可以通过 `cumarr[j] - cumarr[i-1]` 得到。
    
    ```c
    cumarr[-1] = 0
    for i = [0, n)
    	cumarr[i] = cumarr[i-1] + x[i]
    maxsofar = 0
    for i = [0, n)
    	for j = [i, n)
    		sum = cumarr[j] -  cumarr[i-1] // sum is sum of x[i...j]
    		maxsofar = max(max, maxsofar)
    ```
    

## 分治算法：O(nlogn)

前面讨论的算法考虑了所有的子向量，并计算每个子向量中的所有数的和。由于存在 O(n^2) 个子向量，因此至少需要 O(n^2) 的运行时间。

**分治法原理**：*要解决规模为 n 的问题，可以递归地解决两个规模近似为 n/2 的子问题，然后对它们的答案进行合并可以得到整个问题的答案*。

在本问题中，我们将大小为 n 的向量分解为两个大小近似的子向量 a, b，之后递归地找出 a, b 中的最大子向量 ma, mb。

- 整个向量的最大子向量为 ma：子向量在 a 中
- 整个向量的最大子向量为 mb：子向量在 b 中
- 整个向量的最大子向量为 mc：子向量跨越 a, b 之间的边界

```c
float maxsum(l, u)
	if(l > u) return 0; // zero elements
	if(l == u) return max(0, x[l]) // one element
	m = (l+u)/2
	// find max crossing to left
	lmax = sum = 0
	for(i = m; i >= 1; i--)
		sum += x[i]
		lmax = max(lmax, sum)
	// find max crossing to right
	rmax = sum = 0
	for(i = m; i < u; i++)
		sum += x[i]
		rmax = max(rmax, sum)
return max(lmax+rmax, maxsum(l, m), maxsum(m+1, u))
```

## 扫描算法：O(n)

扫描算法是从数组的最左端（x[0]）开始扫描，一直到最右端（x[n-1]），并记下所遇到的总和最大的子向量。

假设已经解决了 x[0...i-1] 的问题，如何将其扩展为包含 x[i] 的问题？

- 类似分治算法原理：前 i 个元素中，最大总和的子向量要么在前 i-1 个元素中（maxsofar），要么其结束位置为 i（maxendinghere）

    |  | maxsofar |  | maxendinghere |
    | --- | --- | --- | --- |
    ||||i|


```c
maxsofar = 0
maxendinghere = 0
for i = [0, n)
	// maxendinghere 一开始为结束位置为 x-1 的最大子向量的和，此处将其更新为结束位置为 i 的最大子向量的和
	// 如果加上 x[i] 之后仍然为正值，那么 maxendinghere 则加上 x[i]；否则将 maxendinghere 设置为 0（空向量）
	maxendinghere = max(maxendinghere + x[i], 0)
	maxsofar = max(maxsofar, maxendinghere)
```

## 原理

上述几个算法给出了几个重要的算法设计技术：

- 保存状态，避免重复计算：动态规划
- 分治算法
- 扫描算法
- 将信息预处理至数据结构中

