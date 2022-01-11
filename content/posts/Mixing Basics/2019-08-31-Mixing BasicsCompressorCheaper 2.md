---
title: Mixing Basics:Compressor Cheaper 2
date: 2020-02-20
categories: [音乐, 混音]
tags: [mixing]
series: ["Mixing Basics"]

---

# Mixing Basics: Compressor: Cheaper 2

## 总览

上节我们讲了压缩器的基本知识，以及压缩器的三种使用场景。但除了那一种压缩器之外，我们还有许许多多不同的压缩器，无论是硬件，还是不同的插件。这一节我们将讲述压缩器的几种不同类型以及他们的特点，使各位能够知道何时何地去怎样使用它们。

如果你已经摆弄过一些压缩器插件，你应该会发现，一些插件提供了许多工作模式。你可能见到过类似“Opto”、“FET”或“VCA”等字眼，其实他们代表了不同类型的压缩器。本文将着重介绍4种非常常见的压缩器类型：“电压控制放大电路”或“VCA”、“场效应晶体管”或“FET”、“光电”或“Optical”以及“可变放大系数管（或电子管）”或“Variable-Mu（Tube）”四种类型的压缩，并且会提到两类不太常见的压缩“二极管桥”或“Diode Bridge”以及“脉宽调制”或“PWM”两种压缩。

## 一、 电压控制放大电路压缩器（VCA）

VCA压缩可谓是现在最常见的压缩器类型了，你见到的可自由调整参数的压缩器多半都属于这种类型。他可调范围宽，并且通常不会带来太多染色，因此无论是想要控制动态或是塑造音色都可以使用，同时VCA压缩还经常使用在总线或乐器总线上来增加色彩或增加混音的融合度。VCA压缩也经常在录音阶段和母带阶段使用。

经典的，知名的VCA压缩有：SSL G-Master Buss Compressor、API 2500、Empirical Labs Distressor.

## 二、场效应晶体管压缩器（FET）

提到FET压缩，就一定绕不开一串数字：“1176”，这串数字所代表的是一个传奇：“Universal Audio 1176 Limiting Amplifier”。

FET压缩的特点就是快，出奇的快，他有着非常快速的反应，以1176为例，1176最快的释放时间是50毫秒，最慢的施放时间是1100毫秒；最快的启动时间只有区区20毫秒，而最慢的启动时间也有800毫秒。并且FET压缩器处理的通常是峰值，因此当你想拥有冲击性的，明显处理过的声音的时候，多半都会选择FET压缩。并且FET在多数声音时都有非常好的表现，笔者很喜欢在鼓和贝斯上使用1176来塑造有冲击力的，有活力的声音。

虽然FET压缩最初并不被设计为总线压缩，但FET压缩在鼓组的压缩上也有不错的表现。

提到1176，有两个小Tips跟大家分享，就是1176的两种特殊的使用方法。

### 1、Dr.Pepper 1176 Setting

Dr.Pepper在这里并不是最开始使用这个设置的人，而是源于一种美国饮料的名字。上世纪60年代，Dr.Pepper饮料的公司出了一个电视广告，上面说你应该每天喝三次饮料，上午10点，下午2点和下午4点，并且一些美国的混音师们发现，在1176上这样的设置能得到很有特点的声音，因此他们借助了这个广告来记忆这个设置。

![Dr.pepper](https://ddupan.top/wp-content/uploads/2020/02/DrPepper.png "Dr.Pepper的宣传画" )
<--Dr.Pepper的宣传画-->

他代表将Attack设置在10点钟方向（度数为3），Release设置在2点钟方向（度数为5），而4则作为压缩比。这样你就得到了Dr.Pepper Setting。
![DrPepperSetting](https://ddupan.top/wp-content/uploads/2020/02/DrPepperSetting.png " Dr.Pepper Setting")
<-- Dr.Pepper Setting（图中插件为Waves CLA-76）-->

接着用Input调节压缩量，Output调整输出电平，调整到听起来不错的位置。这样就得到了一个富有侵略性的声音。下面给一对对比音频。（使用Waves CLA-76）

>插入DrPepper_Original.mp3
>
>插入DrPepper_Compressed.mp3

### 2、 British（Nuke）Mode

1176原本被设计为按下哪个按钮，就使用哪种压缩比，原本美国的混音师们都是这么用的。但事情到了英国人的手里却发生了变化，有的混音师发现，1176的压缩比按钮可以被同时全部按下，之后就得到了一种极端的，夸张的声音，这个模式下压缩比介于19：1至21:1中间， Attack和Release基本没有作用。因为这种模式过于有名，因此现在的插件版1176基本上也都具有这个功能。下面是一个例子。（使用Waves CLA-76）

>插入BritMode_Original.mp3
>
>插入BritMode_Compressed.mp3

P.S. 1176的Attack和Release并不是直觉的数字小是最快，而是数字大是最快，因此最快的位置应在最右边，而最慢的在最左边。

## 三、光电压缩器（Optical）

这种压缩器比较特别，他使用光电管来充当感应器，而使用灯泡的亮度来决定压缩量的大小。由于其物理特性，导致其具有很慢的启动和释放时间。可以将其当成并非对峰值进行处理，而是对平均响度进行压缩。也因其特点，Opto类型压缩并不适合用作瞬时动态的控制，但其有着缓慢，自然的声音特性，因此只要不使用的很过分，Opto压缩都能得到一种自然的，压缩感不是很强的声音。

这类压缩器的代表有Teletronix LA-2A（电子管版本）、Teletronix LA-3A（晶体管版本）以及Tube-Tech CL1B

![LA-2A](https://ddupan.top/wp-content/uploads/2020/02/teletronix-la-2a.jpg "Teletronix LA-2A")
<!--Teletronix LA-2A-->

以LA-2A为首的Opto压缩通常没有很多可以调整的东西，因此使用起来也非常简单，同时Opto压缩也常被用在总线压缩以及各类乐器，人声的压缩，可以说是仅次于VCA压缩的万能压缩器。

提到LA-2A，有趣的是LA-2A有一个螺丝用来控制频率感应的变化，因此早年间LA-2A也被当作De-esser来使用。

## 四、可变放大系数（也称电子管）压缩（Variable-Mu or Tube）

电子管压缩器是最早出现的压缩类型，与上文带有电子管的压缩器不同，电子管压缩使用电子管本身来呈现压缩后的声音。电子管压缩有几大不同于其他压缩器的特点：它有非常软的拐点；他有非常非常慢的启动施放时间，当输入的电平越来越大时压缩比也会增加。但我们通常不使用电子管压缩来处理很多的压缩，而是仅仅用电子管压缩压1dB左右的量，使音频信号获得电子管独有的温暖的，平滑的，以及经典模拟设备的声音。

电子管压缩中最出名的传奇型号莫过于Fairchild 670（它还有一个单声道版本Fairchild 660，但内部电路设计不同），只要稍微过一点点就能得到有独特韵味的声音。

![UAD Fairchild](https://ddupan.top/wp-content/uploads/2020/02/fairchild_collection_thumb__2x-min.jpg "UAD Fairchild Collection")
<!--UAD版本的Fairchild压缩器插件-->

下面是人声使用的一段对比（人声总线压缩，使用Waves Puigchild 670）

> 插入VariMu_Original.mp3
> 
> 插入VariMu_Compressed.mp3

除去Fairchild 670外，其他知名的型号还有Manley Vari Mu，UA-175等。

---

接下来是不太常见的压缩器介绍，说句实在话，笔者也是在写这篇文章时才了解到有这两种压缩器

## 五、 二极管桥（Diode Bridge）

这种压缩器很长一段时间内被用于自动控制电平大小，这种压缩器在上世纪60年代时出现了一些，但现代使用的并不多。因为二极管有独特的谐波失真，所以这种压缩器也有着与众不同的染色，

这种压缩知名的型号有Neve 33609，Neve 2264。

![Neve 33609](https://ddupan.top/wp-content/uploads/2020/02/33609-stereo-compressor-4-35-p.jpg "Neve 33609")
<!--Neve 33609-->

## 六、 脉宽调制（PWM）

这个词第一次听说时还是近几年手机行业的屏幕因为低频率的PWM调整屏幕亮度而产生的闪屏而被诟病，但写这篇文章时却发现居然有这种设计的压缩器。至于为什么这种压缩器几乎没有人听说过则是因为设计这种压缩器的难度过高，依照业内人士的说法是“只有顶尖的工程师才能设计好它。”这种压缩被设计为几乎没有缺点的压缩器。依照Google查到的信息，VCA压缩在控制电压变化时声音会失真，如果在输出时有一个开关来控制能量，那么就能更好的得到一个准确的平均控制电平，使声音更好，原文给出了为数不多的PWM压缩的一种，为Great River PWM 501。
![PWM501](https://ddupan.top/wp-content/uploads/2020/02/pwm501.jpg)
<!--Great River PWM 501 为500模块化设计-->

---

以上就是压缩器的几种类型，希望各位在选择压缩器时能知道自己想要的音色时选择合适的压缩器。

>参考文献：
>《[Audio Compression Overview: Types of audio compressor]( https://www.4soundengineers.com/audio-compression-overview-types-of-audio-compressor)》By Lesa Soundz
>
>《[What are the different types of compressors?](https://audiohertz.com/2017/08/07/what-are-the-different-types-of-compressors) 》 By David Silverstein 