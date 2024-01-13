---
title: 在 GitLab CI 中配置 pip 缓存
category: programming
tags:
  - ci
date: 2023-12-22 20:59:13
---

> 一个多月前，为了规范我们组的开发流程，我为组员们安排了一套新的工作方式，借助 GitLab 的一些功能来帮助开发更可靠的软件。其中就包括 GitLab CI，然而随着测试用例的增多，运行一次 Pipeline 的时间从最初的不到 1 分钟增加到了需要 3 分钟以上，已经开始反过来影响开发的速度了。

<!-- more -->

究其原因，我在最初配置 GitLab CI 的时候并没有使用缓存，而我们的项目依赖不断增加，再加上偶尔的网络波动，使得依赖的安装时间成为一个不可忽视的问题。

为了拯救组员们的开发体验，我花了大概半天时间阅读手册，并解决了之前没有处理的一些问题。

## 为 Python 依赖项设置缓存

我们的 GitLab runner 运行在 Docker 环境下，在没有专门配置的情况下，它默认使用 Docker volume 处理缓存，默认的缓存目录为 `/cache`。

Volume 的名称很复杂，但是可以看到其中包含了 Project ID，根据手册中的说法，不同的项目之间是不可以共享缓存的。

（我想这里应该有截图，可我忘记了，感兴趣的读者可以在自己运行 Runner 的机器上执行 `docker volume ls` 看看）

对于 Python 的依赖，手册中专门提到了推荐的做法——共享 pip 的缓存目录。配置如下：

```yml
default:
  image: python:3.11
  cache:
    paths:
      - .cache/pip
  before_script:
    - python -m venv .venv
    - source .venv/bin/activate

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

# your stage and job configurations...
```

我们并没有将下载好的依赖直接保存，而是利用了 pip 本身的缓存，当安装相同的依赖时，pip 缓存会被命中，避免重复进行下载。当项目的依赖发生变化时，我们也不需要自己去处理缓存。

## 减少无用的任务

引入 CI 过去了一个多月，我发现了一些明显的资源浪费：有一些组员习惯于多次将代码提交到自己的分支上，最后再进行合并，在这种情况下，我们并不需要再每一次提交之后都运行 Pipeline。

根据手册，可以通过以下设置将 Pipeline 的执行条件限定到每一个 Merge Request：

```yml
my_job:
  stage: test
  script:
    - pytest --cov=datashop --cov-report term-missing tests/
  only: [merge_requests]
```

这样，只有当有 Merge Request 创建或更新时，Pipeline 才会被执行。

