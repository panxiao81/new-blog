---
title: "一个简单 C 语言程序中两个的不太容易推断的错误"
date: 2022-04-15T22:17:23+08:00
draft: false
categories: ['Programming']
tags: ['programming']
---


> Reference
> 
> [自增/自减运算符](https://zh.cppreference.com/w/c/language/operator_incdec)
> 
> [求值顺序](https://zh.cppreference.com/w/c/language/eval_order)

我们有一个简短的小程序。

```c
#include <stdio.h>
#include <string.h>
int main(int argc, char const *argv[]) {
    char *p = "abcde";
    char m, n, x, y;
    m = *p;
    n = *(p++);
    x = *p++;
    y = ++(*p);
    printf("%c,%c,%c,%c", m, n, x, y);
    return 0;
}
```

显然，不可能有人真的在正经场合写出这种代码。这是个很典型的指针练习的代码，但他却犯了两个错误，一是他出现了一个段错误，另一个则是输出行为不符合预期。

我们最初的预期输出为

```sh
a,b,b,c
```

但首先第二个输出就不符合预期

## 一个 UB 行为

n 的结果是 `a`，我们最初以为由于括号运算符会使 `p++` 最先执行，所以应该会自增 `1`，但显然结果并不是这样。

根据 `cppreference` 的文档，我们得知了一个与过去的认知不太完全相符的实际实现。

### 自增自减运算符的真相

在初学 C 语言时，我们都会学到:

> 如果是前缀自增，则先自增，再使用，而后缀自增则先使用再自增。

我们也能轻易的从实践中验证这个结论，因此我们认同了他并认为他就是这样的实现。

但这个案例由于不容易解释现象，证明了这个理解并不完全正确。

事实上，前缀自增确实是直接自增了没错，这个表达式也直接返回自增后的值，但后缀自增其实并不是简单的先使用再自增。

执行一个后自增运算会产生一个 `副效应`, 这个副效应会让值自增，但 C 语言规定，副效应会 `在下个序列点或在那之前完成`。

虽然赋值运算确实存在一个序列点，但我们又存在:

> 直接赋值运算符与所有复合赋值运算符的副效应（修改左参数）后序于左右参数的值计算（但非其副效应）。

也就是说，虽然赋值的确存在序列点，但右值运算的副效应却不包含在内，即顺序依然是不确定的。

既然全部不确定，那么就适用于这一条了：

> 若一个标量对象上的副效应与另一个使用同一标量对象之值的值计算相对无顺序，则行为未定义。

即，对于 `n = *(p++)` 这个表达式而言，括号结束就自增是合理的，整个赋值语句结束之后才自增是合理的，甚至这时候把你的硬盘格式化都是合理的（雾

## 一个段错误

多数时候，段错误通常认为是数组越界，我一开始也是这样认为的，那么我们来打开 GDB 调试一下吧

```sh
$ make
gcc -g --std=c11 -c hello.c -Wall
gcc -o hello hello.o
$ gdb ./hello
GNU gdb (Ubuntu 9.2-0ubuntu1~20.04.1) 9.2
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./hello...
(gdb) r
Starting program: /home/panxiao81/code/c-lang/hello

Program received signal SIGSEGV, Segmentation fault.
0x00005555555551a5 in main (argc=1, argv=0x7fffffffe1c8) at hello.c:10
10              y = ++(*p);
(gdb) p p
$1 = 0x555555556006 "cde"
(gdb) p *p
$2 = 99 'c'
```

显然这时候我们能够看到，当前的指针 `p` 所指向的地址并没有越界，且还有不少剩余。

那么此时的段错误到底在说什么？

有些时候，当我们对一些东西有了一定熟练度之后，会忘记一些东西的本来的定义。

段错误指，当程序访问了他不能访问的内存空间时，保护模式下的 CPU 会捕捉到，并保护这段内存空间，操作系统会捕捉到这段异常，并发送信号让程序终止，并抛出段错误异常。

而这个不能访问的内存空间存在很多可能性，例如，程序访问了本就不存在的内存，或访问的内存并非属于这个程序，或访问的内存不可读或不可写。

一般我们所说的数组下标越界，即是因为程序访问了原本不属于它的内存地址空间。

而在这个例子里，程序实际是因为向只读的内存空间里写入数据而导致触发内存保护。

### 只读的字符串

C 语言与 Java 一类的语言不同，他始终是面向值的。

在程序开头的一行，即：

```c
char *s = "abcde";
```

这个存放着 `abcde` 的字符串会随着程序运行时将这个已经固定的字符串分配给一个只读的内存空间，并根据类型转换规则，将这个字符串的首字符地址付给左值的字符类型指针。

即，这个指针 `s` 存放了一个指向只读的字符串 `abcde` 的首字符地址。

在 GDB 的调试中，我们可以很轻松的发现，程序出现段错误的位置即第 10 行.

这一行，我们使用指针解引用运算符取了当前 `p` 指针指向的值，对这个值进行了自增操作。

这时回顾刚才说过的事情，我们对一个只读内存空间的数据进行了赋值操作。

自然的，CPU 发现了这个错误，触发了内存保护，抛出了段错误异常。

对于一个字符串，如果我们已有一个字符串地址空间，将字符串重新不能直接赋值，而需要使用 `strcpy()` 函数完成，即：

```c
char* str = (char*)malloc(100 * sizeof(char));
// This is wrong
str = "I am a new string";
// This is the right way
strcpy(str, "I am a new string");
```

C 是真的属于那种入门容易，但几乎没人能说自己精通的一门编程语言，坑还是非常多的。

CppReference 中说到：

> 因为涉及副效应，必须谨慎地使用自增和自减运算符，以避免违背定序规则所致的未定义行为。

国内的大量 C 语言教材，不但把这些未定义行为当作是考点，且强行赋予解释，并当作所谓 `特性`， 却忽略了这些未定义行为在不同平台，不同编译器下会产生不同的结果。

显然我不建议任何 C 语言新手去学习和纠结那些 UB 行为背后的行为，UB 就是 UB，不要把他当作特性。

这些自增自减运算符由于其过于容易产生 UB，更加现代的编程语言中几乎很多都对其赋予了更加严格的限制，例如 Rust 中只保留了后缀自增，Go 中不但只保留了后缀自增，且规定了在语句中多次使用自增运算符是明确的语法错误。这些也成为了我们使用 C 一类的语言编程时的注意事项，即避免语义的不确定性。

真正的工程代码，通常在刚开始开发时不会存在问题，而在后续的维护中发现代码问题越来越多，如果这时不能下定决心重构，而是在这样的代码基础上继续强行维护的话，一段时间之后这样的代码就成为了所谓屎山代码。与其与纠结这样的语言特性为什么会成为这个样子，不如在一开始就应该养成良好的编码习惯，把这种编码方式直接当成错误处理更有利于长远的技术发展。
