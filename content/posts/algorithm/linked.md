---
title: "算法与数据结构 -- 线性表（3）-- 链表"
date: 2022-12-22T22:37:00+08:00
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

上一节我们介绍了线性表的顺序存储实现，这一节阐述线性表的链式存储实现。

<!--more-->

## 链表

链表由一系列不必在内存中物理相连的结构组成，每个结构都包含一个数据元素和一个指向下一个结构的指针。

根据指针的指向和数量，链表通常又可分为单链表、双向链表和循环链表。

实际使用中，双向链表更为常用。

若我们直接对第一个结构进行操作，那么编程会出现一些麻烦的问题，例如：

- 如何在首结构前插入？
- 如何删除首结构？

要么这些操作很难完成，要么需要在函数中特殊对待，这为编程提供了挑战。事实上只要在首结构前加入一个 Header 就可以很好的解决了。

许多时候我们约定 Header 作为位置 0，虽然我个人不喜欢这种做法。

## 单链表

单链表的节点结构可以表示为：

```go
type Node struct {
    Data interface{}
    Next *Node
}
```

显然的，链表的访问方式与顺序表非常不同，因此代码也有很大变化。

### 初始化

```go
func New() *Node {
    return &Node{}
}
```

### 取值

```go
func (n *Node) Get(i int) (interface{}, error) {
    if i < 0 {
        return nil, errors.New("index out of range")
    }
    for j := 0; j < i; j++ {
        n = n.Next
        if n == nil {
            return nil, errors.New("index out of range")
        }
    }
    return n.Data, nil
}
```

### 插入

```go
func (n *Node) Insert(i int, data interface{}) error {
    if i < 0 {
        return errors.New("index out of range")
    }
    for j := 0; j < i; j++ {
        n = n.Next
        if n == nil {
            return errors.New("index out of range")
        }
    }
    n.Next = &Node{Data: data, Next: n.Next}
    return nil
}
```

### 删除

```go
func (n *Node) Delete(i int) error {
    if i < 0 {
        return errors.New("index out of range")
    }
    for j := 0; j < i; j++ {
        n = n.Next
        if n == nil {
            return errors.New("index out of range")
        }
    }
    n.Next = n.Next.Next
    return nil
}
```

### 创建链表

创建链表根据节点插入位置的不同，可以分为头插法和尾插法。

头插法：

```go
func CreateHead(data []interface{}) *Node {
    var head *Node = nil
    for _, v := range data {
        head := &Node{Data: v, Next: head}
    }
    return head
}
```

尾插法：

```go
func CreateTail(data []interface{}) *Node {
    var head *Node = nil
    var tail *Node = nil
    for _, v := range data {
        if head == nil {
            head = &Node{Data: v, Next: nil}
            tail = head
        } else {
            tail.Next = &Node{Data: v, Next: nil}
            tail = tail.Next
        }
    }
    return head
}
```

## 双向链表

双向链表的节点结构可以表示为：

```go
type Node struct {
    Data interface{}
    Prev *Node
    Next *Node
}
```

即每个节点多维护一个指向前一个节点的指针。

每次插入或删除节点时，需要同时修改前后节点的指针。

## 循环链表

循环链表即为首尾相连的链表，即首节点的前驱为尾节点，尾节点的后继为首节点。

与单链表相比，最大的区别在于遍历的边界条件不同。

在循环链表中，以首节点为起点，遍历到尾节点时，尾节点的后继为首节点，因此遍历的边界条件为：

```go
for n.Next != head {
    // ...
}
```

## 一些练习

### LeetCode 707 设计链表

[LeetCode 707 设计链表](https://leetcode-cn.com/problems/design-linked-list/)

一道链表设计题，基本涵盖了链表的常用操作。

不要小看这道题，好好写完这道链表题你会发现实现链表过程中有很多边界条件是不好掌握的。

```go
type MyLinkedList struct {
    Val int
    Next *MyLinkedList
}


func Constructor() MyLinkedList {
    // return a dummy node
    return MyLinkedList{
        Val: -1,
        Next: nil,
    }
}


func (this *MyLinkedList) Get(index int) int {
    cur := this.Next
    for i := 0; cur != nil; i++ {
        if i == index {
            return cur.Val
        } else {
            cur = cur.Next
        }
    }
    return -1
}


func (this *MyLinkedList) AddAtHead(val int)  {
    new := &MyLinkedList{
        Val: val,
        Next: this.Next,
    }
    this.Next = new
    return
}


func (this *MyLinkedList) AddAtTail(val int)  {
    cur := this
    for cur.Next != nil {
        cur = cur.Next
    }
    new := &MyLinkedList{
        Val: val,
        Next: nil,
    }
    cur.Next = new
}


func (this *MyLinkedList) AddAtIndex(index int, val int)  {
    if index < 0 {
        this.AddAtHead(val)
        return
    }
    if index == 0 {
        this.AddAtHead(val)
        return
    }
    cur := this
    for i := 0; i < index; i++ {
        if cur.Next != nil {
            cur = cur.Next
        } else {
            if i == index {
                this.AddAtTail(val)
                return
            } else {
                return
            }
        }
    }
    new := &MyLinkedList{
        Val: val,
        Next: cur.Next,
    }
    cur.Next = new
}


func (this *MyLinkedList) DeleteAtIndex(index int)  {
    cur := this
    for i := 0; i < index; i++ {
        if cur.Next == nil {
            return
        }
        cur = cur.Next
    }
    if cur.Next != nil {
        cur.Next = cur.Next.Next
    }
    return
}


/**
 * Your MyLinkedList object will be instantiated and called as such:
 * obj := Constructor();
 * param_1 := obj.Get(index);
 * obj.AddAtHead(val);
 * obj.AddAtTail(val);
 * obj.AddAtIndex(index,val);
 * obj.DeleteAtIndex(index);
 */
```

这是一个普通的单链表实现，把它改成双向链表也不是很困难，留给各位读者作为思考题。

### LeetCode 21 合并两个有序链表

[LeetCode 21 合并两个有序链表](https://leetcode-cn.com/problems/merge-two-sorted-lists/)

这是一道基础题目，也是教材上的一道例题。

```go
func mergeTwoLists(list1 *ListNode, list2 *ListNode) *ListNode {
	dummy := &ListNode{}
	cur := dummy
    l1 := list1
    l2 := list2
	for l1 != nil && l2 != nil {
		// 如果 list1 小，把 list1 节点接过来，反之依然
		if l1.Val < l2.Val {
			cur.Next = l1
			l1 = l1.Next
		} else {
			cur.Next = l2
			l2 = l2.Next
		}
		// cur 后移
		cur = cur.Next
	}
	// 如果 list1 和 list2 还有剩余，把剩余的节点接到后面
	if l1 != nil {
		cur.Next = l1
	}
	if l2 != nil {
		cur.Next = l2
	}

	return dummy.Next
}
```

### LeetCode 203 移除链表元素

[LeetCode 203 移除链表元素](https://leetcode-cn.com/problems/remove-linked-list-elements/)

同样是基础题目，遍历一遍遇到就删就可以了。

```go
func removeElements(head *ListNode, val int) *ListNode {
    dummy := &ListNode{
        Val: -1,
        Next: head,
    }
    cur := dummy
    for cur.Next != nil {
        if cur.Next.Val == val {
            cur.Next = cur.Next.Next
        } else {
            cur = cur.Next
        }
    }
    return dummy.Next
}
```

### LeetCode 19 删除链表的倒数第 N 个结点

[LeetCode 19 删除链表的倒数第 N 个结点](https://leetcode-cn.com/problems/remove-nth-node-from-end-of-list/)

如果能够事先知道链表的长度，这道题非常简单，本质上变成小学数学题了。但通常拿到一个链表只能知道链表的头指针，而不知道链表的长度。

这道题又是一个双指针。实现非常巧妙，我们使用两个指针，第一个指针从链表的头指针开始遍历向前走 n 步，第二个指针保持不动；从第 $n+1$ 步开始，第二个指针也开始从链表的头指针开始遍历。由于两个指针的距离保持在 n 步，当第一个指针到达链表的尾结点时，第二个指针正好是倒数第 n 个结点。

```go
func removeNthFromEnd(head *ListNode, n int) *ListNode {
    dummy := &ListNode{
        Val: -1,
        Next: head,
    }
    fast := dummy
    slow := dummy
    for i := 0; i < n; i++ {
        fast = fast.Next
    }

    for fast.Next != nil {
        slow = slow.Next
        fast = fast.Next
    }
    slow.Next = slow.Next.Next
    return dummy.Next
}
```

### LeetCode 83 删除排序链表中的重复元素

[LeetCode 83 删除排序链表中的重复元素](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-list/)

又是简单题目。最简单的解法是遍历链表，如果当前结点的值和下一个结点的值相同，就删除下一个结点。

由于这道题要至少保留一个节点，不会出现删除头节点的情况，所以不需要使用虚拟头结点。

```go
func deleteDuplicates(head *ListNode) *ListNode {
    cur := head
    for cur != nil && cur.Next != nil {
        if cur.Val == cur.Next.Val {
            cur.Next = cur.Next.Next
        } else {
            cur = cur.Next
        }
    }
    return head
}
```
