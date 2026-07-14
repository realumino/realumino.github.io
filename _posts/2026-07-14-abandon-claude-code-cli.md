---
layout: post
title: "Claude Desktop和Claude Code CLI之权衡"
date: 2026-07-14 20:35:00 +0800
tags: [AI Agent, Claude Code]
lang: zh
---

已经受不了两个Claude Code在电脑里面打架了，先讲一下我个人的发现。

## 资料夹共用

用户资料夹中的`.claude`是同时被Claude Desktop和Claude Code使用的，并且目录结构完全一致（但是聊天信息并和不会互相读取，仍然需要手动导入），`plugin`却可以同时被两个程序读取且状态也可以同步。

`settings.json`中设置的环境变量并不会被读取。

整体来说，CLI和Desktop建立的所有文件都是一样的，但是实际使用的时候却又能做到大部分不干扰，而且根本不知道具体文件属于哪个管理。

## 放弃了Claude Code CLI

我本来想的是两个一起用，因为CLI也可以在VS Code装插件用，但是最终实在是受不了数据互相打架然后没办法分离带来的痛苦，而且体验非常糟糕，最终决定还是留下Claude Desktop。

大部分都是差不多的，所以不想继续考虑Claude Code CLI了。