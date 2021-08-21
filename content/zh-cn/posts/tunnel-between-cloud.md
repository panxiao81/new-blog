---
title: "VPN+BGP 打通家中网络与公有云"
date: 2021-08-21T19:47:26+08:00
draft: false
categories: [
    "Linux"
    ,网络
]
tags: [
    linux,
    network
]
---


之前说好的文终于有时间写了。

记录一下打通私有云与多公有云的过程吧。

众所周知，公有云即便提供 VPN 网关价格也是相当的贵，那么有没有便宜的方法呢？答案当然是有的，使用普通的 ECS 解决即可。

<!--more-->

公有云的环境我不想打乱重做给大家看，因此我用 GNS3 重新搭了一个 Lab，尽量给大家看完整的原始方案。
 
![Lab Topology](/images/other/bd906744f7029629da5b5c1a63fc7ff9619b8fe8d8d22accbde29d80cc14fafc.png)

以上为 Lab 拓扑

Internet 部分给提供 Lab 内部提供 Internet 访问与 DNS 服务，以便内部虚拟机可使用网络 Linux 软件源。

上半部分标记 Aliyun 即国内阿里云中转机，与家中私有云使用 OpenVPN 对接。

右下部分为美国某提供商的廉价 VPS，对外提供服务的反向代理设置在这里。

左下为私有云，实际的环境使用的是 VMware vSphere 进行，可参考以前的家庭网络改造一文，此处将其抽象为单台路由器，其中三个环回口分别代表三个网络,Toolbox 用于测试。

## WAN 配置

此处的 WAN 路由用于连接三个站点，外加提供 Internet 访问所需的 NAT。

### 路由器配置

路由器使用 Cisco vIOS 实现。

配置中的 IP 地址已隐去

```
! 连接 Home
interface GigabitEthernet0/0
 ip address 123.186.XXX.1 255.255.255.0
 ip nat inside
```

```
! 连接 Aliyun
interface GigabitEthernet0/1
 ip address 121.43.XXX.1 255.255.255.0
 ip nat inside
```

```
! 连接 US
interface GigabitEthernet0/2
 ip address 80.251.XXX.1 255.255.255.0
 ip nat inside
```

```
! 此接口连接 NAT，使用 DHCP 获取 IP
interface GigabitEthernet0/3
 ip address dhcp
 ip nat outside
```

```
! 配置 NAT 转换，配置 Gi0/3 接口上的 NAT overload
ip nat inside source list 1 interface GigabitEthernet0/3 overload
access-list 1 permit any
```

### DNS

这个 DNS 为 Lab 内部提供 DNS 与 DNS 转发服务

GNS3 有打包好的 DNS Docker 镜像，使用 dnsmasq 实现。

将 DNS 条目写入 /etc/hosts 中即可

网络部分我将其桥接到 NAT 网络中，因此同样使用 192.168.122.0/24 网络

## Home 部分配置

实际上我的家中网络使用的是 pfSense 作为出口路由+防火墙，因此 OpenVPN 与 BGP Session 均运行在此处，实验环境中为简化配置，使用了 VyOS 替代了 pfSense。

配置出口：

```sh
set interfaces ethernet eth0 address '123.186.XXX.22/24'
set protocols static route 0.0.0.0/0 next-hop 123.186.228.1
set nat source rule 1 outbound-interface 'eth0'
set nat source rule 1 source address '0.0.0.0/0'
set nat source rule 1 translation address 'masquerade'
```

配置环回口：

虽然可以配置 lo 口，但是并不建议。

Linux 提供了与环回口实际类似的 `dummy` 接口，与 Cisco 等路由器的 loopback 接口实质一致。

```sh
set interfaces dummy dum1 address '192.168.1.1/24'
set interfaces dummy dum2 address '192.168.50.1/24'
set interfaces dummy dum3 address '192.168.100.1/24'
```

VyOS 并没有类似于 `ip unnunbered` 的功能，但允许在多个接口配置相同 IP

```sh
set interfaces ethernet eth1 address '192.168.1.1/24'
```

配置 OpenVPN Site-to-Site

生成 OpenVPN Static Key

```sh
# 在非配置模式运行
generate openvpn key openvpn.key
```

配置 OpenVPN:

```sh
set interfaces openvpn vtun0 encryption cipher 'aes256' # 加密为 AES-256-CBC
set interfaces openvpn vtun0 hash 'sha256' 
set interfaces openvpn vtun0 local-address 10.0.9.1 # 本端隧道 IP 地址
set interfaces openvpn vtun0 local-port '1194' # 本端端口
set interfaces openvpn vtun0 mode 'site-to-site' # 设置 VPN 类型
set interfaces openvpn vtun0 persistent-tunnel # 打开隧道
set interfaces openvpn vtun0 protocol 'udp' # 使用 UDP 连接
set interfaces openvpn vtun0 remote-address '10.0.9.2' # 对端地址
set interfaces openvpn vtun0 remote-host '121.43.XXX.81' # 对端 IP
set interfaces openvpn vtun0 remote-port '1194' # 对端端口
set interfaces openvpn vtun0 shared-secret-key-file '/config/auth/openvpn.key' # 上一步生成的静态 Key
```

一些普通的系统配置

```sh
set system host-name 'home'
set system name-server '192.168.122.23'
```

## Aliyun 配置

阿里云是非常典型的大型公有云，其 ECS 具有私有网络，其浮动 IP 有一层 NAT，此处使用路由器模拟环境。

### CN 路由器配置

依然使用 Cisco viOS

配置内外接口，以及静态 NAT

```
interface GigabitEthernet0/0
 ip address 121.43.XXX.2 255.255.255.0
 ip nat outside
!
interface GigabitEthernet0/1
 ip address 172.19.0.1 255.255.192.0
 ip nat inside
!
ip route 0.0.0.0 0.0.0.0 121.43.117.1
ip nat inside source static 172.19.42.8 121.43.XXX.81
```

内网地址为 172.19.0.0/18，配置首地址为网关，实际可随意。

### ECS 配置

我在云上实际运行的是 Ubuntu 20.04 LTS，但此处使用的是 Ubuntu 18.04 LTS，区别不大

云上的 ECS 显然网络配置是不需要修改的，仅需加载新网络配置即可，但我这里需要手动配置一下。

使用的 Ubuntu 是带 GNOME 的，所以使用 NetworkManager 配置较方便。

```sh
nmcli connection delete Wired\ Connection\ 1
nmcli connection add 
    type ethernet \
    ifname ens3 \
    con-name ens3 \
    ipv4.method manual \
    ipv4.addresses 172.19.42.8/18 \
    ipv4.gateway 172.19.0.1 \
    ipv4.dns 192.168.122.23 \
    autoconnect yes
nmcli connection up ens3
```

测试 Internet 与 Home 站点连通性后，配置 OpenVPN

```sh
sudo apt install openvpn
sudo vim /etc/openvpn/home.conf
```

pfSense 支持配置文件导出，非常方便。

但这里只能手写了。

写入以下内容

```sh
# /etc/openvpn/home.conf

dev tun
persist-tun
persist-key
cipher aes-256-cbc
auth SHA256
proto udp
remote home.ddupan.top 1194 udp4
ifconfig 10.0.9.2 10.0.9.1
<secret>
# Hidden secret
</secret>
```

其实可以参考 VyOS 中实际 OpenVPN 的配置文件，其位置为 `/run/openvpn/vtun0.conf`

是这样找到的

```sh
ps -aux | grep openvpn | grep -v grep
find / -name vtun0.conf -type f 2> /dev/null
```

启动 VPN，测试连通性即可

```sh
sudo systemctl start openvpn@home
sudo systemctl enable openvpn@home
```

GRE over IPSec 与 US 的连接稍后讨论

## US 配置

我的 US 为某小 VPS 提供商，就是普通的 KVM 虚拟化提供的虚拟机。

公网 IP 是直接配置在 VPS 上的，因此在此拓扑中我也将主机直连在 WAN 路由上。

VPS 运行的是 CentOS 7.9，此处也使用 CentOS 7.9

GNS3 的镜像也是带有桌面的，虽然不带桌面也使用 NetworkManager 管理网络就是了（

```sh
nmcli connection delete Wired\ Connection\ 1
nmcli connection add \
    type ethernet \
    ifname eth0 \
    con-name eth0 \
    ipv4.method manual \
    ipv4.addresses 80.251.XXX.33/24 \
    ipv4.gateway 80.251.XXX.1/24 \
    ipv4.dns 192.168.122.23 \
    autoconnect yes
nmcli connection up eth0
```

有时 CentOS 会有自带的 `virbr0`，虽然在现网中完全没影响，但在这个测试环境中会导致访问不了 DNS，要将此网桥关掉

```sh
nmcli connection down virbr0
nmcli connection modify virbr0 autoconnect no
```

基础配置都完成了，接下来讨论 GRE over IPSec 隧道与 BGP 路由

## GRE over IPSec

GRE over IPSec 经常在数通教材中作为典型的低成本隧道 VPN 案例讲到。

但在 Linux 场景下，GRE over IPSec 因其配置复杂，并不受 Linux 运维人员的青睐，但其作为几乎所有网络设备均支持的隧道方案，仍非常有其研究的价值。

本案例为两台 Linux 间启用 GRE over IPSec，并通过 BGP 管理路由，即 Route-Based IPSec VPN

这一节讨论如何配置 GRE over IPSec，下一节讨论 BGP

很巧的是，两个站点使用了不同的发行版，正好覆盖了 Red Hat 系与 Debian 系两种情况

### CN 站点配置

此站点运行 Ubuntu，Debian 系基本通用

安装 IPSec 功能，Linux 运行 IPSec 使用最多的为 `strongswan`

```sh
sudo apt install strongswan
```

编辑 IPSec 配置，禁止 `strongswan` 管理路由，后续将使用 BGP 手动管理路由

```sh
sudo vim /etc/strongswan.d/charon.conf
```

修改其中配置

```sh
# /etc/strongswan.d/charon.conf

install_routes = no
```

创建 `ipsec-notify.sh` 脚本，用于创建隧道

```sh
# /etc/strongswan.d/ipsec-notify.sh

#!/bin/sh
ip tunnel add gre1 mode gre local 172.19.42.8 remote 80.251.XXX.33
ip link set gre1 up
ip addr add 10.0.0.2/30 dev gre1
ip link set dev gre1 mtu 1400 # IPv4 GRE + IPSecV4 transport
```

关于 MTU 设置的问题，Cisco 文档讲的很详细

[解决 GRE 和 IPsec 中的 IPv4 分段、MTU、MSS 和 PMTUD 问题](https://www.cisco.com/c/zh_cn/support/docs/ip/generic-routing-encapsulation-gre/25885-pmtud-ipfrag.html)

使脚本可运行

```sh
chmod a+x /etc/strongswan.d/ipsec-notify.sh
```

编辑 `/etc/ipsec.conf` 加入以下内容

```sh
conn tunnel # 名字随便起
   leftupdown=/etc/strongswan.d/ipsec-notify.sh # 指定连接时提前运行的脚本
   leftprotoport=47 # 仅加密 47 端口流量，即 GRE 隧道
   right=80.251.XXX.33 # 填写对端地址
   rightprotoport=47 # 同上
   # 由于经过 NAT，本端与对端读取 ID 不一致无法建立一阶段连接
   # 手动指定使其一致
   leftid=121.43.117.81 
   ike=aes256-sha2_256-modp1024! # 加密算法等
   esp=aes-sha2_256!
   authby=secret # 使用 Pre-Share Key 连接
   auto=start # IPSec 服务启动时自动启动
   keyexchange=ikev2 # 使用 IKEv2 模式
   type=transport # 设置为传输模式
```

指定 Pre-share Key

```sh
sudo vim /etc/ipsec.secrets
```

```sh
# /etc/ipsec.secrets

: PSK 'password'
```

启动 IPSec 服务

```sh
sudo systemctl enable ipsec
sudo systemctl restart ipsec
```

### US 节点配置

由于 红帽使用的上游 IPSec 包并非 `strongswan`，如要使用 `strongswan` 可从 `epel` 安装

```sh
sudo yum -y install epel-release
```

安装 `strongswan`

```sh
sudo yum -y install strongswan
```

默认情况下配置文件目录仅有 700 权限，对于习惯使用非 root 用户来说非常不便，因此先给他加点权限

```sh
sudo chmod o+rx /etc/strongswan
```

修改文件，基本同上

```sh
vim /etc/strongswan/ipsec-notify.sh
```

```sh
# /etc/strongswan/ipsec-notify.sh

#!/bin/sh
ip tunnel add gre1 mode gre local 80.251.XXX.33 remote 121.43.XXX.81
ip link set gre1 up
ip addr add 10.0.0.1/30 dev gre1
ip link set dev gre1 mtu 1400 # IPv4 GRE + IPSecV4 transport
```

```sh
chmod +x /etc/strongswan/ipsec-notify.sh
```

建立 IPSec 连接

```sh
sudo vim /etc/strongswan/strongswan.d/charon.conf
```

```sh
# /etc/strongswan/strongswan.d/charon.conf

install_routes = no
```

```sh
sudo vim /etc/strongswan/ipsec.conf
```

```sh
# /etc/strongswan/ipsec.conf

conn tunnel # 名字随便起
   leftupdown=/etc/strongswan/ipsec-notify.sh # 指定连接时提前运行的脚本
   leftprotoport=47 # 仅加密 47 端口流量，即 GRE 隧道
   right=121.43.XXX.81 # 填写对端地址
   rightprotoport=47 # 同上
   ike=aes256-sha2_256-modp1024! # 加密算法等
   esp=aes-sha2_256!
   authby=secret # 使用 Pre-Share Key 连接
   auto=start # IPSec 服务启动时自动启动
   keyexchange=ikev2 # 使用 IKEv2 模式
   type=transport # 设置为传输模式
```

```sh
sudo vim /etc/strongswan/ipsec.secrets
```

```sh
# /etc/strongswan ipsec.secrets

: PSK 'password'
```

CentOS 默认开启防火墙，放行 IPSec 与 GRE 流量

```sh
sudo firewall-cmd --add-service ipsec --permanent
sudo firewall-cmd --add-service gre --permanent
sudo firewall-cmd --reload
```

关闭 SELinux，否则 `ipsec-notify.sh` 无法执行

```sh
sudo setenforce 0
sudo sed -i s/SELINUX=enforcing/SELINUX=disabled/g /etc/selinux/config
```

启动 IPSec

```sh
sudo systemctl start strongswan
sudo systemctl enable strongswan
```

检测隧道连通性吧！

![IPSec 流量](/images/other/166ee9349202d73ed7549a39d53a0012b137f78d72f1730337a5857edd4026f9.png)  

## BGP

在 Linux 上使用动态路由协议通常有两种软件

- BIRD 受许多 Linux 社区欢迎，采用配置文件方式
- FRRouting 简称 `frr`，由于其采用 Cisco-Like 方式配置，对网工出身更友好

我最多只能算半个网工，但 Cisco CLI 我还是蛮熟悉的，我决定使用 FRRouting。

FRRouting 是 GNU zebra 以及 Quagga 开源项目的 Fork，现在正在活跃开发中，而原项目已经停止维护了。

在这个逻辑拓扑中，我将会使用 CN 站点作为路由反射器，使 US 与 Home 可互相通信

### 安装 frr

#### CN

Ubuntu 的 apt 源中没有 frr，虽然可以使用 snap 方式安装，但我更喜欢使用 apt,因此需要添加官方源

```sh
curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -
echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) frr-stable | sudo tee -a /etc/apt/sources.list.d/frr.list
sudo apt update && sudo apt install frr frr-pythontools
```

给其他用户增加配置文件目录读权限

```sh
chmod o+rx /etc/frr
```

编辑 `/etc/frr/daemons` 文件，根据需要打开协议

```sh
# /etc/frr/daemons

bgpd=yes
```

重启 frr 服务

```sh
sudo systemctl restart frr
```

#### US

Ubuntu 源里没有的东西 CentOS 更不可能有，嗯

```sh
curl -O https://rpm.frrouting.org/repo/frr-stable-repo-1-0.el7.noarch.rpm
sudo yum install ./frr-stable-repo-1-0.el7.noarch.rpm
sudo yum install frr frr-pythontools
```

也用同样的方式选择需要的协议服务

```sh
sudo vim /etc/frr/daemons
```

```sh
# /etc/frr/daemons

bgpd=yes
```

防火墙放行 BGP

```sh
sudo firewall-cmd --add-service bgp --permanent
sudo firewall-cmd --reload
```

### 建立 BGP 邻居

FRR 的配置全部使用 `vtysh` 进行

`vtysh` 非常类似 Cisco CLI，或者说几乎一模一样

但在此之前不要忘记打开 Linux 内核 IP 转发啊，别忘了啊！

#### CN

需要与另外两个站点建立 BGP Peer，并设定两个站点为 RR

```sh
router bgp 65000
    no bgp default ipv4-unicast
    neighbor 10.0.0.1 remote-as 65000
    neighbor 10.0.9.1 remote-as 65000
    !
    address-family ipv4 unicast
        neighbor 10.0.0.1 activate
        neighbor 10.0.0.1 route-reflector-client
        neighbor 10.0.0.1 soft-reconfiguration inbound
        neighbor 10.0.9.1 activate
        neighbor 10.0.9.1 route-reflector-client
        neighbor 10.0.9.1 soft-reconfiguration inbound
    exit-address-family
!
```

#### US

由于 RR 的存在，仅与 RR 建立 BGP Peer 即可

```
router bgp 65000
    no bgp default ipv4-unicast
    neighbor 10.0.0.2 remote-as 65000
    !
    address-family ipv4 unicast
        neighbor 10.0.0.2 activate
        neighbor 10.0.0.2 soft-reconfiguration inbound
    exit-address-family
!
```

#### Home

同上，与 RR 建立 Peer 即可

```sh
set protocols bgp 65000 parameters default no-ipv4-unicast
set protocols bgp 65000 neighbor 10.0.9.2 remote-as '65000'
set protocols bgp 65000 neighbor 10.0.9.2 address-family ipv4-unicast soft-reconfiguration inbound
commit
save
```

### 宣告网络

对于 Home，需宣告内部网络

```sh
set protocols bgp 65000 address-family ipv4-unicast network 192.168.1.0/24
set protocols bgp 65000 address-family ipv4-unicast network 192.168.50.0/24
set protocols bgp 65000 address-family ipv4-unicast network 192.168.100.0/24
```

这是毫无疑问的

由于三者之间的 Peer 为 iBGP，并且无法使用 IGP 路由，静态路由实现并不十分优雅，因此我选择在 RR 上宣告中间路由

OpenVPN 这时候带来了一个大麻烦，Site-to-Site VPN 的掩码居然是 32 位的，因此无法宣告 10.0.9.0/30

在 CN 节点上存在一条 10.0.9.1/32 的路由条目，由 OpenVPN 在连接成功时创建的，我准备利用一下这一点。

在 RR 上使用 BGP 宣告 10.0.9.1/32，再在 RR 上做一条 10.0.9.0/30 的聚合，这样至少可以使其他节点可以收到 10.0.9.0/30 的路由，并且当 10.0.9.1/32 路由失效时，这条聚合路由也会消失。

由于不需要其他站点收到 10.0.9.1/32，因此在聚合时选择只通告聚合路由。

配置如下

```sh
router bgp 65000
    !
    address-family ipv4 unicast
        network 10.0.0.0/30
        network 10.0.9.1/32
        aggregate-address 10.0.9.0/30 as-set summary-only
    exit-address-family
!
```

这样一来，另两个站点都可以收到他们自己缺失的所需要的路由条目

对于 US 节点，路由表会显示路径

![US Routing Table](/images/other/69af5fa782a5d90c5824700be7436326a40e5d686812078a970e9fe14f315be2.png)  

BGP 表不存在 incomplete 路由，且所有路由均能正常写入路由表

![US BGP Table](/images/other/749c88a03fc81a350ee45e5f5e0ba7b9aa2e524f37610b3f419a408096d674b9.png)  

对于 Home，可以收到他所缺失的 10.0.0.0/30 路由条目

```console
vyos@home:~$ show ip bgp
BGP table version is 11, local router ID is 192.168.100.1, vrf id 0
Default local pref 100, local AS 65000
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*>i10.0.0.0/30      10.0.9.2                 0    100      0 i
*>i10.0.9.0/30      10.0.9.2                      100      0 i
*> 192.168.1.0/24   0.0.0.0                  0         32768 i
*> 192.168.50.0/24  0.0.0.0                  0         32768 i
*> 192.168.100.0/24 0.0.0.0                  0         32768 i

Displayed  5 routes and 5 total paths
```

唯一的问题是 Home 路由表中会多出 10.0.9.0/30 的路由，但根据路由的最长匹配原则，10.0.9.2/32 应直接匹配原来由 OpenVPN 维护的路由，10.0.9.1/32 则属于本机无需路由

traceroute 的结果，从 Home 到 US 中间有两跳空缺，从 US 反过来只能保证到达目的地

## 总结

最后如果想要 10.0.9.0/30 不传给 Home 的话其实可以用 route-map 做一下过滤，虽然确实存在环路的风险不过我认为大概率并不会。

驱使我搞出来这个东西的主要原因是 GFW 不让我从 US 用 OpenVPN 直连回家，正好我有一台阿里云闲置的轻量应用服务器，就先用上了。

显然这个方法不能解决延迟问题，不过在我实际使用过程中，比起从公网直连，掉包的现象倒是大有改观，不过由于中转节点的带宽限制，实际上用途还是很局限。

这样搞还有一个大原因，就是我的域名没做备案，不然很多事情会变好办一些。

不过强化了一下网络知识，还是很有趣的。
