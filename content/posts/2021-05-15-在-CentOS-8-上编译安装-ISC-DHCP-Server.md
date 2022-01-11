---
title: 在 CentOS 8 上编译安装 ISC-DHCP-Server
date: 2021-05-15
categories: [Linux]
tags: [linux]
---

应要求，我需要在 Red Hat Enterprise Linux 8 上编译安装 ISC-DHCP，但我当时手头只有 CentOS 8 的镜像，根据 CentOS 8 = RHEL 8 的传统，我在 CentOS 8 上写了这篇文，大约在 RHEL 上也完全适用。

Red Hat 给的 dhcpd 包版本是 4.3.6，而最新的源码包是 4.4.2，但实际上是由 Red Hat 负责 Backport 的，而且这东西也不会像 Kubernetes 这种东西几天一小更，几周一大更，版本号刷的贼快，所以完全没必要自己编译，我只是被有这个需求罢了。

姑且还是记录在这里。

## 编译

首先配置编译环境

```sh
dnf install gcc make automake
```

不需要别的东西

解压源码包，进入目录，正常编译

```sh
tar -xvf dhcp-4.4.2.tar.gz
cd dhcp-4.4.2/
./configure
make
make -C server install
```

最后一个 `make install` 只安装了 ISC-DHCP-Server，实际上这个包内还有 `DHCP relay` 和 `dhclient`，上面的 `make` 已经将所有三者全部编译了.

这个 `makefile` 会把编译好的 `dhcpd` 二进制文件安装到 `/usr/local/sbin/dhcpd`，如果你想，也可以把它拷贝到 `/usr/sbin/` 或者 `/sbin`，但我就不打算拷贝了。

现在直接在 Shell 里执行 `dhcpd` 应该就已经能运行了，但是由于没有找到配置文件，以及没有保存租约的数据库文件，服务依然启动不起来，而且需要通过 Shell 来运行 dhcpd 也非常不优雅，因此我们还有两个工作要做，一个是准备配置文件，准备租约数据库文件，以及准备 `dhcpd.servive` 以便可以使用 systemd 来管理 `dhcpd` 守护进程

## 准备配置文件与租约数据库

dhcpd 源码包中确实给了示例，但是给的却是 DHCPv6 的，我对 DHCPv6 现在可没什么兴趣，所以我从别处扒了一个，直接贴下面了

```sh
$ cat > /etc/dhcpd.conf << EOF
# Begin /etc/dhcpd.conf
#
# Example dhcpd.conf(5)
# Use this to enable / disable dynamic dns updates globally.
ddns-update-style none;
# option definitions common to all supported networks...
option domain-name "example.org";
option domain-name-servers ns1.example.org, ns2.example.org;
default-lease-time 600;
max-lease-time 7200;
# This is a very basic subnet declaration.
subnet 10.254.239.0 netmask 255.255.255.224 {
range 10.254.239.10 10.254.239.20;
option routers rtr-239-0-1.example.org, rtr-239-0-2.example.org;
}
# End /etc/dhcpd.conf
EOF
```

如果没有额外指定，那么默认编译选项的 dhcpd 会将 /etc/dhcpd.conf 作为配置文件，如果想修改，如修改为 `/etc/dhcpd/dhcpd.conf`，则需要编译时添加参数选项后重新编译

租约数据库文件默认为 `/var/db/dhcpd.leases`，同样也可以在编译时通过参数修改

```sh
install -v -dm 755 /var/db
touch > /var/db/dhcpd.leases
chmod 644 /var/db/dhcpd.leases
```

## 准备 systemd service

为了最大程度贴近 CentOS 发行时的样子，我从 rpm 包里扒出来了这个文件

```ini
# /usr/lib/systemd/system/dhcpd.service
[Unit]
Description=DHCPv4 Server Daemon
Documentation=man:dhcpd(8) man:dhcpd.conf(5)
Wants=network-online.target
After=network-online.target
After=time-sync.target

[Service]
Type=notify
EnvironmentFile=-/etc/sysconfig/dhcpd
ExecStart=/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid $DHCPDARGS
StandardError=null

[Install]
WantedBy=multi-user.target
```

默认配置编译时不支持 `-user` 和 `-group` 选项，因此这个文件依然需要一些修改才能工作

根据这个文件，我们需要做下面几件事：

- 修改这个 service 文件使其匹配现有环境
- 因这个配置文件的历史遗留原因需创建 `/etc/sysconfig/dhcpd`

```sh
touch /etc/sysconfig/dhcpd
```

这个 `/etc/sysconfig/dhcpd` 是过去用来修改参数的方式，但当前版本已经不需要自行修改监听网卡，因此这个文件也没用了，原文件中只有注释罢了。

修改后的 dhcpd.service 如下

```ini
# /usr/lib/systemd/system/dhcpd.service
[Unit]
Description=DHCPv4 Server Daemon
Documentation=man:dhcpd(8) man:dhcpd.conf(5)
Wants=network-online.target
After=network-online.target
After=time-sync.target

[Service]
EnvironmentFile=-/etc/sysconfig/dhcpd
ExecStart=/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -no-pid $DHCPDARGS
StandardError=null

[Install]
WantedBy=multi-user.target
```

由于没有配置 SELinux 安全上下文导致 dhcpd 可能无法读写 `dhcpd.leases`，关闭 SELinux 能解决一些奇怪的权限问题

现在只要正常修改 `/etc/dhcpd.conf` 应该就可以正常启动服务了

## 尽量贴近发行版配置

CentOS 的 dhcpd 编译选项我会尽量猜测后放在下面，可以对照使用

```sh
( export CFLAGS="${CFLAGS:--g -O2} -Wall -fno-strict-aliasing                 \
        -D_PATH_DHCPD_CONF='\"/etc/dhcp/dhcpd.conf\"' "             && 

./configure --prefix=/usr                                           \
            --sysconfdir=/etc/dhcp                                  \
            --localstatedir=/var                                    \
            --with-srv-lease-file=/var/lib/dhcpd/dhcpd.leases       \
            --enable-paranoia                                       \
            --with-srv6-lease-file=/var/lib/dhcpd/dhcpd6.leases     \
            --with-systemd                                          \
            --disable-static                                        \
            --enable-log-pid
) &&
make -j1
make -C server install
install -v -dm 755 /var/lib/dhcpd
touch /var/lib/dhcpd/dhcpd.leases
touch /etc/sysconfig/dhcpd
useradd dhcpd -r -s /sbin/nologin
```

Red Hat 对源码做了大改，对源码打了几十个补丁，并且实现了支持 systemd-notify 的功能，理论上可以从 CentOS 的源码包中提取出 Patch34 即 `dhcp-sd_notify.patch` 并打上，再编译应出现 `--with-systemd` 选项

> 参考:
>
> [Beyond Linux® From Scratch (systemd Edition) - Version 10.1 - Chapter 14. Connecting to a Network - DHCP 4.4.2](https://www.linuxfromscratch.org/blfs/view/stable-systemd/basicnet/dhcp.html)
>
> [dhcp-4.3.6-41.el8.src.rpm](https://vault.centos.org/8.3.2011/BaseOS/Source/SPackages/dhcp-4.3.6-41.el8.src.rpm)
