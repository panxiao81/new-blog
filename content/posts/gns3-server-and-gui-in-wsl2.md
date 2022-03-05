---
title: "在 WSL 2 中运行 GNS3"
date: 2022-03-05T23:34:32+08:00
categories: [
    "Linux"
]
tags: [
    linux
]
draft: false
---

当你需要一个带有网络拓扑模拟器的虚拟机环境时，为什么不考虑一下把 GNS3 装进 Windows 的 WSL 2 中呢？

<!--more-->

我在以前的文章中曾经提过如何在 WSL 2 上开启 KVM 虚拟化，随着时间流逝，Windows 11 的发布，让整个配置过程又简单了不少，我们已经不需要像过去那样手动编译内核，只要修改配置文件就能开启 WSL 2 中的嵌套虚拟化。

那么，比起我需要一个额外的虚拟机软件运行 GNS 3，为什么我们不能直接把 GNS3 运行在 WSL 2 中呢？实际测试过发现是完全可行的。

## 配置 WSL 2 开启 KVM

现在已经不需要自行编译内核了，直接编辑 `/mnt/c/Users/<username>/.wslconfig`

```ini
[wsl2]
nestedVirtualization=true
```

打开 Powershell，升级 WSL

```bat
wsl --update
```

接下来打开 WSL 后，发现 `/dev/kvm` 权限不正确，根据 WSL 项目中 [#7149](https://github.com/microsoft/WSL/issues/7149) 所述，由于 WSL 没有 udev，设备文件相当于手动创建的，一个解决方法是加入 `boot` 设置，使 WSL 每次启动自动更改权限

编辑 `/etc/wsl.conf` 在末尾加入：

```ini
[boot]
command = /bin/bash -c 'chown root:kvm /dev/kvm && chmod 660 /dev/kvm'
```

重启 WSL 2。

将当前用户加入 kvm 组。

```sh
sudo usermod -aG kvm $(id -un)
```

如果需要使用 Docker 的话可在 Windows 中安装 Docker Desktop，使用 WSL 后端并启用发行版集成。

确认 `/dev/kvm` 权限无误后，即可继续下一步。

```sh
$ ls -la /dev/kvm
crw-rw---- 1 root kvm 10, 232 Mar  5 23:24 /dev/kvm
```

## 安装 GNS3

安装 GNS3 可以直接到 GNS3 官网按照文档安装即可。

```sh
sudo add-apt-repository ppa:gns3/ppa
sudo apt update                                
sudo apt install gns3-gui gns3-server
```

安装一些桌面应用程序

```sh
sudo apt install nautilus firefox
```

可以运行一下 firefox 确认一下 `wslg` 是否按预期工作。

无误后，输入 `gns3` 即可运行。

## 配置 GNS3 使用 Windows Terminal

这个在 WSL 2 里运行的 GNS3 也可以使用 Windows Terminal。

打开终端程序配置，将终端程序设为自定义，后输入：

```sh
wt.exe -w 1 --title %d telnet %h %p
```

Windows 的 PATH 环境变量已经和 WSL 共享，所以完全不成问题。

可以搞一个 FRR 快速测试一下。

![图 1](/images/other/894558071bb5db94d4dabcf6b95d8b0718598710120252f00cc00f386a19cba1.png)  

WSL 2 万岁！
