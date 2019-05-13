---
layout: post
title:  "Leetcode-376-摆动序列"
date:   2019-05-13 20:43:00 +0700
categories: [algorithm, dynamic programming]
---

## 题目
如果连续数字之间的差严格地在正数和负数之间交替，则数字序列称为摆动序列。第一个差（如果存在的话）可能是正数或负数。少于两个元素的序列也是摆动序列。

例如， [1,7,4,9,2,5] 是一个摆动序列，因为差值 (6,-3,5,-7,3) 是正负交替出现的。相反, [1,4,7,2,5] 和 [1,7,4,5,5] 不是摆动序列，第一个序列是因为它的前两个差值都是正数，第二个序列是因为它的最后一个差值为零。

给定一个整数序列，返回作为摆动序列的最长子序列的长度。 通过从原始序列中删除一些（也可以不删除）元素来获得子序列，剩下的元素保持其原始顺序。


#### 示例
```
输入: [1,7,4,9,2,5]
输出: 6 
解释: 整个序列均为摆动序列。

输入: [1,17,5,10,13,15,10,5,16,8]
输出: 7
解释: 这个序列包含几个长度为 7 摆动序列，其中一个可为[1,17,10,13,10,16,8]。

输入: [1,2,3,4,5,6,7,8,9]
输出: 2
```

## 思路
可以使用动态规划, 然而本题存在贪心解如下.

对于一个长度为N的数组arr,假设它的一个最大摆动序列对应原数组的下标为[a1, a2, ..., ak],其中k为此摆动序列的长度.我们可以证明任意两个相邻的ax与ay之间,不失一般性假设arr[ax] < arr[ay] (x <= z <= y), 必然存在`arr[az] < arr[az + 1]`, 如果表达式不成立, 那么必然存在一个包含az的更长的摆动序列, 与假设不符, 故不存在这样的az.

依照这样的规律我们可以得到一个贪心解法.

## 代码
```java
static final boolean UP = true;
static final boolean DOWN = false;
public int wiggleMaxLength(int[] nums) {
	if (nums == null || nums.length == 0) return 0;
	if (nums.length == 1) return 1;
	int i = getFirstValidIndex(nums);
	if (i == nums.length - 1) return 1;
	boolean cur = nums[i] > nums[i + 1] ? UP : DOWN;
	int result = 1;
	while (i != nums.length - 1) {
		if (cur == UP) {
			int pre = nums[i++];
			while (i != nums.length - 1 && nums[i] < pre && nums[i] >= nums[i + 1]) {
				i++;
			}
			cur = DOWN;
			result++;
		} else {
			int pre = nums[i++];
			while (i != nums.length - 1 && nums[i] > pre && nums[i] <= nums[i + 1]) {
				i++;
			}
			cur = UP;
			result++;
		}
	}
	return result;
}

public int getFirstValidIndex(int[] nums) {
	int result = 0;
	for (;;) {
		if (result == nums.length - 1 || nums[result] != nums[result + 1]) 
			break;
		result++;
		continue;
	}
	return result;
}
```
