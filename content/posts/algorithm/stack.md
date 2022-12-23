---
title: "算法与数据结构 -- 栈和队列（1）-- 栈"
date: 2022-12-23T20:51:00+08:00
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

栈是限制插入和删除操作只能在表的末端进行的一种特殊的线性表。这时，表的末端被称为栈顶，相对地，表的另一端被称为栈底。

<!--more-->

栈的基本操作有 `Push`、`Pop` 和 `Top`。`Push` 称为压栈，将元素压入栈顶；`Pop` 称为出栈，将栈顶元素弹出；`Top` 称为取栈顶元素，返回栈顶元素。对空栈的 `Pop` 和 `Top` 操作是非法的。

栈的修改是按照后进先出的原则进行的，即最后压入栈的元素最先被弹出，最先压入栈的元素最后被弹出。一般也将栈称作后进先出（LIFO）表。

栈的实现方式可以使用链表，也可以使用数组，毕竟栈就只是一个普通的线性表，正如前面阐述过的，良好的对程序的抽象应当让程序的其他部分无法得知也不需要得知栈的实际实现方式。

## 栈的链表实现

栈的一种实现方法是使用单链表，可以很简单的通过在链表顶端插入实现 Push，通过删除表顶端元素实现 Pop，通过访问表顶端元素实现 Top，通过判断表是否为空实现判空。

很显然的，这种实现方式的时间复杂度都是 O(1)。但是分配空间的代价较为昂贵，因为每次插入都需要分配新的空间，而每次删除都需要释放空间。对于 C 这种不具备垃圾回收机制的语言来说，这种实现方式的空间复杂度是 O(n)。并且 `malloc` 和 `free` 的代价是比较昂贵的。有一种优化的方法是使用内存池，但是这种方法的实现比较复杂，这里就不展开了。

## 栈的数组实现

一种更加常用且流行的方法是使用数组来模拟栈，这种方式的缺点是我们需要提前估计栈的大小。但通常情况下，尽管栈操作较多，但栈内元素的数量通常不会很多，声明一个足够大的数组并且不浪费太多空间一般都是可行的。

用数组模拟栈是很容易的，我们只需要维护一个栈顶指针，每次插入元素时，将栈顶指针加一，然后将元素插入到栈顶指针指向的位置即可。每次删除元素时，将栈顶指针减一即可。对于一个空栈，我们将栈顶指针初始化为 `-1`，这样实现起来是比较方便的。数组的实现方式的时间复杂度都是 O(1)，它是非常快的。

```go
type Stack struct {
    data []interface{}
    top  int
}

func NewStack() *Stack {
    return &Stack{
        data: make([]int, 0),
        top:  -1,
    }
}

func (s *Stack) Push(x int) {
    s.top++
    s.data = append(s.data, x)
}

func (s *Stack) Pop() (interface{}, error) {
    if s.top == -1 {
        return nil, errors.New("stack is empty")
    }
    ret := s.data[s.top]
    s.data = s.data[:s.top]
    s.top--
    return ret, nil
}

func (s *Stack) Top() interface{} {
    if s.top == -1 {
        return nil
    }
    return s.data[s.top]
}

func (s *Stack) Empty() bool {
    return s.top == -1
}
```

## 栈与递归

递归的实质是由系统维护了函数调用栈，如果你学过汇编，你就会知道，函数调用的实质是压栈和出栈。在函数调用的过程中，函数的参数、返回地址、局部变量等都会被压入栈中，而在函数返回时，这些数据都会被出栈。这些数据的出栈顺序与入栈顺序相反，这就是为什么递归函数的返回值是由最后一个调用的函数返回的，而不是第一个调用的函数返回的原因。

在支持递归的语言当中，递归就是使用栈完成的。在函数调用时，存储当前状态机状态的这些信息称为活动记录(activation record)，更多时候被叫做栈帧(stack frame)。一般现代的计算机都默认栈空间是从内存分区的高端向下增长，虽然 x86 平台并没有明文规定，但是这已经成了约定俗称的事实。许多系统中，栈的大小是有限的，这就意味着递归的深度也是有限的。如果递归的深度超过了栈的大小，那么就会出现栈溢出的情况，这时候程序就会崩溃。

很多递归是可以优化的，例如典型的尾递归一般都可以轻松的改成循环，许多时候编译器就会帮助优化尾递归。

理论上来说，在程序中使用栈来模拟递归过程是完全可行的，甚至等价的非递归程序可能比递归程序更快。但这种方式的缺点是通常代码可读性较差，并且有时需要额外优化。

## 一些练习

栈有很多典型应用，例如括号匹配、表达式求值、函数调用等等。应当看到这些类型问题就直接想到栈。

### LeetCode 504 七进制数

[LeetCode 504 七进制数](https://leetcode-cn.com/problems/base-7/)

严格来说这道题和栈没什么直接关系，这是一道数学题。但在计算出所有的余数之后，我们可以使用栈来反转这些余数，从而得到最终的结果。

```go
func convertToBase7(num int) string {
    if num == 0 {
        return "0"
    }
    sign := num < 0
    if sign {
        num = -num
    }
    stack := make([]int, 0)
    top := -1
    for num > 0 {
        top++
        if len(stack) <= top {
            stack = append(stack, num % 7)
        } else {
            stack[top] = num % 7
        }
        num /= 7
    }
    res := ""
    for top != -1 {
        res += strconv.Itoa(stack[top])
        top--
    }
    if sign {
        return "-" + res
    } else {
        return res
    }
}
```

> 对不起，这样的代码可读性不太好，我应该考虑引入更多 Go Slice 语法的。

### LeetCode 20 有效的括号

[LeetCode 20 有效的括号](https://leetcode-cn.com/problems/valid-parentheses/)

栈的经典题，简单来说，遇到左括号入栈，遇到右括号检查与栈顶括号是否匹配，如果匹配则弹栈，否则直接错误返回。最后检查栈是否为空，如果为空说明括号完全匹配。

任何取栈顶元素的时候比较如果遇到栈为空的情况都是一定不合法。

如何检查括号是否匹配？最简单的方式是 Hash 表，但如果你用的是 C，更简单的方法是写个函数直接打表实现，一个 Switch 搞定，反正括号也没几种。

有一些比较 Trick 的方法，比如直接入栈右括号，这样遇到右括号时直接检查栈顶是否与之匹配，省去了 Hash 表的查找过程。而且代码本身会简单一点，我仍然采用了比较原始的方式。

```go
func pareValid(a rune, b rune) bool {
    switch a {
        case '(':
        return b == ')'
        case '{':
        return b == '}'
        case '[':
        return b == ']'
    }
    return false
}

func isValid(s string) bool {
    stack := make([]rune, 0)
    for _, v := range s {
        if v == '[' || v == '(' || v == '{' {
            stack = append(stack, v)
        } else {
            if len(stack) == 0 {
                return false
            }
            if pareValid(stack[len(stack) - 1], v) {
                stack = stack[:len(stack) - 1]
            } else {
                return false
            }
        }
    }
    if len(stack) == 0 {
        return true
    } else {
        return false
    }
}
```

> Go 的 rune 实际上相当于 C 的 char，并且正确遍历字符串的方法只有 for range 循环。

### LeetCode 150 逆波兰表达式求值

[LeetCode 150 逆波兰表达式求值](https://leetcode-cn.com/problems/evaluate-reverse-polish-notation/)

这是一道非常经典的计算机科学问题。

计算机是如何处理复杂的算数表达式的？比如 `1 + 2 * 3`，我们是如何计算的？我们会先计算 `2 * 3`，然后再计算 `1 + 6`，最后得到 `7`。但计算机如何得到正确的运算顺序呢？

例如更复杂一点的问题，给出一个算式：

$$4.99 \times 1.06 + 5.99 + 6.99 \times 1.06=$$

在一个简单的计算器上，我们通常要先将 4.99 和 1.06 相乘，将值保存到一个临时变量 $A_1$ 中，然后将 5.99 和 $A_1$ 相加，再次保存进 $A_1$ 中，再将 6.99 和 1.06 相乘，将值保存到一个临时变量 $A_2$ 中，最后将 $A_1$ 和 $A_2$ 相加，得到最终结果。我们可以将操作顺序写作如下的表达式：

$$4.99\ 1.06 \times 5.99 + 6.99\ 1.06 \times +$$

这种表达式被称为逆波兰表达式，也叫后缀表达式，因为运算符位于操作数的后面。这种表达式的优点是不需要括号，也不需要考虑运算符的优先级，因为运算符的位置就是运算的顺序。

对于计算机来说，求解逆波兰表达式的方法就非常简单了，只要从左到右遍历表达式，遇到数字将其入栈，遇到运算符时就将其作用于栈顶的两个元素，再将所得结果入栈。最后栈中只剩下一个元素，就是表达式的值。

了解了逆波兰表达式的原理，解题就变得很简单了。

```go
func evalRPN(tokens []string) int {
    stack := make([]int, 0)
    for _, v := range tokens {
        // 这里用到了函数特性，如果 Atoi() 遇到非数字转换失败，则会返回一个错误，利用这个特性判断是否为运算符
        if i, err := strconv.Atoi(v); err == nil {
            stack = append(stack, i)
        } else {
            num1, num2 := stack[len(stack) - 2], stack[len(stack) - 1]
            stack = stack[:len(stack) - 2]
            switch v {
            case "+":
                stack = append(stack, num1 + num2)
            case "-":
                stack = append(stack, num1 - num2)
            case "*":
                stack = append(stack, num1 * num2)
            case "/":
                stack = append(stack, num1 / num2)
            }
        }
    }
    return stack[0]
}
```

实际上，你再解决一个如何将中缀表达式转换成后缀表达式的问题就可以完成一个计算器了。

提示一下，这个问题也是用栈来完成的，但我没有找到 LeetCode 上对应的题目，也许以后会写个加强版附录把一些值得讨论的问题都放进去。