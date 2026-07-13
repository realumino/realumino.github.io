---
layout: post
title: "（撰写中）存档：我正在用的AI智能助手，以及各自的配置方案"
date: 2026-07-13 22:00:00 +0800
tags: [AI Agent, Claude Code, Coding, Cloudflare AI Gateway]
lang: zh
---

首先，我没有钱订阅OpenAI或者Anthropic的Coding Plan，一分钱也没有。我现在只能抱着按量付费的国产DeepSeek感受顶级AI Agent的便利。因此，模型好不好其实无关紧要，重要的是不同的harness和完善程度，这是体验拉开差距的关键。

另外，因为我喜欢用[Cloudflare AI Gateway](https://www.cloudflare.com/products/ai-gateway/)，因此也会部分提及使用它的方法。

# 力压群雄之Claude Code （图形界面版）

[Claude Code](https://claude.com/product/claude-code)可以说是当之无愧的一哥。我以前没想过用它，因为要搞复杂的环境变量配置，但同样抵不过突发奇想，而且我发现其实配置没那么困难（图形界面可以说非常简单了）。这也是唯一一个需要一点小技巧才能用上的AI工具（因为我暂时不想尝试Codex了）。

可以参考OpenRouter的[这一篇教程](https://openrouter.ai/docs/cookbook/coding-agents/claude-desktop-integration)，都能被OpenRouter写出来了大抵就不算是歪门邪道的。而且最惊喜的是自定义Gateway之后都不需要有Anthropic账号的，直接白嫖Anthropic强大的harness。

不过也不要被教程里的吓到了，教程里面说“只能用Claude模型”，而且实际测试下来模型自发现是没用的，依然需要手动输入模型信息。

而且，在配置模型的时候会有个名称检测，如果输入“deepseek”、“dpsk”（没想到吧，这个也是关键字）或者其他一众友商的名称就会拒绝Apply。不过幸亏DeepSeek早有准备，直接把模型名称输入成`claude-opus`、`claude-sonnet`就可以自动映射为DeepSeek自己的模型，详情可以看[DeepSeek的文档](https://api-docs.deepseek.com/guides/anthropic_api)。

图形界面就是这么容易，简单配置完后就可以正常使用了。

当然，在使用Cloudflare AI Gateway的时候一开始遇到了点麻烦，我本来以为要添加一个自定义Provider然后指定为DeepSeek的Anthropic API，但是后来发现不需要，直接把Endpoint设定为`https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/deepseek/anthropic`就可以了。这里的`/anthropic`并不是Cloudflare自己的兼容层，而是调用的DeepSeek自己的兼容接口。这也说明Cloudflare预定义的Provider也是纯粹的Base URL形式，没有搞别的限制或者花样

唯一的缺陷就是在Dashboard中模型名称会显示为伪装后的名称，且计费、token计数都会失效。

# Claude Code CLI

这个是最经典的，虽然感觉整体体验上应该和图形界面版没啥区别。这有个“优势”就是不会强制要求模型ID有`claude`或`anthropic`之类的关键字，但除此之外也没别的了。同样的，Cloudflare AI Gateway对Anthropic API的请求会睁眼瞎，无法计数和预计费用。

