---
title: "自主开通一个美国 eSIM 手机卡"
date: 2021-12-12T13:52:00+08:00
draft: false
categories: [
    "闲话"
]
tags: [
    闲话
]
---

有时候我们需要一个海外手机号用来接码，如果你要注册海外应用，出于隐私考虑可能也希望有一个海外手机号可用。现在 Google Voice 虽然还能注册不少应用，但很多应用会检测到 Google Voice 是 VOIP 号码，不能用来收短信验证码。这篇文章教大家如何自己购买到美国 SIM 卡。

## SIM 卡选择

现在国内主流的美国 SIM 卡有 Ultra Mobile 的 PayGo 套餐卡，一个月只要 $3 月租。在 Wi-Fi Calling 状态下相当于在美国境内，可以走套餐内资费。套餐内有 100 条短信和 100 分钟通话。还有国内不能用的 100MB 流量。缺点是充值麻烦，需要在官网使用双币卡充值，对于没有双币卡的人来说是个麻烦事，淘宝代充要贵很多。Ultra 卡的好处是淘宝很好买到，而且商家可以带激活，当然现在在线激活已经不需要必须手机能收到信号了，自行激活也完全没问题。这张卡还有一个巨大的好处是可以在国内走中国移动的网络漫游，这样在没有 Wi-Fi Calling 的状态下也能收短信，但是需要走国际漫游资费，接收短信 $0.1 每条。

Ultra 还有一个不好的地方是它是实体卡，总归要占用一个手机卡槽。目前 Ultra 好像又不支持 eSIM 申请了，我的 Pixel 主卡槽已经插了国内卡，所以我希望最好是支持 eSIM 的运营商。

还有一个运营商国内提的不太多，是 Tello，这个运营商以携号转网方便和低价著称，而且邀请新人给的返利也很大方。重要的是 Tello 支持 eSIM，也支持 Wi-Fi Calling，支持 PayPal 充值，最便宜的套餐是 $5，贵上 $2 换来的是无限量的短信。支持 eSIM 和 Wi-Fi Calling 意味着可以在国内自主开通，完全没有中间商赚差价。

Tello 注册时填上邀请链接，开户后可以拿到 $10 作为奖励，欢迎填我的邀请码 [P39H4PGG](https://tello.com/account/register?_referral=P39H4PGG), 相当于白嫖两个月话费。

最便宜的，在国内长期用的套餐我推荐无限短信+100 分钟通话，没有数据流量这一档，只要 $5，它比 Ultra 还有一个好处是他的套餐是可以自由升降级的，如果你某一天想要用这张卡，可以升级到高级的套餐。

注意，只有海外版 iPhone 支持 eSIM，并且实体双卡的港澳版也是不支持的，Tello 的实体 SIM 卡我没见到国内有卖的，SIM 卡转运需要特殊地址，大多数不一定有这个条件，因此如果你没有支持 eSIM 的设备，那你只能去淘宝买 Ultra。

我有一台 Pixel 6a 专门使用海外应用，给他来一张海外 SIM 卡那也是再合适不过了。

## 自主开通

去 Tello 注册账号，再次推荐大家填我的邀请码。

进入 My Account，在 MY SETTING 里填上 Wi-Fi Calling 的地址，这里可以随意添一个地址，但必须是美国真实地址，否则无法开通 Wi-Fi Calling。

这张卡我测试下来不能在国内漫游，因此没有 Wi-Fi Calling 的场景下直接是无信号的，也因为如此第一次 Wi-Fi Calling 激活的速度也比较慢，多一点耐心。

接下来，选择 MY SIM -> Add a new line，选择 eSIM，选择套餐，填写个人信息，付款，等待 eSIM 卡的二维码就可以了。

手机扫码下载 eSIM 就可以了，我因为保持全程梯子，所以我也不清楚能否不挂梯子激活成功，不过建议还是常备梯子。

在手机上打开 Wi-Fi 通话功能即可，Tello 走的是 T-Mobile US 的网络，Wi-Fi Calling 实际上用到了 IPSec 协议，注意如果你的梯子机场对 UDP 转发支持不佳，或 NAT 有问题等等，Wi-Fi Calling 也会有问题。T-Mobile US 一部分 Wi-Fi Calling 的网络被墙了，一些地区直连有问题，最好能在路由器上把域名解析到没有被墙的 IP 上。你需要配置如下域名的 hosts

```text
208.54.35.163 epdg.epc.mnc260.mcc310.pub.3gppnetwork.org
208.54.35.163 ss.epdg.epc.mnc260.mcc310.pub.3gppnetwork.org
208.54.87.3 ss.epdg.epc.geo.mnc260.mcc310.pub.3gppnetwork.org
208.54.49.131 epdg.epc.mnc260.mcc310.pub.3gppnetwork.org
208.54.36.3 ss.epdg.epc.mnc260.mcc310.pub.3gppnetwork.org
```

如果我的这些 IP 地址在你那里不管用的话，可以查询一下以上域名的 A 记录，找一个你能正常使用的 IP 地址。

如果你的机场像我一样不太好用的话，在可以直连打开 Wi-Fi Calling 的基础上，把 T-Mobile US 的 Wi-Fi Calling 网段写进 Clash 代理规则让他们分流走直连就好，根据 T-Mobile 官网的说明，[Wi-Fi Calling on a corporate network](https://www.t-mobile.com/support/coverage/wi-fi-calling-on-a-corporate-network)，Clash 分流规则如下:

```yaml
rules:
- IP-CIDR,208.54.0.0/16,DIRECT
- IP-CIDR,66.94.0.0/19,DIRECT
- IP-CIDR,206.29.177.36,DIRECT
```

第一次开通 Wi-Fi Calling 我折腾了两天，因为我在路由器上配置了 hosts，但是 Wi-Fi Calling 也还是无法连接，昨日晚上打开手机突然发现有信号了，才试着改 Clash 分流规则，发现直连也能连接而且非常稳定，我才发现是 Clash 和机场的问题。

不过第一次开通 Wi-Fi Calling 确实需要一些时间，最起码等一天再看看。

有了 Wi-Fi Calling 之后就可以像在美国境内一样使用这个手机号了。

注意，Wi-Fi Calling 在手机上开梯子是没用的，必须在路由器上开梯子才行，如果你有高强度出墙需求，搞个软路由一类的东西应该也不是啥难事。

如果你用的是 iPhone，那么你甚至可以在没有 Wi-Fi 的情况下用另一张卡的流量给 Tello 号码开 Wi-Fi Calling，我觉得我的 Pixel 应该也支持这个功能，但也许跟墙有关，我没成功过。当然也有可能是因为需要 VoLTE？我不清楚这方面的技术细节。

你现在可以用这张卡收短信了，这是一张真正的实体手机卡，所以可以过绝大多数的接吗认证。你也可以试着用这张卡去开 Google Voice，但是开 GV 需要比较高质量的美国原生 IP，大多机场很难满足这个条件，并且我早已经有一个 GV 号码了。

最近很火的 ChatGPT 也需要一个非 VOIP 的号码注册，这张卡是完全没问题的。

![Wi-Fi Calling](/images/other/4780045a75f405da591a6341b6f22bf758a52acccd46c1b62b7bb6f6dd743325.png)  

如果你成功开启 Wi-Fi Calling 的话，应当像图中一样显示 Wi-Fi 通话，iPhone 也是类似的。尤其是如果你用的是支持国内漫游的 Ultra 卡，注意别扣了冤枉钱，漫游费用很贵，而且属于套餐外消费。
