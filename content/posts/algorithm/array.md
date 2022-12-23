---
title: "算法与数据结构 -- 线性表（2）-- 顺序表"
date: 2022-12-22T19:57:00+08:00
categories: [
    "Algorithm"
]
tags: [
    'programming',
    'algorithm',
    'data-structure',
]
series: ["408", "Algorithm"]
---

上一节我们介绍了线性表的概念，这一节阐述线性表的顺序存储实现。

## 顺序表

顺序表最大的特点就是：元素在内存中存储的位置是连续的，因此只要已知首地址就可以通过首地址 + 偏移量访问到顺序表中的元素。

但也因为这个特点，顺序表的插入和删除操作会比较麻烦，因为插入和删除操作会导致后面所有的元素需要移动，而移动元素的时间复杂度是 O(n)。

由于高级语言中的数组就是顺序表的实现，因此顺序表的实现通常都非常简单。

一个顺序表 Header 可定义结构如下：

```go
type SeqList struct {
    data []interface{}
    length int
}
```

其中 `data` 是一个数组，用来存储顺序表中的元素，`length` 是顺序表中元素的个数。

### 初始化

初始化顺序表的方法如下：

```go
func NewSeqList() *SeqList {
    return &SeqList{
        data: make([]interface{}, 0),
        length: 0,
    }
}
```

Go 的 slice 是可以动态扩展的，因此我们初始化顺序表的时候只需要初始化一个空的 slice 即可。

### 插入

插入操作的方法如下：

```go
func (l *SeqList) Insert(index int, value interface{}) error {
    if index < 0 {
        return errors.New("index out of range")
    }

    if l.length <= cap(l.data) {
        l.data = append(l.data, value)
    }
    for i := l.length; i > index; i-- {
        l.data[i] = l.data[i-1]
    }
    l.data[index] = value
    l.length++
    return nil
}
```

插入操作的时间复杂度是 O(n)，因为插入操作会导致后面所有的元素需要移动。

### 删除

删除操作的方法如下：

```go
func (l *SeqList) Delete(index int) (interface{}, error) {
    if index < 0 {
        return nil, errors.New("index out of range")
    }

    value := l.data[index]
    for i := index + 1; i < l.length; i++ {
        l.data[i - 1] = l.data[i]
    }
    l.length--
    return value, nil
}
```

取操作是非常显然的，就不再赘述了。

## 一些练习

每个章节都会挑一些习题，同时给出一些思路。不会找很难的习题，只是为了让大家对这一章的内容有一个更深入的理解。有时候会给一些类型题的解题思路。

### LeetCode 27 移除元素

[中文站链接](https://leetcode-cn.com/problems/remove-element/)

这道题和他的姊妹题 [26. 删除排序数组中的重复项](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-array/) 非常类似，只不过这道题是删除指定的元素，而不是重复的元素。这道题会更简单一些。

它唯一的难点在于要求 O(1) 的空间复杂度，也就是不能使用额外的数组来存储结果。

最暴力的解法是遇到 `val` 就删除，虽然它的时间复杂度是 $O(n^2)$，但测试是可以通过的。

```go
func removeElement(nums []int, val int) int {
    size := len(nums)
    for i := 0; i < size; i++ {
        if nums[i] == val {
            for j := i + 1; j < size; j++ {
                nums[j - 1] = nums[j]
            }
            i--
            size--
        }
    }
    return size
}
```

由于每次删除都要前移，优化的方法是使用所谓 “双指针” 的方法.

所谓双指针，其实不一定真的是一个物理意义上的指针，实际上这里的指针是一个抽象概念，它使用两个变量来存储当前遍历位置的数组下标，也算是一种 "指针“。

双指针指我们定义一个快指针，和一个慢指针，快指针用来遍历数组，慢指针用来记录当前不是 `val` 的元素的位置，当快指针遇到不是 `val` 的元素时，就将它赋值给慢指针对应的元素，然后慢指针向前移动一位。

一个更通用的定义是：

- 快指针用来寻找新数组的元素，即不含有目标元素的数组
- 慢指针指向更新新数组下标的位置

这样一来，保证了 nums[0..slow) 中的元素都不重复，当 fast 遍历完整个数组时，slow 指向的位置就是新数组的长度。

数组中的元素是无法删除的，只能被覆盖。所以我们可以使用快慢指针来覆盖掉所有的 `val` 元素，最后返回慢指针的位置即可。

```go
func removeElement(nums []int, val int) int {
    size := len(nums)
    slow := 0
    for fast := 0; fast < size; fast++ {
        if nums[i] != val {
            nums[slow] = nums[fast]
            slow++
        }
    }
    return slow
}
```

非常类似的相同解法的题目还有 [283. 移动零](https://leetcode-cn.com/problems/move-zeroes/) 和 [26. 删除排序数组中的重复项](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-array/)。

### LeetCode 9 回文数

[LeetCode 9 回文数](https://leetcode-cn.com/problems/palindrome-number/)

这道题的解法非常简单，就是将数字转换成字符串，然后判断字符串是否是回文串即可。

又是一道双指针，双指针在数组题目当中是很常见的。

```go
func isPalindrome(x int) bool {
    s := []rune(strconv.Itoa(x))
    left := 0
    right := len(s) - 1
    for left <= right {
        if s[left] != s[right] {
            return false
        }
        left++
        right--
    }
    return true
}
```