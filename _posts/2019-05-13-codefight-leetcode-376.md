---
layout: post
title:  "Leetcode-375-猜数字大小 II"
date:   2019-05-06 17:43:00 +0700
categories: [algorithm, dynamic programming]
---

## 题目
我们正在玩一个猜数游戏，游戏规则如下：

我从 1 到 n 之间选择一个数字，你来猜我选了哪个数字。

每次你猜错了，我都会告诉你，我选的数字比你的大了或者小了。

然而，当你猜了数字 x 并且猜错了的时候，你需要支付金额为 x 的现金。直到你猜到我选的数字，你才算赢得了这个游戏。


#### 示例
```
n = 10, 我选择了8.

第一轮: 你猜我选择的数字是5，我会告诉你，我的数字更大一些，然后你需要支付5块。
第二轮: 你猜是7，我告诉你，我的数字更大一些，你支付7块。
第三轮: 你猜是9，我告诉你，我的数字更小一些，你支付9块。

游戏结束。8 就是我选的数字。

你最终要支付 5 + 7 + 9 = 21 块钱。
```
给定 n ≥ 1，计算你至少需要拥有多少现金才能确保你能赢得这个游戏。

## 思路
使用动态规划解决.
对于一个`1 ~ n`的数字, 考虑如何取一个值`x`使得`f(1~x-1) + x + f(x+1~n)`最小.
为此我们需要由底向上构建答案, 新建一个二维数组`dp[n][n]`用于保存答案, `dp[i][j]`的含义是在`i~j`范围内的问题的解.

## 代码
```java
public int getMoneyAmount(int n) {
	int[][] arr = new int[n + 1][n + 1];
	for (int i = 2; i <= n; i++) {
		for (int j = 1; j + i - 1 <= n; j++) {
			int temp = Integer.MAX_VALUE;
			int end = j + i -1;
			for (int k = j; k <= end; k++) {
				int res;
				if (k == end) { // 边界处理
					res = arr[j][k - 1] + k;
				} else if (k == j) {
					res = k + arr[k + 1][end];
				} else {
					res = Math.max(arr[j][k - 1], arr[k + 1][end]) + k;
				}
				if (temp > res) {
					temp = res;
				}
			}
			if (temp != Integer.MAX_VALUE) arr[j][end] = temp;
		}
	}
	return arr[1][n];
}
```
