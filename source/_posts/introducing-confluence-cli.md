---
title: 我写了一个 Confluence CLI
category: programming
tags:
  - confluence
  - cli
  - agent
  - rust
date: 2026-05-15 05:18:00
---

最近写了一个新的小工具：[confluence-cli](https://github.com/laipz8200/confluence-cli)。

它是一个面向 Confluence Cloud 的命令行工具。它围绕几类常见的 Confluence 工作流设计：列出空间、搜索页面、读取页面、创建页面、更新页面，让这些操作对人和 Agent 都足够可预测。

<!-- more -->

## 为什么需要一个新的 CLI

我写这个工具的起点很简单：Confluence 实在太难用了。

它的网页交互极其繁琐。创建、查找、编辑页面，经常要在层层菜单和页面之间反复切换。页面加载速度也很慢，等编辑器、目录、侧边栏和搜索结果出现，本身就会消耗耐心。

知识库本该帮人沉淀信息。实际使用中，Confluence 的日常交互常常变成额外负担。既然我已经在很多工作场景里使用 Agent，把这些机械交互交给 Agent 就很自然：我描述要找什么、要改什么、要写什么，Agent 负责完成 Confluence 里的读写动作。

于是 `confluence-cli` 的目标变得清楚：给 Agent 一个稳定、保守、可审阅的 Confluence 操作入口。它需要覆盖最常见的搜索、读取、创建和更新页面流程，并让所有写入先经过 dry-run，方便人在真正执行前检查计划。

## 安装和初始化

安装最新 release：

```bash
curl -fsSL https://raw.githubusercontent.com/laipz8200/confluence-cli/main/install.sh | sh
```

安装脚本默认会把二进制文件放到 `~/.local/bin`。目前提供 Linux x86_64、Linux arm64、macOS x86_64 和 macOS arm64 的 release 二进制文件。

第一次使用时先初始化配置：

```bash
confluence-cli config init
```

初始化过程分成几步。

首先，按提示输入 Confluence site URL、邮箱和 API token。工具会用这组凭据连接 Confluence，确认它们可以正常访问。

接着，工具会列出当前账号可以访问的所有 space。你可以从列表里选一个默认 space。这个步骤看起来很小，实际是在告诉工具和 Agent：之后默认关注哪个知识库空间。

最后，`config init` 会询问是否安装配套的 Agent Skills。直接按 Enter 就会安装，输入 `n` 可以跳过。

## Skills 的作用

CLI 解决的是“怎么和 Confluence 交互”的问题，Skills 解决的是“Agent 应该怎么使用这个 CLI”的问题。

安装 Skills 后，Agent 会获得一份操作说明。它会知道遇到 Confluence 任务时优先使用 `confluence-cli`，通过命令读取、搜索、创建和更新页面。涉及写入时，它会先生成 dry-run 计划，把计划展示或总结给你，再等待明确批准后执行。

这对我来说很重要。我的目标是把 Confluence 的繁琐网页操作交给 Agent，同时保留写入前的人类确认。Agent 可以替我完成页面查找、内容整理和慢页面等待；真正改知识库之前，它需要把计划拿出来让我看一眼。

手动安装 Skills 也很简单：

```bash
npx skills add laipz8200/confluence-cli --skill confluence-cli
```

## 在 Agent 里怎么用

初始化和 Skills 安装完成后，就可以直接用自然语言让 Agent 操作 Confluence。

比如：

1. 帮我在 Confluence 里搜索和 deploy 有关的页面。
2. 读取这个页面并总结重点：`123456`。
3. 把下面这段 Markdown 发布到默认 space，标题叫“Release Notes”。
4. 更新页面 `123456`，把这份文档内容写进去。

读操作会直接返回结果。创建或更新页面时，Agent 会先运行 dry-run，并把准备写入的目标、标题和内容摘要告诉你。你确认后，它才会继续执行真正的写入命令。

这样一来，我平时需要面对的就只剩下意图表达：我要找什么、要写什么、要改哪一页。Confluence 的页面跳转、搜索等待、编辑器加载，都可以交给 Agent 和 CLI 去处理。

## 链接

Landing Page：[confluence-cli](https://yeningxue.com/confluence-cli/)

GitHub 仓库：[laipz8200/confluence-cli](https://github.com/laipz8200/confluence-cli)
