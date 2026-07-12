---
layout: post
title: "转向Proxmox，在LXC里安装Tailscale"
date: 2026-06-29 14:48:40 +0800
tags: [Proxmox, LXC, Tailscale, Alpine]
lang: zh
---

总算是到家，能碰到自己心心念念的homelab了。之前已经苦snowflake形式的配置久矣，一到家就赶紧给它安装上了Proxmox VE。

重头戏依然是先解决Tailscale的问题，不然根本连不回家里。

## 选用正确的CT Templete

为了极致的轻量，我打算使用Alpine的，于是从Alpine官网下载minimal rootfs镜像，只有3MiB，看上去确实很不错。

不过这个镜像是用来放进Docker打包的镜像，而不是运行在LXC里，因为这个真的很精简，要啥没啥。

最后还是得去[Linux Containers](https://images.linuxcontainers.org/)上下载精简完整镜像，这样可以避免很多奇怪的错误（例如，alpine minimal rootfs里是没有rc-service组件的）

## 做好权限配置，安装缺少的依赖

其实接下来的问题很简单，[Tailscale官网](https://tailscale.com/docs/features/containers/lxc/lxc-unprivileged)就有对应的教程，直接把`tun`映射进去就行。

但是这可是alpine，里面依然是啥也没有，因此还得手动装`nftables`和`curl`。

```sh
apk add nftables curl
```

之后就和正常的Tailscale配置一模一样了，复制安装链接，或者手动安装都行。而且Proxmox默认桥接，可以在路由器上看到这个容器"独立的网卡"。

## 如何设置Exit-node和Subnet-router

这是本篇的重头戏。

alpine默认是没有开启`ip_forward`的，因此需要手动修改`sysctl.conf`（也可以在`sysctl.d`中新增，这个无所谓）：

```conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

然后直接`sysctl -p`（或者加上conf文件）就行。

但是，OpenRC默认是不会开机自应用sysctl的，需要使用`rc-service`开启：

```sh
rc-update add sysctl boot
rc-service sysctl start
```

这样无论怎么重启都可以保持几个转发功能的正常使用了。

## 拯救IPv6

当然，一旦把forward弄好，IPv6就会神奇地消失，这是因为forward打开后会自动拒绝ra数据包，导致容器无法被分配IPv6。这个也很好结局也，强制接收ra就行了。

```conf
net.ipv6.conf.all.accept_ra = 2
```
