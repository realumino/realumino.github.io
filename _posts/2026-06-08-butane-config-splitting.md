---
layout: post
title: "Day2：让配置更高级也更可塑"
date: 2026-06-08 20:37:43 +0800
tags: [Butane, CoreOS, YAML, 运维]
lang: zh
---

把一坨配置全都写进一个大配置文件里，修改更新的时候跟裹脚布似的，臃肿且及其不优雅。把butane文件拆分下来是非常有必要的，这样可以避免修改的时候碰到别的东西。

要把这些yaml分开其实很简单，就是直接每一个都定义`variant`和`version`，然后剩下来的直接把有变化的部分定义、写进去就行了。

但是合并的时候是老大难。虽然AI喜欢说butane有merge功能，但是实际测试下来这个和理解的分布式配置毫无关系，**它必须合并的一个已经编译成ignition的json，而不能直接把yaml丢进去**。**它的merge是发生在Ignition层面上的。**

这其实给了两种思路，一个是写一个`main.bu`然后再写乱七八糟的配置文件，之后先把一堆乱七八糟的编译好，用butane自己的merge功能实现真正的合并。但是问题也很明显，这会给配置文件带来状态不同步的问题，可能我已经编译好的突然发现还是需要修改的，那改完还得重编译一下，很麻烦。

还有一种方法就是在传递给butane之前就先合并成大的yaml，然后butane再做他自己的事情。我也更倾向于这个方式。

## 使用yq合并yaml

butane是完全不支持操作yaml的。`yq`就是一个轻量级yaml操作工具，用在这里最合适不过。

```powershell
yq eval-all '. as $item ireduce ({}; (.systemd.units // []) as $su | (.storage.files // []) as $sf | (.storage.directories // []) as $sd | . * $item | .systemd.units = $su + ($item.systemd.units // []) | .storage.files = $sf + ($item.storage.files // []) | .storage.directories = $sd + ($item.storage.directories // []))' $bu > merged.bu
```

yq的精髓就在于中间那一串自定义串，当然，那一长串也是ai写的，而且因为yq识别不了powershell的换行符，所以只能挤在一行里。

不过这里面的逻辑值得深入研究一下。总的来说就是yq也是不能直接把各个文件无缝拼接在一起的，并且也是会出现大包大揽的情况。`*`运算符可以把正在读取的文件**"深度合并"**到累加器中。但是每次都会对原有的内容覆盖。

因此这段命令中写了一个workaround，先将累加器中的特定部分保存下来，然后再进行深度合并，合并完之后再把本来应该有的东西加上去，如此循环往复，就能实现完整拼接。

但是这个命令也有一定局限性，那就是它只对systemd、storage下的一部分起作用，如果后期要改，这个命令也要接着更新。

## 把inline换成local

另外一个提高编写时的幸福感的方式是使用butane的local功能。

```yaml
storage:
  files:
    - path: /etc/sysconfig/nftables.conf
      mode: 0644
      overwrite: true
      contents:
        local: files/nftables.conf
```

如果还是inline的话，还得一边修改butane自己的配置，一边看这些又臭又长还不能正确高亮显示的配置文件，这样做可以显著提升可读性，并且配置文件本身也可以被正确的解释器读取，做出正确的高亮显示。

当然butane命令自己也得修改一下：

```powershell
butane merged.bu --strict --pretty -d . -o ignition.json
```

搞好这些终于可以愉快地继续去搞caddy配置文件了，再也不要担心会把其他部分搞花力（喜）

## 更新：注意行尾换行符！

文字不只是看上去就像看上去的那样，在Windows中，末尾换行符默认使用的是CRLF（`\r\n`），在Unix下`\r`会被识别成内容，进而出现一些预期之外的错误，编译成Ignition文件后这个依然会被保留的！

背锅案例：

```bash
/etc/sysconfig/nftables.conf:2:14-15: Error: syntax error, unexpected CRLF line terminators, expecting end of file or newline or semicolon

flush ruleset

^^

/etc/sysconfig/nftables.conf:2:101-106: Error: syntax error, unexpected policy

flush ruleset

^^^^^^

/etc/sysconfig/nftables.conf:2:113-114: Error: syntax error, unexpected CRLF line terminators

flush ruleset
```

因此，一定要时刻注意末尾换行符，能换成LF就换成LF！

## 总结

1. 能使用分离式配置文件就用配置文件
2. 花了很多时间在yq上，但是时间效用存疑
3. **跨系统运维要时刻注意文件编码的兼容性**
