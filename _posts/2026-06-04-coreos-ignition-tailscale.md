---
layout: post
title: "Day1：初识Ignition，配置防火墙，安装Tailscale"
date: 2026-06-04 21:12:46 +0800
tags: [CoreOS, Ignition, Tailscale, nftables, SELinux]
lang: zh
---

## Tailscale讲起

首先其实是一个Container常识，那就是**被map的目录必须要是事前存在的**，quadlet不会自动建立目录，因此添加创建目录至关重要。否则Quadlet直接报错，还不直接告诉你哪错了，只会返回125错误，还是一点一点手打出完整的podman命令才发现实际问题来自podman本身，也就是目录映射问题。
```yaml
storage:
  directories:
    - path: /var/lib/tailscale
      mode: 0755
```

此外，在CoreOS中，**SELinux也是不得不品的一环**，因此在Volume中还需要加上:Z标签，而且不管是文件还是设备，都需要加上。
```yaml
- path: /etc/containers/systemd/tailscaled.container
    mode: 0644
    overwrite: true
    contents:
      inline: |
        [Container]
        Volume=/dev/net/tun:/dev/net/tun:Z
        Volume=/var/lib/tailscale:/var/lib/tailscale:z
```

自此Tailscale可以在Userspace模式下运行，可以在dashboard中看到机器。

### 永远不要尝试使用Tailscale SSH

Tailscale已经在容器里面了，要访问主机上的Shell会非常麻烦甚至基本不可能，因此在这种情况下绝对不要想着去弄Tailscale SSH这种给自己加戏太多的功能。==直接宿主机上运行原生sshd，然后走tailscale连接不暴露在公网就行，period。==

## nftables设置

Fedora CoreOS默认的防火墙并**不是firewalld，而是纯粹的nftables**。因此只需要直接用storage写一个nftables的配置文件就行。
比较重要的一点是设置好tailscale0的权限，不然用Tailnet会访问不了机器上的SSH。
```yaml
- path: /etc/sysconfig/nftables.conf
      mode: 0644
      overwrite: true
      contents:
        inline: |
          # nftables ruleset for public-facing interface
          flush ruleset
          table inet filter {
              chain input {
                  # allow tailscale
                  meta iifname "tailscale0" accept
              }
          }
```

在CoreOS中，nftables的配置文件放置在 `/etc/sysconfig/nftables.conf` 下，因为在 `nftables.service` 下指定就是这个目录，因此如果想开机自动apply必须保存在这个目录下。

不要忘记允许nftables服务，否则不会开机自动apply：
```yaml
systemd:
  units:
    - name: nftables.service
      enabled: true
```

## 总结

1. 容器映射到目录，必须要事先存在
2. SELinux存在的情况下，z标签需要时刻注意打上
3. 不要尝试让一个容器接管太多事情，当你觉得这有点越界的时候就不要死磕了

## 其他值得记住的东西

- 现在很多都可以使用directory摆放配置文件，例如sysctl、ssh等，其特征是数字越大话语权越大，例如`99-xxx.conf`事实上意味着这里面提到的才是最终一锤定音的。
```yaml
- path: /etc/ssh/sshd_config.d/99-hardening.conf
      mode: 0600
      overwrite: true
      contents:
        inline: |
          PermitRootLogin no
          PasswordAuthentication no
          PubkeyAuthentication yes
          ChallengeResponseAuthentication no
          MaxAuthTries 15
          X11Forwarding no
```
