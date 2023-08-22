---
title: Python 和 Pydantic 中的时区问题
tags:
  - python
  - timezone
  - pydantic
category: Programming
date: 2023-08-23 01:23:03
---


> 记录一个关于时区的小问题。

<!-- more -->

## Problem

最近经手的项目里有这样一个场景:

> 为用户设置一个有效期，在用户登录时将过期的用户调整为不可用并拒绝登录。

为了实现这个需求，我在用户模型中增加了一个字段，大概是这样

```python
class User(Base):
    # ...
    validity_period: Mapped[Optional[datetime]]
    # ...
```

对应的 Pydantic 结构也增加这样一个属性

```python
class UserDetail(BaseModel):
    # ...
    validity_period: Optional[datetime]
    # ...
```

然而这样一个功能却遇到了问题，测试同学反馈说保存成功后，有效期总是比设置的时间提前了 8 小时。

相信每一位东八区的开发在听到这个描述时都会有十足的把握: 是时区的问题。然而当我尝试复现问题时，却发现情况并没有那么简单。

首先，前端同事调用接口时传递的是一个 `Unix 时间戳` ，也就是说并没有携带时区信息， `Pydantic` 非常贴心地将其转换成了 `datetime` 类型，并跟随业务代码将这个 `UTC 时间` 保存到了数据库中。

这作为一位后端开发，保存 `UTC 时间` 也是我期望的结果，这一步并没有什么问题。然而在之后的读取时，程序返回了这样的结构: `2024-01-01T18:00:00` 。

前端同学在像我展示这个错误的时候，使用 `date = new Date('2024-01-01T18:00:00')` 得到了一个错误的时间。

## Reason

我盯着眼前这一串代表时间的字符，突然发现它其实并不是一个严格的 ISO-8601 时间，相较于标准化的时间，它缺少了结尾的时区信息。
在补充上结尾的时区信息后，`date = new Date('2024-01-01T18:00:00Z')` 显示了正确的结果。

那么是什么导致了这个现象呢？

由于 Python 并不强制 datetime 类型携带时区信息， Pydantic 在将输入的时间戳转换为 datetime 时也没有自作聪明地为其添加额外信息。
在 SQLAlchemy 中，Datetime 类型默认也不处理时区信息。
因此，程序从数据库读取出的 datetime 类型的时间中是不包含时区信息的， Pydantic 输出时将其转换为类似 ISO-8601 格式的字符串，但不包含尾部的时区信息，反而使得错误不易发现。

## Solution

知道了问题，解决起来就非常容易了，我不想贸然在 SQLAlchemy 中设置时区信息，因此选择先在 Pydantic 进行转换时添加这一信息。
以下是修改后的 Pydantic 模型:

```python
class UserDetail(BaseModel):
    # ...
    validity_period: Optional[datetime]
    # ...

    @validator
    def validate_validity_period(cls, v):
        if isinstance(v, datetime):
            v = v.replace(tzinfo=timezone.utc)
        return v
```

为 `validity_period` 添加了时区信息之后，序列化正确返回了包含时区部分的 ISO-8601 格式，前端的同事也能够正确解析时间了。
