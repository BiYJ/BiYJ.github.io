---
title: 第 k 小的数
categories: algorithm
---

## 一、[寻找两个有序数组的中位数](https://leetcode-cn.com/problems/median-of-two-sorted-arrays/)

#### 1、问题描述

给定两个大小为 m 和 n 的不同时为空的有序数组 nums1 和 nums2。找出这两个有序数组的中位数，并且要求算法的时间复杂度为 O(log(m + n))。

#### 2、算法分析

题目要求的时间复杂度是 O(log(m + n))，要产生这样级别的时间复杂度只有采用二分查找法，用分治递归的思路来考虑这个问题。

需要转换题目中求中位数的问题为求第 k 小数的问题。如果 m + n 是奇数，那么寻找第 <font color=#cc0000>k = (m + n)/2 + 1</font> 小的数即可；如果长度和是偶数，那么我们还需要寻找第 (m + n)/2 小的数，然后计算两数的平均值。

在求解整个问题的过程中，我们始终需要考虑一个很重要的问题--<font color=#cc0000>数组索引越界问题</font>。

下面将详细地分析整个递归流程。

①、首先定义递归函数的作用：寻找两个有序数组 nums1 数组中 \[L1, R1\] 范围内和 nums2 数组 \[L2, R2\] 范围内第 k 小的数，k 从 1开始计数。

```c
/**
 *  L1   nums1数组的寻找范围的左边界
 *  R1   nums1数组的寻找范围的右边界
 *  L2   nums2数组的寻找范围的左边界
 *  R2   nums2数组的寻找范围的右边界
 *  k    需要寻找第k小的元素
 */
int findKth(int[] nums1, int L1, int R1, int[] nums2, int L2, int R2, int k)
```

②、用 len1 = R1 - L1 + 1 来记录 nums1 数组中寻找范围的长度，用 len2 = R2 - L2 + 1 来记录 nums2 数组中寻找范围的长度。

③、如果要寻找的 k > len1 + len2，就像只有 3 个数字要找第 4 小的数一样，超出寻找区域，显然无法找到。

④、递归的终止条件：

1. 当 len1 = 0 时，说明只有 nums2 数组中有元素，直接取 nums2\[L2 + k - 1\] 位元素即可。

2. 当 k = 1 时，说明要取的是两个有序数组中的最小值 MIN(nums1\[L1\], nums2\[L2\])。

⑤、递归过程：

由于要求的是第 k 小的数，而且是在两个有序数组中求。划分两个数组时按照 k 值来分。取变量 i = MIN(len1, k/2)，之所以这么取，是为了防止 L1 + k/2 - 1 > len1 导致从 nums1 取值越界。再取变量 j = MIN(len2, k/2)。

接下来比较 nums1\[L1 + i - 1\] 和 nums2\[L2 + i - 1\] 这两个值。

如果 nums1\[L1 + i - 1\] <= nums2\[L2 + j - 1\]，<font color=#cc0000>显然 nums1 数组中索引为 L1 + i - 1 及之前的元素不可能是中位数</font>，去除 nums1 数组中 \[L1, L1 + i - 1\] 范围内的元素，缩小了查找范围。我们递归调用该函数，此时在 nums1 中的查找范围变成了 nums1\[L1 + i, R1\]，此时要找的也不应该是第 k 小的元素，因为<font color=#cc0000>已经剔除了 i 个比 k 小的元素</font>，因此我们要找的元素变成了第 k - i 小的元素。

如果 nums1\[L1 + i - 1\] > nums2\[L2 + j - 1\]，同理，nums2 数组中索引为 L2 + j - 1 及之前的元素不可能是中位数，缩小查找范围，剔除了 j 个比 k 小的元素，因此我们要找的元素变成了第 k - j 小的元素。

> 因为 i + j = MIN(len1, k/2) + MIN(len2, k/2) <= k，所以可以直接判断 \[L1, L1 + i - 1\] 或者 \[L2, L2 + j -1\] 区间的元素不可能是中位数。 

<center>
![](https://upload-images.jianshu.io/upload_images/5294842-19b4f9b39cfc6c28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</center>

总结：<font color=#cc0000>算法的思想是不断的剔除数据，逐渐逼近第 k 小的数</font>。

## 3、时间复杂度

假设数组长度足够长，每次剔除的元素都是 k/2(i 或者j)，显然我们需要 log(k) 次才能找到第 k 小数，这和二分查找法是同理的，而我们要找的 k 值要么是 (m + n)/2 + 1，要么额外再加上 (m + n)/2，因此时间复杂度是 O(log(m + n)) 级别的。

## 4、代码实现

```c
#define MIN(a, b) (a) < (b) ? (a) : (b)

int findKth(int* nums1, int left1, int right1, int* nums2, int left2, int right2, int k)
{
    int n1 = right1 - left1 + 1;
    int n2 = right2 - left2 + 1;
    
    // 递归退出条件
    if(k > n1 + n2) {
        return 0;  // 实际上 k 不会小于 n1 + n2
    }

    if(n1 == 0) {
        return nums2[left2 + k - 1];
    }
    else if (n2 == 0) {
        return nums1[left1 + k - 1];
    }
    if(k == 1) {
        return MIN(nums1[left1], nums2[left2]);
    }
    int i = MIN(n1, k / 2);
    int j = MIN(n2, k / 2);
    
    // 剔除比第 k 小的数还小的数，逐渐逼近
    if(nums1[left1 + i - 1] > nums2[left2 + j - 1]) {
        return findKth(nums1, left1, right1, nums2, left2 + j, right2, k - j);
    }
    else {
        return findKth(nums1, left1 + i, right1, nums2, left2, right2, k - i);
    }
}

double findMedianSortedArrays(int* nums1, int nums1Size, int* nums2, int nums2Size)
{
    // k = (nums1Size + nums2Size) /2 + 1，因为 k 从 1 开始计数
    int mid1 = findKth(nums1, 0, nums1Size - 1, nums2, 0, nums2Size - 1, (nums1Size + nums2Size) / 2 + 1);
    
    // 两个数组总长度是奇数
    if((nums1Size + nums2Size) % 2 != 0) {
        return mid1;
    }
    // 两个数组总长度是偶数
    else {
        // 额外求 (nums1Size + nums2Size) / 2 的值
        int mid2 = findKth(nums1, 0, nums1Size - 1, nums2, 0, nums2Size - 1, (nums1Size + nums2Size) / 2);
        return (mid1 + mid2) / 2.0;
    }
}
```

## 二、文章

[LeetCode004——两个排序数组的中位数](https://blog.csdn.net/qq_41231926/article/details/81805795) 
