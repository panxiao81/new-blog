---
title:  chroot 进不同架构的 rootfs 中
date: 2022-06-23T03:03:00+08:00
categories: [Linux]
tags: [linux]
---

我们如何在当前的 PC 上模拟一个大端序系统呢？当然你可以用 QEMU 来完全模拟一个其他架构的计算机，但显然这样的效率比较低。其实可以利用 QEMU-User 来实现 chroot 进入一个与宿主机完全不同架构的环境，这是依靠 QEMU-User 与 binfmt 实现的。

<!--more-->

先来看看这个神奇的电脑吧。

```sh
lscpu
Architecture:        mips
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Big Endian
Address sizes:       45 bits physical, 48 bits virtual
CPU(s):              8
On-line CPU(s) list: 0-7
Thread(s) per core:  1
Core(s) per socket:  1
Socket(s):           8
NUMA node(s):        1
Vendor ID:           GenuineIntel
CPU family:          6
Model:               158
Model name:          Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
Stepping:            10
CPU MHz:             2208.000
BogoMIPS:            4416.00
Virtualization:      VT-x
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            9216K
NUMA node0 CPU(s):   0-7
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc cpuid tsc_known_freq pni pclmulqdq vmx ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single pti ssbd ibrs ibpb stibp tpr_shadow vnmi ept vpid fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap clflushopt xsaveopt xsavec xgetbv1 xsaves arat md_clear flush_l1d arch_capabilities
```

CPU 是 Intel 的 CPU，支持的指令集也显然都是 x86 那一套，但这是一块 MIPS 架构的 Intel CPU？

CS:APP 上有关于字节序的介绍，而且也推荐我们找一些大端的计算机来尝试。但现在我们能接触到的最多的大端电脑是 PowerPC Mac，无论是装一台虚拟机还是买一台真机都显然太不合适了。

我曾经看到有种方法可以在普通 PC 上调试其他架构程序的方法，是利用到了 QEMU-User，QEMU-User 可以只仿真用户态程序，这为我们的操作提供了不少便利。

但一般的程序都需要链接系统库，我又不想搭建一套交叉编译工具链，这显然把简单的问题搞复杂了，因此我们打算起一个新的 rootfs，chroot 进去来获得一套完整的 MIPS (这货是大端序) 的环境。

## 安装 QEMU-User

由于我们需要在 chroot 环境下使用，最好是安装静态编译版本的 QEMU-User，这两者在 Ubuntu 的源中都有。

```sh
sudo apt install qemu-user-static
```

这一步会自动安装 binfmt 作为依赖，且帮我们自动配置好 binfmt，不需要我们手动配置。

非常简单的，我们可以通过 `qemu-arch-static` 来使用其他架构的程序，或此时直接运行其他架构的程序就可以了。

## 准备 chroot 环境

由于小众架构支持的系统其实不多，加上目前大端 CPU 早已不是主流，比较现代化的 Linux 中只有 Debian 10 还支持 MIPS 架构，因此我打算使用 debootstrap 工具来构建一个最小化的 Debian rootfs

首先安装工具

```sh
sudo apt install debootstrap debian-keyring
```

使用 debootstrap 构建最小 rootfs

```sh
sudo mkdir /mnt/mips_root
sudo debootstrap --arch=mips buster /mnt/mips_root
```

最后将主机的 /dev 以及 proc 和 sysfs 挂载进 chroot 中

```sh
sudo mount --bind /dev /mnt/mips_root/dev
sudo mount -t proc proc /mnt/mios_root/proc
sudo mount -t sysfs sysfs /mnt/mios_root/sys
```

注意，每次重启后，要进入 chroot 环境，都要记得先挂载以上 Filesystem。

## 配置环境与验证

接下来就可以 chroot 进新环境了，注意使用特权用户进行：

```sh
sudo chroot /mnt/mips_root
```

接下来即可正常使用了，使用起来就像原生的 MIPS 环境一样。但在这之前，我们还需要进行一些系统初始配置。例如安装 locales 配置 i18n。

```sh
apt install locales 
```

接下来就可以安装编译环境了。

```sh
apt install build-essential
```

即可看到我们使用的是原生的 MIPS 编译器

```sh
$ gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc/mips-linux-gnu/8/lto-wrapper
Target: mips-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Debian 8.3.0-6' --with-bugurl=file:///usr/share/doc/gcc-8/README.Bugs --enable-languages=c,ada,c++,go,d,fortran,objc,obj-c++ --prefix=/usr --with-gcc-major-version-only --program-suffix=-8 --program-prefix=mips-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-libitm --disable-libsanitizer --disable-libquadmath --disable-libquadmath-support --enable-plugin --enable-default-pie --with-system-zlib --disable-libphobos --enable-multiarch --disable-werror --enable-multilib --with-arch-32=mips32r2 --with-fp-32=xx --with-lxc1-sxc1=no --enable-targets=all --with-arch-64=mips64r2 --enable-checking=release --build=mips-linux-gnu --host=mips-linux-gnu --target=mips-linux-gnu
Thread model: posix
gcc version 8.3.0 (Debian 8.3.0-6) 
$ file /usr/bin/gcc
/usr/bin/gcc: symbolic link to gcc-8
$ file /usr/bin/gcc-8
/usr/bin/gcc-8: symbolic link to mips-linux-gnu-gcc-8
```

我们对 CS:APP 中的示例程序编译，可以得到一个检验程序。

以上是在 x86_64 系统下的结果

```sh
calling show_twocomp
 39 30
 c7 cf
Calling simple_show_a
 21
 21 43
 21 43 65
Calling simple_show_b
 78
 78 56
 78 56 34
Calling float_eg
For x = 3490593
 21 43 35 00
 84 0c 55 4a
For x = 3510593
 41 91 35 00
 04 45 56 4a
Calling string_ueg
 41 42 43 44 45 46
Calling string_leg
 61 62 63 64 65 66
```

以下则是 MIPS 系统下的结果

```sh
calling show_twocomp
 30 39
 cf c7
Calling simple_show_a
 87
 87 65
 87 65 43
Calling simple_show_b
 12
 12 34
 12 34 56
Calling float_eg
For x = 3490593
 00 35 43 21
 4a 55 0c 84
For x = 3510593
 00 35 91 41
 4a 56 45 04
Calling string_ueg
 41 42 43 44 45 46
Calling string_leg
 61 62 63 64 65 66
```

可以看到数字在内存中的存储顺序是完全不一样的。
