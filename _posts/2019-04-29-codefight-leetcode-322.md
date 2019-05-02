---
layout: post
title:  "Leetcode-322-零钱兑换"
date:   2019-05-02 13:39:00 +0700
categories: [algorithm, dynamic programming]
---

## 题目
给定不同面额的硬币 coins 和一个总金额 amount。编写一个函数来计算可以凑成总金额所需的最少的硬币个数。如果没有任何一种硬币组合能组成总金额，返回 `-1`.

#### 示例1
```
Input：coins = [1, 2, 5], amount = 11
Output: 3
Explain: 11 = 5 + 5 + 1
```

## 思路
可以使用动态规划的思想来解题,使用一个数组`dp[i][j]`,`dp[i][j]`的意思是使用`coins[0...i]`的硬币凑成总金额j所需最少的硬币个数.

递推关系:`dp[i][j] = min{dp[i-1][j]`, (使用最大`amount/coins[i]`个硬币中所需最少的那一种)}

上面的式子可能不太好懂,现在来说明一下.对于总金额`j`,我们通过选择不同个数个`coins[i]`会得到不同的结果,那么我们可以选择从0个`coins[i]`到`j/coins[i]`个,假设选择了k个`coins[i]`,那么此时的所需硬币数为:
```
dp[i - 1][j - k*coins[i]] + k
```
对k的所有取值中选择使硬币数最少的那个即可,这样的时间复杂度是N<sup>3</sup>,但是我们可以观察得到`dp[i-1][j-coins[i]]`的值即为k的取值范围为`0 - (j-coins[i]) / coins[i]`时的结果,那么我们就可以利用这一点使时间复杂度降低到N<sup>2</sup>.

下面是题目的代码解答

## 代码
```java
public int coinChange(int[] coins, int amount) {
	if (coins == null || coins.length == 0 || amount == 0) return 0;
	int[][] arr = new int[coins.length][amount + 1];
	for (int i = 0; i <= amount; i++) {
		if (i % coins[0] == 0) arr[0][i] = i / coins[0];
		else arr[0][i] = Integer.MAX_VALUE;
	}
	int result = arr[0][amount];
	for (int i = 1; i < coins.length; i++) {
		for (int j = 0, k = 0; j <= amount; j++, k++) {
			if (j < coins[i]) {
				arr[i][j] = arr[i - 1][j];
			} else {
				arr[i][j] = Math.min(arr[i - 1][j] - 1, arr[i][j - coins[i]]) + 1;
			}
			if (j == amount) {
				result = Math.min(result, arr[i][j]);
			}
		}
	}
	
	if (amount != 0 && result == Integer.MAX_VALUE) {
		return -1;
	} else {
		return result;
	}
}
```
