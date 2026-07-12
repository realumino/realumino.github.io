---
layout: post
title: "Linode能获取到IPv6地址但是依然没用"
date: 2026-07-11 17:22:10 +0800
tags: [Linode, IPv6, CoreOS, NetworkManager]
lang: zh
---

在Linode上手动安装好CoreOS后发现IPv6虽然可以获取到，但是实际上是没用的，一筹莫展。

## 判断原因

从逻辑上看，路由表似乎是没有问题的，唯一能称得上问题的是好像第一条不是以公网网段开头的规则（最后也证明这个不是原因）。

以此为切入点，开始怀疑是不是因为Router Advertisement没接受，遂使用`sysctl`检查`net.ipv6.conf.all.accept_ra`，还真的是`0`，再手动指定成`2`，但是没有任何作用。

随后，Gemini提到了一个关键信息：Linode要求机器上的IPv6地址和控制面板上显示的完全一致。我又执行了一次`ip addr`，对比了控制面板上的IPv6和机器上显示的IPv6，后半部分则截然不同。Linode分配给机器的IPv6使用**EUI64**规则产生，大部分云厂商的自带镜像也会配置好使用该算法，但是因为我使用的是手动安装，取得前缀后就是用标准的**Stable Privacy**生成后半部分，因此Linode拒绝路由我的IPv6流量。

但是，我在Vultr上是没有遇到任何问题的。虽然我早就发现控制面板上的IPv6和机器上的不一样，但因为它还能没有任何问题地用，所以我当时还觉得Vultr抽风了。这里也很搞笑，Vultr是直接支持CoreOS的，但是好像也没有把EUI64写死在里面。

## 解决方案

解决方案目标十分明确，就是对`NetworkManager`下手，里面有一个选项就是`ipv6.addr-gen-mode`，直接指定成`eui64`就万事大吉了。

这里也有个小插曲，因为我一开始是打算用NetworkManager的config形式来指定的，但是Gemini犯了一个致命的错误：它把`ipv6.addr-gen-mode`的键值对应搞错了，说`1`是`eui64`，但事实上应该是`0`。结果导致我无论怎么调试都没办法切换到EUI64形式，我还以为这一招不管用了。

最终还是用的创建`.nmconnection`的形式完成配置的，懒得再切换回去尝试了。

`.nmconnection`是一个Delta形式的keyfile，里面只包含涉及修改部分，而且还支持自动匹配，因此可以不指定任何识别器，直接指定地址生成模式就可以保留其他设置默认。而且，默认情况下如果没有用`nmcli`进行过任何改动的话，`.nmconnection`根本不会生成，而通过它修改后也只会保存手动修改的部分。

一下是一个举例（里面还包含了DNS over TLS配置，这个无所谓了）：

```conf
[connection]
id=WiredPublic
type=ethernet
match-device=type:ethernet
dns-over-tls=2

[ethernet]

[ipv4]
method=auto
dns=1.1.1.1;1.0.0.1
ignore-auto-dns=true

[ipv6]
method=auto
addr-gen-mode=eui64
dns=2606:4700:4700::1111;2606:4700:4700::1001
ignore-auto-dns=true
```

然后再启动，默认的IPv6地址就会使用EUI64标准生成，和云后台控制面板上的是一致的，也可以正常使用IPv6通信了。

## 总结教训

当一个东西看上去逻辑完全正常的时候，往往不是你错了，而是碰上了你无法控制的一些限制。

## 后期更新

最后又尝试了一下用config形式来指定EUI64，确实成功了，可惜它不能解析拒绝DNS的要求，于是又换回来了。
