---
title: 使用关系型数据库实现事件源模式
tags: event sourcing
date: 2022-05-16 22:35:51
---


关系型数据库是日常开发中最常用的数据库类型，本文记载使用关系型数据库实现事件源模式的要点和一些问题。

要阅读本篇文章，你可能需要先自行了解 *关系型数据库* 、 *事件源模式* 和 *领域驱动设计* 相关知识。

## 事件存储

首先考虑事件的存储，实际上只需要四个字段， DDL 如下（以 PostgreSQL 为例）：

```sql
CREATE TABLE events
(
    id        SERIAL      PRIMARY KEY,
    stream_id VARCHAR(50) NOT NULL,
    version   BIGINT      NOT NULL,
    data      TEXT        NOT NULL,
    UNIQUE (stream_id, version)
);
```

其中， `stream_id` 是实体或聚合的唯一标识，也可以把这个字段叫做 `name` 或者其他你喜欢的名字， `version` 是事件的版本，由于事件是只追加的，版本会不断增加，并且对于同一个实体或聚合来说，版本号不会有重复。

`id` 是事件的标识符，由数据库生成即可。

使用这种简单的结构，我们已经可以实现两种事件源模式需要的功能了。首先是获取事件流：

```sql
SELECT id, stream_id, version, data FROM events 
    WHERE stream_id = :stream_id ORDER BY version ASC;
```

根据有序的事件流，我们可以重建出一个实体或聚合的最新状态，这是事件源模式的关键功能。

第二项功能是记录事件，使用：

```sql
INSERT INTO events(stream_id, version, data)
    VALUES(:stream_id, :current+1, :data);
```

需要注意的是，这里的 `current+1` 仅仅表示插入数据时应该让版本增加 1，而不是可以直接在 SQL 语句中这样写。
考虑到主流关系型数据库的默认隔离级别，这种写法会产生一些并发问题。（对于 PostgreSQL ，可以指定隔离级别为 *Repeatable Read* ）

## 投影 (PROJECTION)

事件源模式的另一个问题是无法像传统表结构那样进行过滤，我们使用投影来解决这个问题。

投影是通过提前聚合事件形成的一种视图，我们既可以同步更新投影，也可以异步更新投影。在异步更新时，系统将获得更高的性能，但会存在一定延迟，这时的系统实现的是 *最终一致性* 。

实现最终一致性并不是一件容易的事，在关系型数据库中，为了简单，我们可以先同步更新投影，这种情况下应该利用关系库的一致性保证，即使用 **事务**。

每个事务中我们做两件事：

- 插入新的事件
- 更新投影

因为事件存储是整个系统的唯一数据源，所以投影可以被轻易放弃、调整结构，只需要重放所有事件即可生成新的投影用来适配客户端的查询需求。

## 快照 (SNAPSHOT)

快照解决事件源模式中的另一个问题：当事件数量不断增加时，重建一个实体或聚合的成本也会不断增加。

可以将快照视为一种特殊的投影，与投影相比，快照最大的不同点是它需要保存版本信息，这样我们就可以在重建时先查询快照，然后仅将快照对应版本之后发生的事件应用在实体或聚合上。

和投影一样，使用关系库时，我们可以在同一个事务中更新快照，不过快照并不需要频繁更新，通常每 50-100 个版本更新一次即可[^1]。

## 滚动升级

引入快照和投影之后，我们失去了一些灵活性，其中很重要的一点就是要如何修改投影和快照的结构。

当条件允许时，最简单的方式是停止服务，删除旧的投影和快照表，然后重放事件并生成一份新的表。然而生产服务并不会允许我们这样做，因此当需要升级时，我们应该考虑以下步骤：

1. 创建新的投影和快照表。
2. 修改程序，当新的事件写入时，旧表和新表都应该被更新，当然，只有当实体已经存在于新表中时，它才会被更新。
3. 如果有新的实体或聚合被创建，它会同时被保存到两个表中。
4. 最后，我们重放所有已有的事件，并将它们聚合后的结果写入新表中。
5. 当以上步骤完成，新旧两个版本的数据会同步更新，此时我们可以删除旧表和相关逻辑。

## 异步

最后，让我们考虑一下性能问题。当系统成长到一定规模时，我们会很自然地想到采用异步更新投影的方式来改善性能，事件源模式最终通常会引入消息队列（甚至是 Kafka ）。

然而在关系库和消息队列之间并没有一种方式可以维持一致性，无论在数据库事务的哪个阶段发送消息，都可能产生一些副作用。

由于使用不同组件，我们没有方法可以彻底消除副作用，但可以通过一些手段来实现最终一致性：

1. 创建一张队列 (queue) 表 ，用来存放新生成的事件。
2. 当有新的事件生成时，同时写入事件存储和队列。
3. 使用一个 **单线程的** 程序轮询队列，分批取出其中的事件并发送到消息队列，然后从队列中删除这些事件。
4. 创建事件监听器，从消息队列中获取事件并更新投影和快照。

注意以上操作中的第 3 步，消息发送后，删除操作依然有可能失败，此时，同一事件可能被多次发送到消息队列中。
不过问题不大，由于事件存在版本，消费者可以轻松找出不需要处理的事件并丢弃它们。


## 参考链接

[Implementing event sourcing using a relational database](https://softwaremill.com/implementing-event-sourcing-using-a-relational-database/)

[^1]: [实现领域驱动设计](https://book.douban.com/subject/25844633/)