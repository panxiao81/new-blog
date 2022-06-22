---
title:  不同版本的 lscpu 的不同输出
date: 2022-06-23T03:03:00+08:00
categories: [Linux]
tags: [linux]
---

我发现了 lscpu 在旧版和新版的不同输出行为，为了验证我的猜想，我重新编译了 util-linux，这篇文章是对整个过程的记录

<!--more-->

我近期开始学习 RHCA 课程，首门课程是 RH442。

我平时很少使用 Red Hat 及其衍生发行版，平时更是 Ubuntu 和 Debian 系最为常用。首先在课上发现了 lscpu 的输出与手边的 Ubuntu 22.04 和 Debian 11 大不相同。

这个问题尤其集中在缓存的大小显示上。

例如，在我笔记本的虚拟机中，AlmaLinux 8 显示的内容如下

```sh
$ lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              2
On-line CPU(s) list: 0,1
Thread(s) per core:  1
Core(s) per socket:  1
Socket(s):           2
NUMA node(s):        1
Vendor ID:           GenuineIntel
CPU family:          6
Model:               158
Model name:          Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
Stepping:            10
CPU MHz:             2208.000
BogoMIPS:            4416.00
Hypervisor vendor:   VMware
Virtualization type: full
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            9216K
NUMA node0 CPU(s):   0,1
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc cpuid pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single pti ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap clflushopt xsaveopt xsavec xgetbv1 xsaves arat md_clear flush_l1d arch_capabilities
```

注意此处的 L1 缓存和 L2 缓存为 32K+32K 以及 256K

而同样此电脑上的 Ubuntu 显示如下：

```sh
$ lscpu
Architecture:            x86_64
  CPU op-mode(s):        32-bit, 64-bit
  Address sizes:         45 bits physical, 48 bits virtual
  Byte Order:            Little Endian
CPU(s):                  2
  On-line CPU(s) list:   0,1
Vendor ID:               GenuineIntel
  Model name:            Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
    CPU family:          6
    Model:               158
    Thread(s) per core:  1
    Core(s) per socket:  1
    Socket(s):           2
    Stepping:            10
    BogoMIPS:            4416.00
    Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_t
                         sc cpuid tsc_known_freq pni pclmulqdq vmx ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault inv
                         pcid_single pti ssbd ibrs ibpb stibp tpr_shadow vnmi ept vpid fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap clflushopt xsaveopt xsavec xgetbv1 xsaves arat md_clear flush_l1d a
                         rch_capabilities
Virtualization features: 
  Virtualization:        VT-x
  Hypervisor vendor:     VMware
  Virtualization type:   full
Caches (sum of all):     
  L1d:                   64 KiB (2 instances)
  L1i:                   64 KiB (2 instances)
  L2:                    512 KiB (2 instances)
  L3:                    18 MiB (2 instances)
NUMA:                    
  NUMA node(s):          1
  NUMA node0 CPU(s):     0,1
Vulnerabilities:         
  Itlb multihit:         KVM: Mitigation: VMX disabled
  L1tf:                  Mitigation; PTE Inversion; VMX flush not necessary, SMT disabled
  Mds:                   Mitigation; Clear CPU buffers; SMT Host state unknown
  Meltdown:              Mitigation; PTI
  Mmio stale data:       Mitigation; Clear CPU buffers; SMT Host state unknown
  Spec store bypass:     Mitigation; Speculative Store Bypass disabled via prctl and seccomp
  Spectre v1:            Mitigation; usercopy/swapgs barriers and __user pointer sanitization
  Spectre v2:            Mitigation; Retpolines, IBPB conditional, IBRS_FW, STIBP disabled, RSB filling
  Srbds:                 Unknown: Dependent on hypervisor status
  Tsx async abort:       Not affected
```

可以看到，不但显示的内容多了许多，并且注意此处的 Cache 值，他是所有的核心缓存加在一起的值。

更加让人迷惑的，是 Debian 11 附带的 lscpu，他的显示效果如下。

```sh
$ lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              2
On-line CPU(s) list: 0,1
Thread(s) per core:  1
Core(s) per socket:  1
Socket(s):           2
NUMA node(s):        1
Vendor ID:           GenuineIntel
CPU family:          6
Model:               158
Model name:          Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
Stepping:            10
CPU MHz:             2208.000
BogoMIPS:            4416.00
Hypervisor vendor:   VMware
Virtualization type: full
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            9216K
NUMA node0 CPU(s):   0,1
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc cpuid pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single pti ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap clflushopt xsaveopt xsavec xgetbv1 xsaves arat md_clear flush_l1d arch_capabilities
```

此时表面上看起来，他和 AlmaLinux 中的结果差距不大，但此时的缓存是以所有缓存量相加在一起计算的，且不像 Ubuntu 22.04，他并没有注明是多个 instences 的值。这就更令人迷惑了。

我一开始认为，既然两种发行版的 lscpu 都来源于上游的 util-linux 包，他们不应该有这么大的差别，马上想到的是 Red Hat 这个 backport 狂魔用补丁改了行为。

因此我去翻看了来源 [CentOS Stream 8 的 RPM 源码树](https://gitlab.com/redhat/centos-stream/rpms/util-linux/-/tree/c8s)，尽管确实有些与 lscpu 相关的 patch，但并不存在修改输出这种 patch。

从 util-linux 的编译选项中也看不出什么会影响输出的区别。

因此，只能猜测是由于版本不同，上游的包发生了 break change。

并且，三个系统的 util-linux 包版本确实各不相同：

```sh
# AlmaLinux 8
lscpu from util-linux 2.32.1
# Debian 11
lscpu from util-linux 2.36.1
# Ubuntu 22.04
lscpu from util-linux 2.37.2
```

为了验证猜测，我决定干脆来试试。

我选择只编译 2.32.1 和 2.36.1 两个版本。

util-linux 可以从 [kernel.org](https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/) 下载到上游打包好的源码包。

从网站上下载到 [util-linux-2.32.1.tar.gz](https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/v2.32/util-linux-2.32.1.tar.gz) 和 [util-linux-2.36.1.tar.xz](https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/v2.36/util-linux-2.36.1.tar.xz) 两个源码包并解压。

为了控制变量，我选择了在 AlmaLinux 环境下，使用 AlmaLinux RPM 包中的 configure 选项编译，并且不改变 CFLAGS 的值。

首先安装编译依赖。

由于编译依赖有许多，为了不影响我的环境，我选择使用一个 Docker 容器来编译。

```sh
docker run -it -v `pwd`:/usr/src almalinux:8 bash

# 进入容器内
cd /usr/src
```

在安装依赖前，其中两个包需要启用 PowerTools 源

```sh
dnf config-manager --set-enabled powertools
```

安装编译依赖，这些依赖都是从 SRPM 的 SPEC 文件中得来的。

```sh
dnf install audit-libs-devel gettext-devel libselinux-devel ncurses-devel pam-devel zlib-devel popt-devel libutempter-devel systemd-devel systemd libuser-devel libcap-ng-devel python3-devel gcc automake autoconf libtool bison git-core make diffutils
```

接下来完成编译

```sh
cd util-linux-2.32.1
./autogen.sh

./configure \
	--disable-assert \
	--with-systemdsystemunitdir=/usr/lib/systemd/system \
	--disable-silent-rules \
	--disable-bfs \
	--disable-pg \
	--enable-chfn-chsh \
	--enable-usrdir-path \
	--enable-write \
	--enable-raw \
	--with-python=3.6 \
	--with-systemd \
	--with-udev \
	--with-selinux \
	--with-audit \
	--with-utempter \
	--disable-makeinstall-chown

make
```

2.36 版也使用相同方式编译。

编译之后，就可以在源码根目录得到我们所需要的 `lscpu` 二进制文件。在 AlmaLinux 环境中运行这两个版本的文件，得到了确实不同的结果。

```sh
$ cat /etc/redhat-release
AlmaLinux release 8.6 (Sky Tiger)
$ ./util-linux-2.32.1/lscpu --version
lt-lscpu from util-linux 2.32.1
$ ./util-linux-2.32.1/lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              2
On-line CPU(s) list: 0,1
Thread(s) per core:  1
Core(s) per socket:  1
Socket(s):           2
NUMA node(s):        1
Vendor ID:           GenuineIntel
CPU family:          6
Model:               158
Model name:          Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
Stepping:            10
CPU MHz:             2208.000
BogoMIPS:            4416.00
Hypervisor vendor:   VMware
Virtualization type: full
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            9216K
NUMA node0 CPU(s):   0,1
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc cpuid pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single pti ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap clflushopt xsaveopt xsavec xgetbv1 xsaves arat md_clear flush_l1d arch_capabilities
$ ./util-linux-2.36.1/lscpu --version
lscpu from util-linux 2.36.1
$ ./util-linux-2.36.1/lscpu
Architecture:                    x86_64
CPU op-mode(s):                  32-bit, 64-bit
Byte Order:                      Little Endian
Address sizes:                   45 bits physical, 48 bits virtual
CPU(s):                          2
On-line CPU(s) list:             0,1
Thread(s) per core:              1
Core(s) per socket:              1
Socket(s):                       2
NUMA node(s):                    1
Vendor ID:                       GenuineIntel
CPU family:                      6
Model:                           158
Model name:                      Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
Stepping:                        10
CPU MHz:                         2208.000
BogoMIPS:                        4416.00
Hypervisor vendor:               VMware
Virtualization type:             full
L1d cache:                       64 KiB
L1i cache:                       64 KiB
L2 cache:                        512 KiB
L3 cache:                        18 MiB
NUMA node0 CPU(s):               0,1
Vulnerability Itlb multihit:     KVM: Mitigation: VMX unsupported
Vulnerability L1tf:              Mitigation; PTE Inversion
Vulnerability Mds:               Mitigation; Clear CPU buffers; SMT Host state unknown
Vulnerability Meltdown:          Mitigation; PTI
Vulnerability Spec store bypass: Mitigation; Speculative Store Bypass disabled via prctl and seccomp
Vulnerability Spectre v1:        Mitigation; usercopy/swapgs barriers and __user pointer sanitization
Vulnerability Spectre v2:        Mitigation; Retpolines, IBPB conditional, IBRS_FW, STIBP disabled, RSB filling
Vulnerability Srbds:             Unknown: Dependent on hypervisor status
Vulnerability Tsx async abort:   Not affected
Flags:                           fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc cpuid pni pclmulqdq ssse3 fma cx16 pcid s
                                 se4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single pti ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap clflus
                                 hopt xsaveopt xsavec xgetbv1 xsaves arat md_clear flush_l1d arch_capabilities
```

这样一来，基本可以确定，这个不同的行为就是 util-linux 升级版本产生的 break change.

在翻 RPM 的时候我发现 RHEL 9 的 util-linux 的上游源码使用的是 2.37.2 版本，即与 Ubuntu 22.04 使用的版本一致，因此以后的 RHEL 也许也将会看到类似的结果了。
