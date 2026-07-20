---
layout: post
title: "Podman 代理配置导致局域网服务无法访问的排查与解决"
date: 2026-07-20 13:30:00 +0800
tags: [Podman, Proxy, Home Lab, Servarr]
lang: zh
---

在中国使用容器引擎必然要用一些中国特色的手段，比如设置代理。

但是即使设置的时候是在`[engine]`下，代理设置也依然会被传递到容器里，Docker就不会这样。

这就导致在不知情的情况下一些本应直连的流量走了代理，本地流量亦是如此，结果就导致互相无法访问（例如无法连接Torrent客户端、几个arr都无法互相访问等）。

如果使用Podman设置代理后发现无法访问本地资源，可以使用以下命令验证：

```bash
podman exec -it <container name> env | grep -i proxy
```

不出意外会发现容器内设置了代理环境变量。

解决方法也很简单，那就是在配置文件`[container]`字段中指定`http_proxy`为`false`即可阻止代理环境变量被传入容器。

整个配置如下：

```bash
[containers]
http_proxy = false

[engine]
env = [
    "ALL_PROXY=socks5://x.x.x.x:1080",
    "HTTP_PROXY=socks5://x.x.x.x:1080",
    "HTTPS_PROXY=socks5://x.x.x.x:1080"
    ]
```

如果容器确实需要用到代理，可以直接在Quadlets文件中指定环境变量，或者使用容器内程序自带的Proxy兼容功能。