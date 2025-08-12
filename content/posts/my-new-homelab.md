---
title: 我的新 HomeLab —— 移动版
date: 2025-08-12T23:07:00+08:00
categories: ['HomeLab']
tags: ['homelab']
---

我现在把自用的电脑换成了 Thinkbook 14+ 2024 Intel 款。从 2018 年到现在，技术还是有进步的，一些新特性也算是享受到了。能 Type-C 供电，还有 OcuLink 外接显卡。续航也说得过去。整体来说比较满意。

那之前的 XPS 怎么办呢？8 代 i7 性能并不差，32G 内存实际跟我这台新电脑一样大，SSD 虽然被我拆出来装到了新电脑上，但我也又找了一块旧的 512GB SSD 给他装了回去。

从去年开始，我人已经去了日本。明年 4 月要去新地方上学了。（那边的考试体验记之后会写的，再等等）

随着家里的 HomeLab 闲置下来，我需要一个新的主机充当我的 HomeLab。我立马想到用这台旧的 XPS 来做新 HomeLab，最大的好处是，他是移动的，而且自带键盘鼠标显示器，甚至自带 UPS（笔记本电脑的电池何尝不是一种 UPS？）

## 环境

这台电脑除了一个 M.2 SSD 硬盘位之外，还有一个 SATA 硬盘位。我给这个 SATA 盘位加了一块 1T SSD，这就是所有的存储了。因此我也没有打算将其作为 NAS 来存许多东西。

系统毫不犹豫的直接选择用 Ubuntu 24.04。

这次我打算跑 Ceph。肯定会有人问，在单盘单节点上跑 Ceph 根本没有意义啊。你说的没错，但是 Ceph 的接口实在是很通用很好用，改改设置除了报 HEALTH_WARN 之外也并不会有其他明显的副作用。

可以在主机上直接起单节点 K8s，虚拟化部分我还在考虑是否上 OpenStack。

## 一些配置

为了使其能做到移动，网络是主要问题。把电脑拿到其他网络环境后 IP 地址一定会发生变化，这使得许多写死 IP 地址的服务都需要重新配置。尽管用 DNS 取代写死的 IP 或许才是正确操作，但 Ceph 貌似并没有很良好的兼容性，Ceph 在更换网络环境后需要重新配置。因此我认为将网络隔离才是好选择。

在 Linux 上，我们可以起一个 Linux Bridge 来做一个系统的网络接口。

```sh
 sudo nmcli con add type bridge con-name bridge-bridge0 ifname bridge0 ipv4.method manual ipv4.addresses 192.168.200.2/24 autoconnect yes
```

这样一来就可以用这个接口 IP 作为 Ceph 的 mon-ip 来 Bootstrap Ceph CLuster 了。

我使用的是 MicroCeph 发行版。

```sh
sudo snap install microceph

sudo microceph cluster bootstrap --mon-ip 192.168.200.2
sudo microceph disk add /dev/sda --wipe
sudo ceph config set global osd_pool_default_size 1
sudo ceph config set mgr mgr_standby_modules false
sudo ceph config set osd osd_crush_chooseleaf_type 0
```

以上初始化了一个 Ceph 单节点，并且配置 CRUSH MAP 复制单位为 OSD，仅一个副本，并且无需等待写入全部完成。对单节点单副本来说以上配置是完全安全的。

这样以来，运行 `sudo ceph -s` 后应当能看到类似如下情况了。

```sh
  cluster:
    id:     a45eaa55-384e-4ab5-8f33-07695072f6cb
    health: HEALTH_WARN
            1 pool(s) have no replicas configured

  services:
    mon: 1 daemons, quorum home.lab.ddupan.top (age 29m)
    mgr: home.lab.ddupan.top(active, since 29m)
    osd: 1 osds: 1 up (since 29m), 1 in (since 29m)

  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 577 KiB
    usage:   27 MiB used, 224 GiB / 224 GiB avail
    pgs:     1 active+clean
```

那么从自己的电脑如何连接到这台 LAB 里的子网呢，当然两边写写路由在局域网里完全可以直接通。其他协议用来组网也很多，但显然限制在 Windows 这一边，这一边没有那么多选择，许多高级网络功能是需要 Windows Server 才支持的。最简单的办法反而是用 VPN。

所以就起个 Wireguard 吧。

Linux 这边，生成 Wireguard Key 需要 Wireguard 工具组，但 Wireguard 接口组是内核支持，理论上你在网上找个配置生成工具啥的都可以拿来用。

```sh
sudo apt install wireguard

wg genkey | sudo tee /etc/wireguard/$HOSTNAME.private.key | wg pubkey | sudo tee /etc/wireguard/$HOSTNAME.public.key

sudo chmod 600 /etc/wireguard/*
```

用 netplan 起一个接口

```yaml
network:
  tunnels:
    wg0:
      mode: wireguard
      port: 51820
      key: omited...
      addresses:
        - 10.0.0.1/24
      peers:
        - allowed-ips: [10.0.0.2/24]
          keys:
            public: omited...
```

在 Windows 这边安装 Wireguard 之后，直接新建隧道。

```ini
[Interface]
PrivateKey = omited
ListenPort = 51820
Address = 10.0.0.2/24

[Peer]
PublicKey = omited
AllowedIPs = 192.168.200.0/24, 10.0.0.1/24
Endpoint = 192.168.50.28:51820
```

别忘了在 Linux 这边开启路由内核选项。这样一来应该就连的通了。
