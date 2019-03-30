---
title: 数据结构与算法之排序
date: 2019-03-23 16:57:20
tags:
    - 排序
categories:
    - 数据结构与算法
---

分析排序算法的几个要点：

1. 排序算法的执行效率，比如最好、最好、平均时间复杂度，移动次数等

2. 排序算法的内存消耗，也就是它们的空间复杂度

3. 排序算法的稳定性，如果待排序的序列中存在值相等的元素，经过排序之后，相等元素原有的先后顺序不变则稳定

![](http://picture.wzmmmmj.com/sort.png)

> 附一张排序图，其中 n=数据量     k=桶的个数     In-place=没有占用额外内存，out-place=占用额外内存
<!-- more -->

### 冒泡排序

冒泡排序是最简单的排序算法之一，也是面试中经常遇到的题。其大题思想通过相邻两位比较和交换，将小（大）的数交换到最前面（后面），类似气泡上升一样。

1. 通过比较相邻元素，保证右侧元素大于左侧，使每次遍历的最大值在序列的最后项；
2. 对剩下的n-1项重复执行步骤1。

```python
def bubble_sort(nums):
    if len(nums) <= 1:
        return nums
    for i in range(1, len(nums)):
        for j in range(len(nums) - i):
            if nums[j] > nums[j + 1]:
                nums[j], nums[j + 1] = nums[j + 1], nums[j]
    return nums
```

### 选择排序

选择排序，顾名思义，就是每次遍历选择序列中最小值，放置在此次遍历的首位。与冒泡排序不同的是，选择排序是通过对整体的比较。因为每次遍历都会遍历全部元素，所以选择排序的最好，最坏时间复杂度都是 O(n^2^)

1. 将序列首位元素设置为标志位；
2. 遍历序列，与标志位比较，选出最小值，与首位元素交换；
3. 对剩下的项重复执行1，2步骤。

```python
def select_sort(nums):
    for i in range(len(nums)):
        min_index = i
        for j in range(i, len(nums)):
            if nums[min_index] > nums[j]:
                min_index = j
        nums[i], nums[min_index] = nums[min_index], nums[i]
    return nums
```

### 插入排序

插入排序是O(n^2^)的3种排序中最为稳定的排序方法，排序思想类似于选择排序，不过插入排序 是将每次的值，与前值比较，直到比某个值n小，插入到n后，其余元素后移一位。与选择排序相比，插入排序可以减少交换次数。

> 类似打扑克时，理牌的过程

元素m与前值n比较，

1. 如果m>n，则进行下次循环
2. 如果m<n，交换位置，进行下一次比较

```python
def insert_sort(cls, nums):
    for i in range(1, len(nums)):
        for j in range(i, 0, -1):
            if nums[j] < nums[j - 1]:
                nums[j - 1], nums[j] = nums[j], nums[j - 1]
            else:
                break
    return nums
```

### 希尔排序

希尔排序是插入排序的一个变种，可以更高效的完成排序。希尔排序的关键在于间隔序列的设定，以确定增量。

1. 选择间隔，确定增量（固定间隔或动态定义间隔）；
2. 根据间隔，对序列进行分组，对每组元素进行插入排序；
3. 对新序列重复执行1，2步骤，直到完成排序。

```python
def shell_sort(nums):
    length, gap = len(nums), 1
    while gap < length:  # 动态生成间隔值
        gap = gap * 3 + 1
    while gap > 0:
        for i in range(gap, length:  # 执行
            for j in range(i, 0, -gap):
                if nums[j] < nums[j - gap]:
                    nums[j - gap], nums[j] = nums[j], nums[j - gap]
                else:
                    break
        gap //= 3
   return nums
```

### 快速排序

快速排序，俗称快排，最实用的排序方法，手写快排应该是出场率最高的面试题之一了。

1. 找到一个基准值
2. 根据与基准值比较大小，将序列分为左右两个两个子序列
3. 对子序列重复1，2步骤，直到子序列只有一个元素

**快速排序的好坏，关键在于基准值**

> 基准值的选择可以使用三数取中法即（每次排序随机取三个数的中间值作为分区点）或随机法（随机取值）。

```python
def _qsort(nums, start, end):
    base, left, right = nums[start], start, end
    while left < right:
        while left < right and nums[right] >= base:
            right -= 1
        if left != right:
            nums[left], nums[right] = nums[right], nums[left]
        else:
            break
        while left < right and nums[left] <= base:
            left += 1
        if left != right:
            nums[left], nums[right] = nums[right], nums[left]
        else:
            break
    if left - 1 > start:
        _qsort(nums, start, left -1)
    if right + 1 < end:
        _qsort(nums, right + 1, end)
    return nums

def quick_sort(nums):
    if len(nums) <= 1:
        return nums
    return _qsort(nums, 0, len(nums))
```

### 归并排序

归并排序比快速排序更加稳定，不受数据影响，不过需要空间复杂度**O(n)**作为代价。

1. 将序列分为两个子序列
2. 重复1步骤，直到每个序列至多两个元素
3. 递归序列内排序，直到排序完成

```python
def _merge(left, right):
    a, b, ans = 0, 0, []
    while a < len(left) and b < len(right):
        if left[a] < right[b]:
            ans.append(left[a])
            a += 1
        else:
            ans.append(right[b])
            b += 1
    ans += left[a:] or right[b:]
    return ans

def merge_sort(nums):
    if len(nums) <= 1:
        return nums
    left = merge_sort(nums[:len(nums) // 2])
    right = merge_sort(nums[len(nums) // 2:])
    return _merge(left, right)
```

## 总结

1. 不能一味的根据时间复杂度，来选择某种排序方法，而是要根据实际情况。例如当序列元素个数少时，插入排序的效率就要大于快排。如果数据量过大，内存不充分的情况下，也不宜使用需要额外空间的归并排序。

2. 不同语言在实现sort函数时，使用的排序方法不同。不过都有根据元素的个数来改变使用的排序方法
