---
title: 注册表模式的 Golang 实现
tags:
  - registry pattern
  - 注册表模式
  - golang
  - 企业应用架构模式
category: Programming
date: 2023-03-01 00:41:59
---


> 年终总结时挖坑说要写一篇关于注册表模式的文章，今天填坑。
>
> 注册表模式，简单来说就是把需要在许多地方使用的东西注册到一个固定的位置。

<!-- more -->

## 背景与需求

在几个项目中应用存储库模式后，我遇到了一个问题：

业务系统中通常会遇到很大的聚合，一个聚合根或许会引用 3、4 个其他实体，它们被持久化到数据库的不同表中。如果每一次都要全部加载，会产生许多不必要的网络开销。

为了避免无谓的数据库查询，我们可以使用按需加载的形式，仅在该属性被访问时才把它加载到主存中。

然而接下来的问题是：加载数据需要使用到存储库，我该从什么地方获得存储库呢？

在之前的项目中，我一直使用依赖注入的方法将存储库实例注入到每一个需要使用它的服务中，然而这回需要使用存储库的从服务变成了聚合，总不能让每个聚合对象保持一个对存储库的引用吧？

使用注册表可以有效解决这个问题。

## 注册表

当你需要定位某一对象时，通常会从另一个与其有关联关系的对象入手，通过关联来查找目标对象。但是在某些情况下，可能并没有合适的初始对象。

注册表是一个众所周知的对象，看起来就像一个全局对象，你可以通过它找到公共的对象和服务。

## 实现注册表模式

在 Go 语言中，我们通常不需要考虑进程级别的并发，所以在下面的案例中我会创建一个进程作用域的注册表（即其他进程无法访问的）。

在创建进程作用域的对象时，通常使用 *单例模式* ，在 Go 语言中，只需要简单地写下：

```go
var registry IRegistry

type IRegistry interface {
    Repository() IRepository
}
```

这个案例中，我们的注册表接口只有一个方法，用于返回一个存储库接口 —— 关于为什么存储库需要一个单独的接口，我准备放在以后再单独写一篇文章来介绍。

需要注意的是变量 `registry` 是非导出的，这意味着其他模块无法使用它，我倾向于编写一个函数用于控制对它的访问：

```go
func Registry() IRegistry {
    return registry
}
```

由此，我们就可以在任何地方写下形似 `Registry().Repository().Users().Get(ctx, userID)` 的代码了。

我们还需要一个具体实现和两个额外的函数，分别用于创建和加载注册表对象：

```go
type registryImpl struct {
    repo IRepository
}

func (r registryImpl) Repository() IRepository {
    return r.repo
}

func InitRegistry(repository IRepository) IRegistry {
    return registryImpl{repository}
}

func SetRegistry(r IRegistry) {
    registry = r
}
```

实际项目中，我们使用 `wire` 等依赖注入工具，配合 `InitRegistry()` 方法在程序启动时创建一个新的注册表，接着使用 `SetRegistry()` 方法设置它，之后即可在代码中的任意位置引用这个注册表，并借助它找到我们想要的一切服务（如存储库）。

由于注册表可能在任何地方使用到，因此我建议将它放在系统的最下层，即领域层。

## 案例

前段时间，我将平时开发常用的代码写成了模板，其中使用到了注册表模式，仓库地址在 [gin-template](https://github.com/laipz8200/gin-template) ，欢迎参考。
