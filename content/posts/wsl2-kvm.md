---
title: "在 WSL2 上获得近乎完整的 Linux 体验"
date: 2021-06-19T22:19:32+08:00
categories: [
    "Linux"
]
tags: [
    linux
]
description: 在 WSL 2 上配置 KVM 虚拟化的方法
draft: false
---

WSL 2 支持 KVM 虚拟化了，配合 WSLg 是相当不错的虚拟化解决方案了

<!--more-->

众所周知，WSL2 是好东西，但毕竟还不是完整的 Linux，比如没有 systemd，一些内核功能不正常等。

并且打开 WSL2 后更大的问题是 VMware 等虚拟机软件无法再打开嵌套虚拟化，Hyper-V 只能说不太好用。

因此我们的目标是，在 WSL2 上使用 systemd，并且打开 KVM，甚至让其运行 libvirtd，最终可以使用 virt-manager 管理虚拟机。

要进行这些操作，需要重新编译内核，配置 WSL2，启用嵌套虚拟化以启用 KVM。

我们需要：

- Windows 10 19645 及以上版本 ( Insider Dev 通道 )
- Intel CPU，AMD 未测试，大概率尚不支持
- 如非最新 Dev 通道版本，需 X Server 来转发图形界面，或使用 WSLg
- WSL 2 
- WSL 2 中安装某一发行版 ( 本文使用 Ubuntu 20.04 LTS )

实测需要高版本 Windows 10 才支持 WSL 2 中的 KVM，如果当前还在使用 21H1，则需要打开 Insider 测试，选择 Dev 通道并升级，截至写文时目前最新版为 21390

升级部分不再表述

## 编译内核

使用 KVM 需要内核功能支持，我们需要自行编译内核加入 KVM 支持。

### 升级系统，安装编译环境

```sh
sudo apt update && sudo apt -y upgrade
sudo apt -y install build-essential libncurses-dev bison flex libssl-dev libelf-dev cpu-checker qemu-kvm libvirtd virt-manager git
```

### 获得内核源码

可以使用 Linux 官方源码，但我使用了微软专门为 WSL 2 修改后的源码。

需要访问 Github

```sh
git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
cd WSL2-Linux-Kernel
git checkout linux-msft-wsl-5.10.y
```

我使用了 5.10 版本内核，可以根据自己的需要切换到其他分支

使用当前系统的 config，修改后编译内核

```sh
zcat /proc/config.gz > .config
make menuconfig
```

在 `Virtualization` 中确认 Intel 支持已被选中，`virtio-net` 也被选中 ( 如果有 )

按两次 ESC，再导航至 `Processor type and features -> Linux guest support`，确定 `KVM Guest support` 已被选中

可以编译成模块再加载，但我选择直接编译进内核，同时可以编译 Multipath 依赖的支持，以便后续 systemd 可以少一个报错

保存并退出

### 编译

```sh
make -j 8
```

等吧

### 安装新内核

将编译好的内核拷贝到当前用户的家目录 ( Windows
这边！) 并且配置使用新内核

```sh
cp arch/x86/boot/bzImage /mnt/c/Users/<username>/bzImage
vim /mnt/c/Users/<username>/.wslconfig
```

不会用 vim 真的别强求，用 nano 就好了

注意将 <username> 替换成自己的用户名

将以下内容粘贴，并修改 <username> 为自己的用户名

```ini
[wsl2]
nestedVirtualization=true
kernel=C:\\Users\\<username>\\bzImage
```

### 重新启动 WSL 2

打开 Powershell

```sh
wsl --shutdown Ubuntu
```

根据自己的发行版做调整

打开新的 WSL 窗口即可。

### 验证

```sh
uname -ar
```

查看内核日期，应是刚才编译的日期

```sh
kvm-ok
```

应该提示支持

## 配置 KVM-Intel

编辑配置文件 `.wslconfig` 加入

```ini
kernelCommandLine=intel_iommu=on iommu=pt kvm.ignore_msrs=1 kvm-intel.nested=1 kvm-intel.ept=1 kvm-intel.emulate_invalid_guest_state=0 kvm-intel.enable_shadow_vmcs=1 kvm-intel.enable_apicv=1
```

再次重启 WSL2

## 在 WSL2 中启用 systemd

过去有脚本可以替换 init 为 systemd，但现在不再推荐使用。

现在推荐使用 genie 工具，他将在用户态创建 systemd 进程

首先要安装 dotnet runtime，因 genie 使用 .net core

```sh
wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get install -y dotnet-runtime-5.0
```

此为 Ubuntu 20.04，其他版本看文档

[Instll Documentations](https://docs.microsoft.com/en-us/dotnet/core/install/linux-ubuntu#2004-)

加入源

```sh
sudo apt install apt-transport-https

sudo wget -O /etc/apt/trusted.gpg.d/wsl-transdebian.gpg https://arkane-systems.github.io/wsl-transdebian/apt/wsl-transdebian.gpg

sudo chmod a+r /etc/apt/trusted.gpg.d/wsl-transdebian.gpg

sudo cat << EOF > /etc/apt/sources.list.d/wsl-transdebian.list
deb https://arkane-systems.github.io/wsl-transdebian/apt/ $(lsb_release -cs) main
deb-src https://arkane-systems.github.io/wsl-transdebian/apt/ $(lsb_release -cs) main
EOF

sudo apt update
sudo apt -y install systemd-genie
```

先解决一些问题

```sh
sudo ssh-keygen -A
sudo echo '' > /etc/fstab
```

运行：

```sh
genie -s
```

一些 systemd 模块无法加载。直接禁用即可

```sh
sudo systemctl disable systemd-modules-load
sudo systemctl disable multipath
```

现在可以启用 sshd 了

```sh
sed -i s/PasswordAuthentication no/PasswordAuthentication yes/g /etc/ssh/sshd_config
sudo systemctl start ssh
```

顺便启用 libvirtd

```sh
sudo systemctl start libvirtd
```

如果使用 WSLg，在开始菜单里找到 `虚拟系统管理器` 打开使用即可

目前 KVM 可以正常使用，可以安装 CentOS，Ubuntu 等，也可以直接在 WSL 2 中安装 Docker 而不使用 Docker Desktop

> 但折腾一圈的目的是啥？ 好像不如我直接去用 Linux 算了？
