---
title: Private VLAN-专用 VLAN
date: 2021-03-16
categories: [网络]
tags: [network]
---
先上一个开胃小菜 ( 其实已经吃饱了 )

A Free Video From CBT Nuggets: [MicroNugget: Private VLANs](https://www.youtube.com/watch?v=xl3_zgaZuH8)

( 科学上网需求注意 )

Private VLAN 是一种更好的控制广播域的手段，他可将一个大的广播域再分割成几个更小的广播子域，而他们与上行链路的通信不受影响。

Private VLAN 通常翻译作 `专用 VLAN` 或 `私有 VLAN` 

使用专用 VLAN 时，客户端的 IP 地址通常制定在一个子网内，但交换机可能会阻塞数据链路层的通信。( 是否允许数据链路层的访问需要视配置而定 )

# 专用 VLAN 的类型

专用 VLAN 将一个主要 VLAN 分割成多个次要 VLAN，并可保留相同的 IP 配置。同一主要 VLAN 的网络设备都配置在同一子网下。

专用 VLAN 按工作模式应分为两种：

- Primary VLAN: 主要 VLAN 将数据转发至属于他的次要 VLAN
- Secondary VLAN: 次要 VLAN 为网络设备实际处在的 VLAN，他按照设备是否可进行二层通信而又分为两类：
    - Isolated: Isolated 中的设备都可与主要 VLAN 以及对应的上行链路通信，但在 Isolated VLAN 中的设备无法互相通信。一个主要 VLAN 可以包含多个 Isolated VLAN，但通常只需要一个。
    - Community: Community 中的设备不但可与主要 VLAN 以及对应的上行链路通信，而且同处一个 VLAN 的设备也可互相通过二层通信。一个主要 VLAN 同样可以包含多个 Community VLAN

但不要忘记 VLAN 最开始的定义，不同 VLAN 间是无法通过二层通信的。

专有 VLAN 的接口主要分两种：

- Promiscuous Port: 这种端口连接上行网关设备，设为这种接口的设备可与连接到主要 VLAN 和所有隶属于此主要 VLAN 的所有次要 VLAN 进行通信。
- Host Port: 主机端口按工作模式与连接的 VLAN 不同又分为两种：
    - Isolated Port: 连接至 Isolated VLAN 的端口，此端口只能与 P-Ports 通信
    - Community Port: 连接至 Community VLAN 的端口，此端口可与 P-Ports 和同一 VLAN 下的其他设备通信。

# Private VLAN 配置

对于 Cisco 设备，以及具有 Cisco-Like CLI 的设备，配置 Private VLAN 大概分下面几步：

- 对于 Cisco，首先需要将 VTP 置于 Transparent 模式
- 建立多个 VLAN
- 在 VLAN 配置模式中配置 Private VLAN 类型
- 将主要 VLAN 与对应的 Secondary VLAN 作关联
- 在端口配置模式中配置端口类型
  - 对于 Promiscuous Port，需要制定 Primary VLAN 以及关联的 Secondary VLAN 们。
  - 对于 C-Ports 和 I-Ports ，需要指定 Primary VLAN 与 Secondary VLAN

本例中， VLAN 10 为 Primary VLAN，VLAN 11 为 Community VLAN，Isolated VLAN 配置与其类似，不再赘述。

```console
SW1(config)#! 配置 VTP 为 Transparent MODE 以使用 Private VLAN
SW1(config)#vtp mode transparent
SW1(config)#! Add a Community VLAN
SW1(config)#vlan 11
SW1(config-vlan)#private-vlan community 
SW1(config-vlan)#exit
SW1(config)#! Add a Isolated VLAN
SW1(config)#vlan 12
SW1(config-vlan)#private-vlan isolated
SW1(config-vlan)#vlan 10
SW1(config-vlan)#private-vlan primary
SW1(config-vlan)#private-vlan association add 11,12
SW1(config-vlan)#int g0/1
SW1(config-if)#switchport mode private-vlan host
SW1(config-if)#switchport private-vlan host-association 10 11
SW1(config-if)#int g0/2
SW1(config-if)#switchport mode private-vlan promiscuous
SW1(config-if)#switchport private-vlan mapping 10 11,12
```

如果要做 Layer 3 VLAN SVI 与 Private VLAN 绑定的话，只能做在 Primary VLAN 的 SVI 上：
```console
SW1(config)#int vlan 10
SW1(config)#! Mapping the Secondary VLAN To SVI of Private VLAN
SW1(config)#private-vlan mapping 11-12
```

Private VLAN 的实验貌似在 GNS3 中配合 IOSvL2 并不能正常工作，如果需要实验这部分最好找真机。

> [Consolidated Platform Configuration Guide, Cisco IOS Release 15.2(7)E (Catalyst 2960-X Switch) Cheaper:Configuring Private VLANs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst2960x/software/15-2_7_e/b_1527e_consolidated_2960x_cg/configuring_private_vlans.html)
>
> 啥叫正经文档啊 ( 战术后仰 )
>
> Ruijie 你那也好意思叫文档？
>
> 我都不好意思说我照着 Cisco 文档配你家 OSPFv3 的事