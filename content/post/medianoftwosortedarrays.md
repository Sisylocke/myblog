---
title: "Median Of Two Sorted Arrays"
date: 2020-03-23T22:41:55+08:00
draft: false
categories: "刷了个题"
---

## 复习 -- Binary Search
```
# 参考python源码bisect.py, 返回值为第一个不小于val的值的索引.
def lower_bound(array, val):
    low = 0
    high = len(array)
    while low < high:
        mid = low + (high - low) // 2
        if array[mid] < val: low = mid + 1
        else: high = mid
    return low
```

Notes:
  
1. 算法搜索区间为`[low, high)`, 为了保证这个搜索区间:
    - `high`的初始值为`len(array)`, 而不是`len(array)-1`
    - 循环的判断使用`<`, 而不是`<=`. 若使用`while(low <= high)`将搜索空间扩展为`[low, high]`, 最后会导致无限循环.
    - 更新`low`使用`mid + 1`, 而不是`mid`
2. 不能使用`array[mid] == val`来判断是否终止循环. 当`array`存在重复元素, 这时返回的值可能不是第一个不小于`val`的索引.但如果只是为了找到任意一个满足条件的索引则可以这样做.
3. 以上代码使用与非降序排列的数组. 如果不是升序, 则将搜索区间设置为`(low, high]`. 然后 `mid = hgih - (high - low) // 2`总之要从闭区间侧更新`mid`
4. 要求upper_bound咋办? 将`x < val` 改为 `x <= val`就行了.

---
## Leetcode -- median of two sorted arrays
#### 问题描述:
给定两个大小为 m 和 n 的有序数组 nums1 和 nums2。
请你找出这两个有序数组的中位数，并且要求算法的时间复杂度为 O(log(m + n))。
你可以假设 nums1 和 nums2 不会同时为空。

#### 思路
先来看中位数的定义:
>将一个集合划分为两个长度相等的子集，其中一个子集中的元素总是大于另一个子集中的元素。

这题用暴力解决的话, 因为有多个数组合并, 涉及到重新排序, 时间和空间复杂度都会是`O(m+n)`, 不满足要求, 所以这种思想就要不得了.~~不过用`python3`的`sort`来做这题也能提交通过的~~

由于题目要求的时间复杂度为`O(log(n))`, 而且都是有序数组, 自然会想到`binary search`.

因为不需要返回索引, 那么就可以不用真的合并两个数组. 将这两个数组`A和B`分别分割为两部分`left`和`right`, 使得<img src="https://tex.s2cms.ru/svg/len(A_%7Bleft%7D)%20%2B%20len(B_%7Bleft%7D)%20%3D%20len(A_%7Bright%7D)%20%2B%20len(B_%7Bright%7D)" alt="len(A_{left}) + len(B_{left}) = len(A_{right}) + len(B_{right})" />. 按照中位数定义, 只需要找到这样一个位置`i`, 使得`i`对应的值大于所有`left`中的值同时小于所有`right`中的值.

<img src="https://tex.s2cms.ru/svg/%0A%5Cbegin%7Bcenter%7D%0A%5Cbegin%7Btabular%7D%7B%7Cc%7Cc%7Cc%7C%7Cc%7Cc%7Cc%7D%0A%5Chline%20x_1%26x_2%26x_3%26x_4%26x_5%20%5C%5C%0A%5Chline%0A%5Cend%7Btabular%7D%0A%5Cend%7Bcenter%7D%0A" alt="
\begin{center}
\begin{tabular}{|c|c|c||c|c|c}
\hline x_1&amp;x_2&amp;x_3&amp;x_4&amp;x_5 \\
\hline
\end{tabular}
\end{center}
" />

<img src="https://tex.s2cms.ru/svg/%0A%5Cbegin%7Bcenter%7D%0A%5Cbegin%7Btabular%7D%7B%7Cc%7Cc%7C%7Cc%7Cc%7Cc%7Cc%7D%0A%5Chline%20y_1%26y_2%26y_3%26y_4%26y_5%20%5C%5C%0A%5Chline%0A%5Cend%7Btabular%7D%0A%5Cend%7Bcenter%7D%0A" alt="
\begin{center}
\begin{tabular}{|c|c||c|c|c|c}
\hline y_1&amp;y_2&amp;y_3&amp;y_4&amp;y_5 \\
\hline
\end{tabular}
\end{center}
" />

例如在上面两个数组中, left部分油<img src="https://tex.s2cms.ru/svg/%5Bx_1%2C%20x_2%2C%20x_3%5D" alt="[x_1, x_2, x_3]" />和<img src="https://tex.s2cms.ru/svg/%5By_1%2C%20y_2%5D" alt="[y_1, y_2]" />构成, right部分为<img src="https://tex.s2cms.ru/svg/%5Bx_4%2C%20x_5%5D" alt="[x_4, x_5]" />和<img src="https://tex.s2cms.ru/svg/%5By_3%2C%20y_4%2C%20y_5%5D" alt="[y_3, y_4, y_5]" />构成, 此时`len(left) == len(right) == 5`, . 现在我们的要做两个判断(x, y为有序数组, 显然<img src="https://tex.s2cms.ru/svg/x_3%20%3C%3D%20x_4%2C%20y_2%20%3C%3D%20y_3" alt="x_3 &lt;= x_4, y_2 &lt;= y_3" />, 不用判断):

<img src="https://tex.s2cms.ru/svg/%0A%5C%5B%0Ax_3%20%3C%3D%20y_3%20%5C%5C%0Ay_2%20%3C%3D%20x_4%0A%5C%5D%0A" alt="
\[
x_3 &lt;= y_3 \\
y_2 &lt;= x_4
\]
" />
假如以上判断成立, 根据中位数的定义, 中位数由<img src="https://tex.s2cms.ru/svg/%5Bx_3%2C%20x_4%2C%20y_2%2C%20y_3%5D" alt="[x_3, x_4, y_2, y_3]" />决定, 因为这个例子总长度为偶数, 所以<img src="https://tex.s2cms.ru/svg/%20median%20%3D%20avg(max(x_3%2C%20y_2)%2C%20min(x_4%2C%20y_3))%20" alt=" median = avg(max(x_3, y_2), min(x_4, y_3)) " />. 假如两个数组总长度为奇数的话, <img src="https://tex.s2cms.ru/svg/%20median%20%3D%20max(x_3%2C%20y_2)%20" alt=" median = max(x_3, y_2) " />.

为了找到这样一个分割, 可以对数组 <img src="https://tex.s2cms.ru/svg/x" alt="x" /> 进行二分查找.代码如下:

```
class Solution:
    def findMedianSortedArrays(self, nums1: List[int], nums2: List[int]) -> float:
        
        #对较短的数组进行二分查找可以加快速度
        if len(nums1) > len(nums2):
            nums1, nums2 = nums2, nums1

        n = len(nums1)
        m = len(nums2)
        half = (m + n + 1) // 2

        #使用的是二分查找模板
        left = 0
        right = n
        while left < right:
            mid = (left + right) // 2
            co_mid = half - mid

            if nums2[co_mid - 1] > nums1[mid]:
                left = mid + 1
            else:
                right = mid

        # while循环结束可以保证max(left) < min(right),但是存在边界情况.
        mid = left
        co_mid = half - mid
        
        # 这里直接将越界(out of index)的值设置为正或负无穷.
        nums1_left_max = float('-inf') if mid == 0 else nums1[mid-1]
        nums1_right_min = float('inf') if mid == n else nums1[mid]

        nums2_left_max = float('-inf') if co_mid == 0 else nums2[co_mid - 1]
        nums2_right_min = float('inf') if co_mid == m else nums2[co_mid]

        if (m + n) % 2:
            return max(nums1_left_max, nums2_left_max)
        else:
            return sum([max(nums1_left_max, nums2_left_max), min(nums1_right_min, nums2_right_min)]) / 2

```
