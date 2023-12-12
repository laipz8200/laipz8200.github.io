---
title: Python 的可变 Exception
tags:
  - python
  - exception
date: 2023-12-12 23:51:16
---

> 今天经同事提醒，发现踩了个和 Exception 相关的坑，特此记录

<!-- more -->

## 起因

最近使用 FastAPI 做了各种各样的业务系统，每个系统中都会有一些异常处理相关的代码，我采用的方式大致是这样：

1. 在 exceptions module 中定义各种自定义的异常，比如 `PermissionDeniedError` 。
2. 在代码中将对应的异常抛出。
3. 使用 FastAPI 的 error_handler 处理对应类型的异常。

在这个过程中，为了避免总是写相同的错误提示，我在 exceptions 中增加了一些比较常用的错误，比如 `UserNotFoundError = RecordNotFoundError("user not found")` 。

就在这里出现了问题。

## 错误

在这里，一个被我忽略的事实是 「Python 中的 Exception 是 **可变的**」，而我将它当成了不可变的常量来使用。

根据 [raise 表达式的文档](https://docs.python.org/zh-cn/3/reference/simple_stmts.html#the-raise-statement) ，每当异常被抛出，Python 都会创建一个 Traceback 对象并关联到异常的 `__traceback__` 属性，试想一下当我们每次抛出的都是同一个异常对象时会发生什么？——这个对象中记录的信息越来越多，并且永远无法被回收！

## 测试

尝试编写一小段代码来验证这个问题：

```python
def main():
    CustomError = Exception("this is a custom error")
    for _ in range(1000):
        try:
            raise CustomError
        except Exception:
            pass
    cnt = 0
    p = CustomError.__traceback__
    while p is not None:
        cnt += 1
        p = p.tb_next
    return cnt


if __name__ == "__main__":
    pts = main()
    print(pts)
```

每次 `CustomError` 被抛出，都会有一个新的 `trackback` 对象被创建出来，前一个 `traceback` 对象会被设置为新对象的 `tb_next` —— 一个链表。

最后我们遍历这个链表并统计其中包含了多少 `traceback` 对象：在这个例子中， `CustomError` 被抛出了 1000 次，所以结果也是 1000 个。

可想而知，因为这 1000 个对象都被关联到 `CustomError` ，而 `CustomError` 的生命周期与程序本身相同，因此这些对象永远不会被回收掉。实际情况中，随着 `CustomError` 不断被抛出，链表中的对象也会越来越多。

## 解决方案

推荐使用传统的方式处理异常 —— 每当需要抛出时重新创建一个异常实例。

如果需要复用，比如我们有很多异常中的提示信息都是一样的，这里可以选择以下两种方案：

1. 创建一个用于生成异常实例的函数，需要抛出异常时调用它。
2. 在自定义异常时重写构造函数，然后直接抛出异常类。

	> `raise` 会将第一个表达式求值为异常对象。 它必须为 `BaseException` 的子类或实例。 如果它是一个类，当需要时会通过不带参数地实例化该类来获得异常的实例。
