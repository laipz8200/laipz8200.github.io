---
title: 制作一个简单的 Alfred Workflow
date: 2022-07-10 19:40:05
tags:
  - alfred
  - workflow
  - macos
  - ruby
---


> 在论坛或群聊中给人发送一些内容时，有时需要简单隐藏一下内容，通常我会使用 base64 对内容进行编码。
>
> 之前一直使用 Chrome 插件提供的功能来完成，最近突然想起了 Alfred Workflow 更适合做这件事。

Alfred and workflow
-------------------

Alfred 是 macOS 平台上的一个工具类 App ，提供全局搜索和启动器等功能。

此外，它拥有一个叫做 「Workflow」 的功能，可以通过自定义的方式实现自己的工作流。

之前我一直使用一个叫做 encode 的 workflow 来快速进行编解码操作，然而它的作者似乎已经没有在维护了，随着 macOS 的更新，系统也不再默认提供它所依赖的 php runtime 了。

在到处都找不到好用的工具后，我决定自己来实现这个功能。

Create a simple
---------------

虽然听起来很复杂，但自己创建一个 workflow 其实并不困难， Alfred 为了让普通人也能够创建 workflow ，提供了相当多方便的操作和直观的界面。

不过对于我们这种简单的应用来说，只需要使用其中两项功能就足够了。

Input: script filter
--------------------

首先在空白的 workflow 上右键选择 `Inputs > ScriptFilter` 。

我们可以在上方设置触发功能使用的关键词，提示用语等内容，并在下面的 `Script` 区域对输入进行处理。

Alfred 使用 JSON 格式来控制输出，我们只会用到以下最简单的几项:

- title: 显示内容的标题
- subtitle: 副标题
- arg: 传递给下一步的参数

将以上内容的列表作为 `items` 的值输出即可让它们显示在 Alfred 的结果中了。

考虑到我暂时只需要 base64 编码和解码的功能，我选择最简单的 Ruby 实现:

```ruby
query = ARGV[0]

require 'json'
require 'base64'

b64 = Base64.encode64(query)

results = {items: [
	{ title: b64, subtitle: 'base64', arg: b64 }
]}
print results.to_json
```

解码同理，使用 `Base64.decode64` 即可。

Output: copy to clipboard
-------------------------

实现功能后，我们需要考虑如何使用结果，最简单的方法当然是将结果复制下来。

右键选择 `Outputs > Copy to Clipboard` ，然后将之前创建的 ScriptFilter 连过去就可以。

![workflow](workflow.png)

![encode](encode.png)
