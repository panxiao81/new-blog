---
title: 新的 E5 更新程序
date: 2026-03-22T01:11:00+09:00
categories: ['Programming']
tags: ['e5']
---

Microsoft 一直很慷慨的给我们白嫖 E5 Developer 订阅，可以使用几乎大部分功能。当然这个环境是给 365 生态系统开发者用的沙盒环境，但只要检测到积极使用即可每 60 天一次续约，因此理论上可以无限续下去。

众所周知网上已经有一些成熟的程序可用了，但我不想搞一台 Windows Server 就为了挂这玩意。而且某个著名的 Renew X 居然用的是 Password flow 让我总觉得安全性差点意思，毕竟要我把密码交出去，所以发挥自己动手丰衣足食的传统，我 Vibe 了一个。

我其实完全自己写了这个程序的原型，开始尝试过用 .NET 但又想在低资源机上运行，而我又不熟悉 .NET 生态（其实最开始想用 Java 但很快就被我自己否定掉了），于是现在的原型版本是 Go 写的。后面的一些重构，测试等等则是完全交给了 AI，我只进行了一些架构设计。

目前的版本仍依赖数据库，最初的版本是想在 OCI 上运行所以只支持 MySQL（OCI Free tier 中有 MySQL 实例），后来挪到本地运行，而本地的主机上只有 PostgreSQL，因此目前这个程序只支持这两个数据库。我暂时没有给他加入 SQLite 支持的打算。整个数据库访问层是手写 SQL + sqlc 生成代码的，意味着每加一个后端支持就要多写一份 SQL，而且要改 Repository 层的代码加入支持，我暂时没有 SQLite 支持的需求就不做了。

目前我不打算将我的服务器作为共享服务给大家使用，虽然有把 Web Page 暴露公网，但你不用我的 Entra 租户账户登录是登录不进后台的，因此如果各位也想用的话请大家自己部署，在 ghcr.io 里已经有打包好的 Docker 镜像，以及 Helm Chart，各位可以参考使用。

文档没写，不过想必各位用随便什么 AI Agent 读一下 AGENTS.md 然后让 AI 给你装也不是什么难事吧，起码来个 OpenClaw 是不（

既然有 Web UI 我就把验证流程用了 Authorization Code flow。理论上稍微重构一下加一个 Device flow 也是可行的，不过都有 Web 页面了就没必要了吧。

名字就暂时叫 [e5renew](https://github.com/panxiao81/e5renew) 了。
