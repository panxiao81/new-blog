---
title: Mixing Basics:Compressor Cheaper 1
date: 2020-02-20
categories: [音乐, 混音]
tags: [mixing]
series: ["Mixing Basics"]

---

## 总览

压缩器（Compressor）或动态范围压缩（Dynamic range compression）是混音中的好伙伴。你通常会在绝大多数轨道上都见到他的身影。本文是压缩器的第一章节，会介绍压缩器的基础，同时给出基本的例子来实际体会压缩器的用途。

在实际使用压缩器之前，我们先来了解一下为什么会出现压缩器。

压缩器是为了保证广播运行而出现的，为了不让这篇文章的话题跑的太远，感兴趣的可以去看Pro Audio Files上的这篇文章。

[A Brief History of Audio Compression](https://theproaudiofiles.com/video/a-brief-history-compression-explained/) （音频压缩简史）

由于当音频信号输入电平过大时，简单的说会产生信号失真，因此压缩器被发明了出来。

随着数字音频工作站（DAW）的出现，压缩器也自然成为了最基础最基础的效果器之一，无论你用的是哪家DAW都不会少了压缩器，而数字压缩器也成为了调整最自由，最方便的压缩器。

本文就将以Pro tools自带的压缩器为例来讲解压缩器的基本知识，以及给出压缩器基本的用法。

## 一、认识压缩器

![Avid Dyn3 Compressor/Limiter](/images/mixing/Compresor/cheaper1/AvidDyn3CompressorLimiter.png)
<!--Avid Dyn3 Compressor/Limiter-->

压缩器通常会有4个核心值，分别为Threshold（阈值，也称阀值）、Ratio（压缩比）、Attack Time（启动时间）、Release Time（施放时间），同时还有对输出电平进行增益补偿的Gain（增益），部分压缩器还可调整Knee（这里译为拐点），也可能带有内置的Side-Chain（侧链）功能，图1为Pro tools自带的Avid Dyn3 Compressor/Limiter，他具备了以上提到的所有功能，是Pro tools中非常好用的自带插件之一。

同时，压缩器还会具备三种表（Meter），分别为输入电平表（Input Meter）、输出电平表（Output Meter）以及动态表（Dynamic Range Meter，简称GR表），通常我们要使用的都是GR表来观察压缩量的变化。

### 1、Threshold

Threshold决定了压缩器何时开始工作，顾名思义，压缩器需要有一个值来决定何时开始压缩，而高于Threshold的部分会依照压缩比来进行有比例的电平减少。（注意此处并非是在高于阈值时对整个音频信号进行压缩，而是只有阈值以上的部分会进行压缩。）

我们通常通过Threshold来进行压缩量的调整，也就是说当你打开一个压缩器要进行使用时，Threshold通常是第一个需要调整的参数。

### 2、 Ratio（压缩比）

当决定了压缩的量之后，接下来就要决定压缩的大小，Ratio（压缩比）就登场了。

在信号高于电平之后，高于电平的信号就会被按照设定的压缩比进行压缩。

做一个简单的算术题：假设我们现在设定的压缩比为4:1，而高于Threshold的电平大小为4dB，那么在经过压缩器处理之后输出的结果为，高于压缩比的部分剩下1dB，而有3dB会被压缩掉，这3dB就会在GR表上表现出来。

通常压缩器都是向下压缩（输出信号比输入信号小），但仍然有一种绝大多数压缩器都不具备但理论上存在的一种压缩模式为向上压缩，这种时候压缩比可以设置为类似于1:4的方式，也就是当高于Threshold的电平为1dB时，信号放大4倍成为4dB，这种特殊的压缩不在本文的讨论范围。

<img src="https://ddupan.top/wp-content/uploads/2020/02/Compression_ratio.png" alt="Compression_ratio" title="相同信号超过Threshold时，不同的压缩比的图示" style="zoom: 50%;" />

<!--相同音频信号超过Threshold时，不同的压缩比的图示-->

### 3、Attack and Release Time

 当信号超过Threshold时，压缩器并不是立刻开始工作的，而是要经过一段时间才能完全达到设定的压缩比的工作状态，同理在电平回到Threshold下时，压缩器也要经过一段时间才会回到完全不工作的状态，而这个时间在压缩器当中就成为了Attack Time和Release Time。

![Attack and Release](https://ddupan.top/wp-content/uploads/2020/02/1920px-Audio_Compression_Attack_and_Release-2.svg_.png "Attack和Release Time的工作方式图示")

<!--Attack和Release Time的工作方式图示-->

上图用电平变化的图示来形象的表现出了Attack和Release Time的工作方式，注意在Attack和Release的时间内，压缩量是缓慢变化的，而不是突然间的变化。且在变化过程中的速度也是不确定的，且没有统一的标准。

### 4、增益补偿（Make-up Gain / Gain）

当电平信号被压缩后，音量一定会出现变化，因此我们需要一个增益将信号重新提升到原来的响度，因此压缩器通常都会具备一个增益功能，通常这个压缩器自带的电平补偿会根据不同的压缩器而具有不同的音色（音染），尤其是在使用硬件压缩器或插件复刻的硬件压缩器时。

### 5、拐点（Knee）

![Compression Knee](https://ddupan.top/wp-content/uploads/2020/02/1280px-Compression_knee.svg_.png "硬拐点与软拐点")

<!--硬拐点与软拐点-->

拐点决定了在启动时间与施放时间中压缩器的变化是突然的，还是较缓慢的，通常硬拐点时压缩器的压缩痕迹会较为明显，而软拐点时会较为自然。

### 6、侧链 （Side-Chain）

通常在电子音乐制作中会经常使用侧链压缩来制造抽吸感，侧链指的是用另一个音频信号来触发Threshold来对原本的音频信号进行压缩。

![Sidechain](https://ddupan.top/wp-content/uploads/2020/02/300px-Compressor_Sidechain.svg_.png "前馈式压缩器中侧链的工作方式")

<!--前馈式压缩器中侧链的工作方式-->

## 二、压缩器的用途

通过以上的说明，读者应该对压缩器已经有了一些基本的了解，接下来笔者将把压缩器的使用分为三个大部分来进行说明。

笔者认为，压缩器的用途基本分三种，分别为控制动态、塑造音色，以及染色。

### 1、控制动态

控制动态应该是压缩器出现之出最先出现的，也是最原本的目的，就是将大小不一的电平信号进行处理后让其相对平稳，以便更容易融入混音。

笔者将以一段人声录音为例来说明压缩器在此场景下的使用

> 插入Example1_Original.mp3

这是一段没有经过任何处理的人声录音，我们可以听出人声忽大忽小，因此很难融入混音中，当音量过大时会从音乐中突出，而音量小时又会被音乐埋没，我们当然可以通过调音台上的音量推子进行自动化来解决这个问题，但这样解决起来过于麻烦，因此我们使用压缩器来处理

![Example1](https://ddupan.top/wp-content/uploads/2020/02/Example1_Compress.png)

<!--压缩器的调整-->

如上图所示，笔者对人声进行了压缩处理。Threshold设置为人声电平最小的位置左右，这样在压缩后电平整体提升后可以将最小音量的位置提起来，同时又不会导致过度压缩，通常情况下都要避免过度压缩，因为过度压缩会使音频变得极为不自然，通常应将单个压缩器的压缩量保持在6dB以内。

Ratio设置在3:1的一个中度的压缩比，使人声动态得到一定控制的同时又不会导致动态被压缩 的过平而导致失去人声美感。在不同的压缩目的中会使用不同的压缩比，通常2:1以下为轻度压缩，2:1-6:1为中度压缩，8:1以上为重度压缩，这三种压缩比配合不同的压缩量会产生不同的音色。

Attack设置在偏快的位置，通常压缩器的Attack值会有两个甜点，第一个甜点相对自然，不会有过度压缩感，而第二个甜点会产生音色的变化，这里将Attack设置在第一个甜点，判断甜点可以通过耳朵听来辨别，会有明显的音色变化，较快的值会使信号更平滑，而较慢的值会带来更多的冲击感。

Release设置在偏快的位置，基本保留了压缩器的默认值，通过观察GR表可以发现当一段乐句结束时压缩器也正好停止工作，这就是一个比较恰当的点，如果释放过快会产生明显的抽吸感，如果释放太慢会使下一句开始时压缩依然没有被释放而产生明显的音乐性丧失，尤其在对鼓的压缩时更为明显，同时释放过慢也会带来更多的不自然的压缩感。

经过压缩之后，我们得到了另一段处理后的音频

> 插入Example1_Compressed.mp3

上文说到通常情况下应避免过度压缩，过度压缩之后会得到一个非常不自然，人工化严重的，没有音乐性的音色。笔者在以上的处理结果中调整了Attack和Release time得到了压缩感严重的极端的声音。

> 插入Example1_Overcompressed.mp3

请读者将这条过度压缩的音频与正常压缩的音频进行对比。

可以听到，就此条例子而言，不但动态全无失去了人声音乐性，并且音色上发生了很大的变化，增加了压缩带来的可闻的低频失真，这显然不是我们想在人声上达到的效果。

小总结，在为了进行控制动态而调整压缩器时，先调整Threshold选择合适的位置，再调整压缩比得到正确的不失动态同时又使动态得到平衡的值，最后调整Attack和Release到合适的位置，通常都为较快的位置。

### 2、塑造音色

你可能会问，塑造音色与压缩器存在什么关系呢？

如果你对合成器有所了解，你一定会知道合成器塑造音色的重要方式是通过Amp调整ADSR包络来塑造基本的音色构成。即便是你没有使用合成器的经历，返回上一节看Attack and Release Time的工作方式你应该也能有所发现。声音的起振和衰减经过压缩之后产生了变化，因此我们也可以通过压缩器来强化这种变化。

最常见的使用方式就是强化打击感。我们使用一个慢速的启动时间来保留较多的音头，之后将音头后的衰减部分压缩，因此我们得到了一个反差更大的波形，因而强化了对比，从而强化了打击感。我们此时只需要将增益调整回压缩前衰减部分的电平即可强化打击感。

> 插入Example2_Original.mp3

> 插入Example2_Compressed.mp3

同理，通过Attack和Release Time的控制，我们还可以做到延长衰减，方法是使用快速的启动和施放时间，使音头被压缩后快速恢复，之后使用Gain将整体电平提升即可。

### 3、染色

许多硬件压缩器有着非常不同的声音特性，而这些声音的“不准确性”往往是具有音乐性的，因此我们可以单独使用想要的压缩器只压缩1dB来得到想得到的硬件染色。此处不展开讨论。

## 三、撕声消除器（De-esser）

在进行压缩或EQ的高频提升后，我们往往会发现人声的撕声也变得刺耳，此时我们可以使用压缩器的侧链，当撕声的频率（通常是4k-6k）进入压缩器，当撕声出现时，电平超过阈值而对整体的信号进行压缩。这种传统的方式存在的问题是这样进行的是全频段的压缩，而多数时候我们只想单独对撕声进行处理，这时我们可以使用多段压缩或专用的撕声消除器（De-esser）来完成这项工作。

## 总结

压缩器是混音最重要的四项效果器之一，原理并不复杂但使用方法很多，下一节我们将会讲述更多不同的压缩器的种类及他们的特点。

> 参考文献：
>
> Wikipedia Dynamic range compression词条 https://en.wikipedia.org/wiki/Dynamic_range_compression
>
> 《混音指南》/（英）伊扎基（Izhaki,R.）著；雷伟译 —— 北京：人民邮电出版社，2010.11