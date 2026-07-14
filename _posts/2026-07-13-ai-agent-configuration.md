---
layout: post
title: "留档：我正在用的AI智能助手，以及各自的配置方案"
date: 2026-07-13 22:00:00 +0800
tags: [AI Agent, Claude Code, Coding, Cloudflare AI Gateway]
lang: zh
---

首先，我没有钱订阅OpenAI或者Anthropic的Coding Plan，一分钱也没有。我现在只能抱着按量付费的国产DeepSeek感受顶级AI Agent的便利。因此，模型好不好其实无关紧要，重要的是不同的harness和完善程度，这是体验拉开差距的关键。

另外，因为我喜欢用[Cloudflare AI Gateway](https://www.cloudflare.com/products/ai-gateway/)，因此也会部分提及使用它的方法。

## 力压群雄之Claude Code （图形界面版）

（这几个纯粹就是记录一下配置方案和一些痛点，因为太强大了，无需多言）

[Claude Code](https://claude.com/product/claude-code)可以说是当之无愧的一哥。我以前没想过用它，因为要搞复杂的环境变量配置，但同样抵不过突发奇想，而且我发现其实配置没那么困难（图形界面可以说非常简单了）。这也是唯一一个需要一点小技巧才能用上的AI工具（因为我暂时不想尝试Codex了）。

可以参考OpenRouter的[这一篇教程](https://openrouter.ai/docs/cookbook/coding-agents/claude-desktop-integration)，都能被OpenRouter写出来了大抵就不算是歪门邪道的。而且最惊喜的是自定义Gateway之后都不需要有Anthropic账号的，直接白嫖Anthropic强大的harness。

不过也不要被教程里的吓到了，教程里面说“只能用Claude模型”，而且实际测试下来模型自发现是没用的，依然需要手动输入模型信息。

而且，在配置模型的时候会有个名称检测，如果输入“deepseek”、“dpsk”（没想到吧，这个也是关键字）或者其他一众友商的名称就会拒绝Apply。不过幸亏DeepSeek早有准备，直接把模型名称输入成`claude-opus`、`claude-sonnet`就可以自动映射为DeepSeek自己的模型，详情可以看[DeepSeek的文档](https://api-docs.deepseek.com/guides/anthropic_api)。

图形界面就是这么容易，简单配置完后就可以正常使用了。

有几个小点需要注意一下：

1. 网络访问权限是有限的，默认只给访问指定的Gateway。但是可以在Inference Settings（左下角）中Workspace Restrictions中设置把Egress权限全部放开，就可以访问任何网络。
2. 同样是在Restrictions中可以开启Chat模式，很适合问一些简单的问题。
3. 可以直接用Anthropic的搜索工具（好像甚至不需要任何其他辅助），而且搜索的内容是没有过滤过的。

当然，在使用Cloudflare AI Gateway的时候一开始遇到了点麻烦，我本来以为要添加一个自定义Provider然后指定为DeepSeek的Anthropic API，但是后来发现不需要，直接把Endpoint设定为`https://gateway.ai.cloudflare.com/v1/{account-id}/{gateway-name}/deepseek/anthropic`就可以了。这里的`/anthropic`并不是Cloudflare自己的兼容层，而是调用的DeepSeek自己的兼容接口。这也说明Cloudflare预定义的Provider也是纯粹的Base URL形式，没有搞别的限制或者花样

唯一的缺陷就是在Cloudflare Dashboard中模型名称会显示为伪装后的名称，且计费、token计数都会失效。

## Claude Code CLI

这个是最经典的，虽然感觉整体体验上应该和图形界面版没啥区别。这有个“优势”就是不会强制要求模型ID有`claude`或`anthropic`之类的关键字，但除此之外也没别的了。同样的，Cloudflare AI Gateway对Anthropic API的请求会睁眼瞎，无法计数和预计费用。

当然，CLI也有CLI的好处。Desktop虽然外观精美，而且各项操作非常顺手，但是权限上貌似没有CLI的高的，而且它的Cowork是在后台开一个虚拟机，不能直接操控电脑。而且它很明显是给公司审计设计的，各种权限压得特别死，想要扩展没那么方便。

同样还是OpenRouter，也给出了完整的[设置方案](https://openrouter.ai/docs/cookbook/coding-agents/claude-code-integration#project-settings-file)。

整体来说就是靠环境变量实现自定义。此处留一个备份：

```json
{
  "autoUpdatesChannel": "latest",
  "env": {
    "ANTHROPIC_BASE_URL": "https://openrouter.ai/api", //换成自己的兼容端点
    "ANTHROPIC_AUTH_TOKEN": "<your-openrouter-api-key>", //换成自己的Key
    "ANTHROPIC_API_KEY": "", //重要：需要特意留空
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro", //都是A➗自己的模型，这里可以按照自定义模型的能力一一对应
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-flash",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4-pro"
  }
}
```

## 全能助手之Hermes

如果不是OpenClaw太垃圾，我应该是不会去了解Hermes的。Hermes也是我接触到的第一个CLI形式的AI Agent。准确来说，这应该是一个用CLI文字绘制GUI的Agent（Claude Code CLI也是如此，“TUI”）。相比OpenClaw，Hermes的操作逻辑还是非常清晰的，不像OpenClaw那样逻辑混乱错综复杂，而且不会莫名其妙卡住。

NousResearch（Hermes的开发团队）最近也上新了GUI版本，不过其实就是把CLI作为后端，就是给CLI套了一层GUI的壳，所有功能和扩展性和CLI版本保持一致。装GUI送CLI，用啥都行。

此外，Hermes提供的安装方式也是一套全包，缺什么就会以自己装一个自管理包，不会影响到其他程序。当然即使是自管理包也会被添加到Path，在其他地方也可以用，需要注意一下，否则以后自己配置了相对应的环境可能还会继续用Hermes配置的版本。

[官网](https://hermes-agent.nousresearch.com/docs)已经说得非常详细了，无需多言。

遇到的瑕疵：

1. 模型ID依赖自发现功能，只在第一次输入自定义端点的时候才可以输入模型名称，后续只能在列表里面选。如果不慎设置了聚合端点，那么下次换模型基本就不可能了，因为需要在几千个模型里面挑，输入准确的序数。
2. 浏览器默认无头，但是跟它对话可以让它弄出一个有头的出来，还行。

还有的就想不起来了。整体来说我用它做一些自动化的任务，应付一些破事，而不是用它生产具体的成果。但是毕竟它的扩展性在那。

## 国产新星Trae Work

这应该是我用的第一款专注型AI Agent，原因很简单：我之前用了Trae IDE，看到了它的宣传。我的第一感觉是“这不就是OpenClaw的套壳版么”，后来又发觉“这个也太会跟进Claude Code的脚步了”。

在这里根本也不用写什么教程，因为完全图形化的界面几乎没有任何上手门槛，自定义模型的设置方式几乎没有复杂的逻辑，都用图形界面写好了。

功能上它和Claude Code的区别也很小，Cowork对应Work，Code也叫Code，Design是Claude Code推出Design后立马加的（不过Claude Code的Design功能没法通过自定义Gateway使用），Trae Work没有Chat板块，所以不好跟它闲聊。

还有一个亮点就是支持手机端操作，省去了配置一堆第三方即时通信服务的麻烦，不过毕竟还是要走他们自己的服务器走一遍，可能会有些隐私担忧。

我以前用它完成了很多事情（一开始是当成图形界面完善、生态更丰富的Hermes/OpenClaw用的），当然主要是完成水课的作业和编写复习讲义，在这方面Trae Work给了我很满意的结果。不过现在有了Claude Code，优势就很难说了，大部分我肯定都会给Claude Code做，Design功能是我不放弃的理由，但是如果Claude Code哪天把Design也端上Gateway了，可能我真的就会放弃Trae Work了。

## 小透明之OpenCode

这个太透明了，我本来就是打算用OpenCode，而不是Claude Code的，但是后来成功克服困难用上后者就感觉前者根本没有用武之地，因此装了也不知道要干啥，目前我没有用它做过任何事情，而且体验似乎也不是很好。

## 已经抛弃的Trae IDE

Trae IDE是我接触的第一款AI辅助工具，打开了我接触AI“高效落地”的大门。卖点就是里面内置AI助手，最引以为傲的则是SOLO模式，一开始感觉神乎其神（好像在推出前确实是一个很新颖的概念），但是此后的Claude Code做的也是类似的事。后来SOLO就变成了Trae Work去抢更下沉的市场了。

因为有个AI侧边栏，所以直接打开项目目录就可以问AI相关的问题，也可以不用离开编辑器让AI修改代码，这也是我没有放弃它的原因。

但是现在情况逆转了，先不说Cursor这类竞品早就有了（不过这个是收费的，我也不太想用），我现在打开VS Code发现AI侧边栏也早有了，甚至连对标SOLO的Agent模式也上线了，而且也支持自定义端点，我完全没有留着Trae IDE的理由。

## 其他用过的但最终不继续使用的AI工具

其实只有两款：Antigravity IDE和Antigravity。

一开始Public Beta的时候就使用了，最吸引我的功能是自带一个浏览器插件，可以支持很多CDP做不到的事情，因为当时要写个爬虫所以就用了。但除此之外它也没有吸引我的理由，因为IDE我已经有很多个了，而后来Antigravity直接更新成了一个对话框形式（是的，即使以前装了老版，那么大一个IDE直接没了，想用还需要重下），而且它并不支持自定义端点，Quota也不行像刚公测那会那么Generous，所以之后也就没用过。