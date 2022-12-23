---
title: "算法与数据结构 -- 栈和队列（2）-- 队列"
date: 2022-12-24T00:40:00+08:00
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

和栈类似，队列 (Queue) 也是一种线性表，只不过它的操作方式与栈不同。队列在尾部插入元素，在头部删除元素。

<!--more-->

队列的基本操作称为入队 (Enqueue) 和出队 (Dequeue)。 它在表的末端 (称为队尾 rear) 进行插入操作，而在表的前端 (称为队头 front) 进行删除操作。

队列是先进先出的，与栈不同，由于这个特性，队列又被称作先进先出表 (First In First Out, FIFO)。

和栈类似，队列使用链表和数组实现都是合法的，链表实现队列是很显然的，只要各维护好队头和队尾指针即可。无论是入队还是出队，时间复杂度都是 O(1)。

队列的数组实现稍微麻烦，但也不复杂，以下内容讨论的都是数组实现的队列。

## 队列的数组实现

队列的数组实现严格来说需要维护两个指针，分别指向队头和队尾。队头指针指向队头元素，队尾指针指向队尾元素的下一个位置。这样，队列的大小就是队尾指针减去队头指针。

但是由于 Go Slice 的特性，我们可以不用维护队头指针，只需要维护队尾指针即可。

这是最简单的队列实现，很容易写出代码:

```go
type Queue struct {
    data []interface{}
    tail int
}

func NewQueue() *Queue {
    return &Queue{
        data: make([]interface{}, 0),
        tail: 0,
    }
}

func (q *Queue) Enqueue(v interface{})  {
    q.data = append(q.data, v)
    q.tail++
    return
}

func (q *Queue) Dequeue() (interface{}, error) {
    if q.tail == 0 {
        return errors.New("queue is empty")
    }
    v := q.data[0]
    q.data = q.data[1:]
    return v, nil
}
```

但是对于像 C 那种没那么灵活的语言显然这样是不行的，Go Slice 简化了很多东西，这样实现队列就没有任何难点了，所以还是给出 C 的实现。

## 队列的数组实现（C）

C 语言中，数组是固定大小的，没有办法像 Go Slice 那样灵活，因此需要更原教旨一点的实现方式。

```c
typedef struct {
    int *data;
    int head;
    int tail;
    int size;
} Queue;

Queue* queueCreate(int maxSize) {
    Queue *q = (Queue *)malloc(sizeof(Queue));
    q->data = (int *)malloc(sizeof(int) * maxSize);
    q->head = 0;
    q->tail = 0;
    q->size = maxSize;
    return q;
}

bool queueEnQueue(Queue* obj, int value) {
    if (queueIsFull(obj)) {
        return false;
    }
    obj->data[obj->tail] = value;
    obj->tail = obj->tail + 1;
    return true;
}

bool queueDeQueue(Queue* obj) {
    if (queueIsEmpty(obj)) {
        return false;
    }
    obj->head = obj->head + 1;
    return true;
}
```

但这样一来有一个潜在的问题，当几次入队出队操作之后，队列看似是满了，但实际上队列的头部还有许多空闲空间，这样就造成了空间的浪费。

一个解决方法是，只要头指针和尾指针到达数组末尾，就重新绕回开头。这种实现称为循环队列。

循环队列使得队列判空和判满操作变得复杂，需要额外的判断。

一种常用的方法是，以一个单位空间为代价，即少用一个元素的空间，这样就可以区分队列满和队列空的情况。

当头尾指针相同时，认为队空，而当尾指针的下一个位置是头指针时，认为队满。

这样一来就有了下面的完整实现：

```c
typedef struct {
    int *data;
    int head;
    int tail;
    int size;
} Queue;

Queue* queueCreate(int maxSize) {
    Queue *q = (Queue *)malloc(sizeof(Queue));
    q->data = (int *)malloc(sizeof(int) * maxSize);
    q->head = 0;
    q->tail = 0;
    q->size = maxSize;
    return q;
}

bool queueIsEmpty(Queue* obj) {
    return obj->head == obj->tail;
}

bool queueIsFull(Queue* obj) {
    return (obj->tail + 1) % obj->size == obj->head;
}

bool queueEnQueue(Queue* obj, int value) {
    if (queueIsFull(obj)) {
        return false;
    }
    obj->data[obj->tail] = value;
    obj->tail = (obj->tail + 1) % obj->size;
    return true;
}

bool queueDeQueue(Queue* obj) {
    if (queueIsEmpty(obj)) {
        return false;
    }
    obj->head = (obj->head + 1) % obj->size;
    return true;
}

int queueFront(Queue* obj) {
    if (queueIsEmpty(obj)) {
        return -1;
    }
    return obj->data[obj->head];
}

void queueFree(Queue* obj) {
    free(obj->data);
    free(obj);
}
```

## 一些练习

意外的，队列在算法题中的考察并不多。至少 LeetCode 上很少有纯粹使用队列的题目。

但是，队列的应用场景很广，比如 BFS、LRU Cache 等等，都是使用队列来实现的。

操作系统，计算机网络中有许多排队问题，都是需要队列来模拟的。

### LeetCode 622. 设计循环队列

[LeetCode 622. 设计循环队列](https://leetcode-cn.com/problems/design-circular-queue/)

这道题目是 LeetCode 上唯一一道纯粹使用队列的题目。

```go
type MyCircularQueue struct {
    data []int
    tail int
    head int
    size int
}


func Constructor(k int) MyCircularQueue {
    return MyCircularQueue{
        data: make([]int, k + 1),
        tail: 0,
        head: 0,
        size: k + 1,
    }
}


func (this *MyCircularQueue) EnQueue(value int) bool {
    if this.IsFull() {
        return false
    }
    this.data[this.tail] = value
    this.tail = (this.tail + 1) % this.size
    return true
}


func (this *MyCircularQueue) DeQueue() bool {
    if this.IsEmpty() {
        return false
    }
    this.head = (this.head + 1) % this.size
    return true
}


func (this *MyCircularQueue) Front() int {
    if this.IsEmpty() {
        return -1
    }
    return this.data[this.head]
}


func (this *MyCircularQueue) Rear() int {
    if this.IsEmpty() {
        return -1
    }
    return this.data[(this.tail - 1 + this.size) % this.size]
}


func (this *MyCircularQueue) IsEmpty() bool {
    return this.head == this.tail
}


func (this *MyCircularQueue) IsFull() bool {
    return (this.tail + 1) % this.size == this.head
}


/**
 * Your MyCircularQueue object will be instantiated and called as such:
 * obj := Constructor(k);
 * param_1 := obj.EnQueue(value);
 * param_2 := obj.DeQueue();
 * param_3 := obj.Front();
 * param_4 := obj.Rear();
 * param_5 := obj.IsEmpty();
 * param_6 := obj.IsFull();
 */
```

> 结果我还是把正经循环队列用 Go 又写了一遍。
