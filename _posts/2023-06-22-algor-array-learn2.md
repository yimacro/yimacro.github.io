---
layout: post

title: 算法：链表学习4

categories: [数据结构算法]


---



#### [283. 移动零](https://leetcode.cn/problems/move-zeroes/)

给定一个数组 `nums`，编写一个函数将所有 `0` 移动到数组的末尾，同时保持非零元素的相对顺序。

**请注意** ，必须在不复制数组的情况下原地对数组进行操作。

```
输入: nums = [0,1,0,3,12]
输出: [1,3,12,0,0]
```

ps:

* 借用前面remove函数的思路，在移动完成后进行对数组的末尾进行添0操作



#### [167. 两数之和 II - 输入有序数组](https://leetcode.cn/problems/two-sum-ii-input-array-is-sorted/)

给你一个下标从 1 开始的整数数组 numbers ，该数组已按 非递减顺序排列  ，请你从数组中找出满足相加之和等于目标数 target 的两个数。如果设这两个数分别是 numbers[index1] 和 numbers[index2] ，则 1 <= index1 < index2 <= numbers.length 。

以长度为 2 的整数数组 [index1, index2] 的形式返回这两个整数的下标 index1 和 index2。

```
输入：numbers = [2,7,11,15], target = 9
输出：[1,2]
解释：2 与 7 之和等于目标数 9 。因此 index1 = 1, index2 = 2 。返回 [1, 2] 。
```

ps:

* 两个数从两边逼近
* 如果小了，左边就向右移位left++
* 如果大了，右边就向左移位right--
* 找到和为target的值就返回下标



#### [5. 最长回文子串](https://leetcode.cn/problems/longest-palindromic-substring/)

给你一个字符串 `s`，找到 `s` 中最长的回文子串。

如果字符串的反序与原始字符串相同，则该字符串称为回文字符串。

```
输入：s = "babad"
输出："bab"
解释："aba" 同样是符合题意的答案。
```

ps:

* 从中心向两端扩散的双指针技巧
* 找到奇数或偶数的回文字符串，遍历整个字符串，找到最长的字符串