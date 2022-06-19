---
title: 过去手写 HTML 的方法 - HTML 结构
date: 2022-05-06T17:28:00+08:00
draft: true
categories: [前端]
tags: [frontend]
series: ["过去手写 HTML 的方法"]
---

HTML 由元素构成，本文将解析一个基本的 HTML 文档是怎样写成的。

<!--more-->

HTML 在没有特殊的样式指定的情况下将以自上到下的顺序在浏览器中渲染，因此看到内容的顺序也自然是从上到下的，这个行为可以被改变，这部分内容会在后面的排版方式中谈到。

## 标签

HTML 文档使用标记来注明内容，HTML 标记包含一些元素，这些元素通过标签来标记特定的作用，标签的组成为一对尖括号 `<>` 加上内部的元素名组成。

元素名在 XHTML 中强制为小写，而在 HTML 中不区分大小写。

对于一个完整的元素，例如

```html
<p>Hello World!</p>
```

两端的标签分别称为 `开始标签` 和 `结束标签`，标签包裹的内容也称为 `内容`，包括两端的标签和内容即为一个完整的元素。

元素间可以嵌套。例如我们要加粗文字，可以将表示加粗的标签放在其中：

```html
<p>Hello <b>World!</b></p>
```

标签嵌套时应注意顺序。

### 块级元素与内联元素

对于一个 HTML 文档内容，一些元素会自动换行，而另一些则不会，这并不是随机的，而是有规律性的。

这些会自动换行的元素，称为块级元素 —— 在页面中以块的形式呈现，通常用于展示结构化内容的元素，如段落，列表等，都属于块级元素。

不会自动换行的元素，称为内联元素 —— 他出现在块级元素中，并环绕文档的一部分，而不是一整个段落或一组内容，例如超链接，图片，加粗，斜体等。

### 空元素

并不是所有元素都完整的包含了开始标签，内容，结束标签这三部分，有些元素只有一个标签，通常他表示在这个地方插入或嵌入一些东西，例如

- `br` 换行
- `hr` 分隔线
- `img` 插入图片

等。

有时我们也把这种只有一个标签的元素叫单标签元素，将普通的元素称作双标签元素，但本文写作不使用这种称呼。

另，根据 XML 规范，所有标签都应闭合，因此在 XHTML 中，空元素应写成

```html
<br />
```

来创建一个自闭合标签。

### 属性

元素还可以拥有属性，属性包含了一个元素的额外信息，这些信息不会直接出现在实际的内容中，例如：

```html
<a href="https://www.ddupan.top/" id="blog" class="a-link">我的博客</a>
```

一个属性必须包含如下内容

- 空格，在属性名和元素名之间，以及多个属性之间，都使用空格分割
- 属性名和属性值之间使用等于号
- 属性值通常使用一对引号扩起来，引号可以省略，但在一些场景下可能会出现歧义，因此推荐使用引号

对于引号，单引号和双引号均可，两种引号可以互相嵌套使用。

有时也存在没有值的属性，它代表一个布尔值属性，例如对于一个表单输入，可以标记为不允许用户输入信息，方法是在 `input` 元素中加入 `disabled` 属性

```html
<input type="text" disabled>
```

但对于 XHTML，这种写法是不合法的，因为 XML 强制要求所有属性均有值，因此要写成：

```html
<input type="text" disabled="disabled" />
```

## 一个完整的文档

HTML 基本结构想必并不陌生，一个基础的 XHTML 1.0 的文档是这个样子的

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
        <title>无标题文档</title>
    </head>

    <body>
    </body>
</html>
```

XHTML 1.0 可以近似的看作 HTML 4.01，不使用严格模式的情况下使用 HTML 4 的语法多数浏览器也都可以正常处理。

这个基本结构中，有文档头，有一些 HTML 标签，我们先来解析这些东西。

### &lt;!DOCTYPE&gt;

严格来说 `<!DOCTYPE>` 并不是一个标签，而只是一个声明，他声明了当前使用了何种 HTML 版本。

这个声明必须放在文档的开始处，否则部分浏览器的行为会出问题。

对于 HTML 4 和 XHTML，由于基于 `SGML`，因此需要给出标记语言规则的 DTD，而在 HTML 5 时由于不再基于 SGML，因而简化成了 `<!DOCTYPE html>`

另，由于早期版本的 HTML 不含这个声明，因此浏览器在发现一个 HTML 文档不含此声明时会进入到一个特殊的名为怪异模式的特殊渲染模式，用于兼容 Netscape Navigator 和 Internet Explorer 5 及其更早版本的渲染引擎，因当时 HTML 还没有形成标准。

### &lt;html&gt;

`<html>` 元素表示一个 HTML 文档的根，其他的所有元素必须都是 `<html>` 的子元素

`<html>` 的子元素只有两种，为 `<head>` 和 `<body>`

`<html>` 元素属性中的 `xmlns` 指定了 XHTML 的命名空间，HTML 可省略此标记。

`<html>` 中通常还有 `lang` 属性，指定此 HTML 文档内容的主要语言。

### &lt;head&gt;

`head` 元素指定文档的元数据，他包括文档的标题，引用的文档样式和脚本等。

#### head 里有什么

一个大型的网页会包含许多元数据，但最开始，我们通常都会有的内容其实不太多，因此不打算把这部分单独写一篇文，而直接写在这里。

<!--通常他包括 `<title>`，`<link>`，`<meta>`，`<style>`，`<script>` 等。-->

##### 标题

这个标题指的是显示在浏览器标题栏或选项卡上的文字，使用元素 `<title>` 实现，元素的内容即为该页面的标题。

##### 元数据

`<meta>` 用于描述文档的元数据，大型网站会使用很多元数据，但现阶段只需要知道一样东西，即实例中的

```html
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
```

这行指定了响应头，表明这个文档类型为 HTML，且使用 UTF-8 编码。

在现在的网页开发中，指定编码有了一个更短的形式，为

```html
<meta charset="utf-8">
```

##### 链接

可以在 `head` 中指定应用的 CSS，JavaScript 和 显示在浏览器中的图标。

例如，为网站增加图标

```html
<link rel="icon" href="favicon.ico" type="image/x-icon">
```

又例如，应用 CSS

若使用外部链接的方式，则使用如下的方式

```html
<link rel="stylesheet" href="style.css">
```

若选择内嵌样式，则使用 `<style>` 标签包裹，并写入 CSS

```html
<style>
    html {
        margin: 0;
        box-sizing: border-box;
    }
</style>
```

再例如，应用 JavaScript

在 `head` 中应用 JavaScript 并不是现在多数时候推荐的方法，因大多数 JavaScript 并不需要在 HTML 未完全显示时加载。

如

```html
<script src="app.js"></script>
```

注意，`script` 并不是一个空元素，尽管这里的 `script` 元素没有内容，也可以不引用一个 JavaScript 文件，而是将 JavaScript 代码直接写在元素内容中，例如：

```html
<script defer>
    link = document.getElementById("link");
    link.innerHTML = "一个链接！"
</script>
```

加入 `defer` 的目的是让浏览器在 HTML 加载完成后再解析 JavaScript。

### &lt;body&gt;

`body` 元素表示文档的正文内容。

即所有要显示在浏览器中的内容都放在 `body` 元素下面作为其子元素存在。

HTML 在现代网页开发中专注于描述页面的结构层次和页面的内容，下篇文章将详述如何使用 HTML 来建立一个具有结构的页面。
