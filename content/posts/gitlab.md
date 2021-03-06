---
title: "全新的私有 GitLab 来了！"
date: 2021-08-08T18:15:00+08:00
draft: false
categories: ['Linux','网络']
tags: [linux]
---

私有 GitLab 仓库来了！

地址是 [gitlab.ddupan.top](https://gitlab.ddupan.top)

<!--more-->

曾经有很短的一段时间我的 US 节点可以直接用 OpenVPN 连回家中的私有云。但很快就被 GFW 搞了。

我还有一台 VPS 跑在国内版阿里云，且一直没在跑业务。利用了放假的周末时间，利用这台节点做中转，使用 GRE over IPSec + BGP 路由将内网成功宣告，现在 US 节点又可以愉快的访问内网了。

这个解决方案回头应该会抽空写文，不过不是现在。届时我会用 EVE-ng 重新搭一个实验拓扑来解说。

家里的私有云配置并不高，况且当前除了内存，也非常缺存储，除了添置内存，我最想的就是添置更多的存储，当然最好是成熟的商业存储机，不过用 Ceph 做也不是不可以。

正好也重新整理了 US 节点，把 Caddy 迁移到了 v2，总算摆脱了旧版，v1 版本现在连文档都很难找了。

并且，现在为我们带来了一个副作用，我们可以通过这个隧道网络的内部地址访问两台云上的服务器了。

这也姑且可以算是混合云吧（
