---
title: 珍惜时间，Ruby on Rails API 开发指北
category: Programming
tags:
  - ruby
  - rails
  - active record
date: 2022-05-19 23:43:04
---

> 继上次简单写了一些关于 Event Sourcing 的内容后，感觉进入了一段情绪低迷期，决定学一些和找工作无关的东西。
>
> 实际上，绝大多数项目并不需要使用领域驱动设计，也不需要支持高并发，在这种情况下，为了节省宝贵的时间，我们可以使用一种简单且有效的模式：活动记录 (Active Record)。

<!-- more -->

## Active Record Pattern

活动记录的特点是 **一个模型类对应关系型数据库的一个表，一个实例对应数据库中的一行记录**。活动记录一般兼有 ORM 的功能，但并不是简单的 ORM，目前 Python 中常用的 ORM，如 SQLAlchemy 和 Django ORM 都实现了 Active Record 的功能。

关于 Active Record 模式的著名案例是 *全栈 Web 开发框架 Ruby on Rails*[^1]，今天我们就用它来实现一个简单的接口，感受一下它的魅力。

## 需求

我们实现一个不复杂但是很常见的需求：呈现一个树形结构。树的每一个节点 (node) 都拥有一个名字，最后需要提供一个可以展示整个树的 JSON 结构的 Web API。

它可能长这个样子：

```yaml
- root node
    - child node 1
    - child node 2
......
```

## 分析

有很多种在关系型数据库中保存树形结构的方式，但各有优缺点，在这个案例中，我们选择最简单的一种：自关联。

具体来说，我们为每个节点定义一个 `parent_id` 字段，该字段保存节点父亲节点的唯一标识。

## 实现

### 创建项目

首先，我们跳过环境的安装教学，毕竟我并不打算做一个 「手把手教你xxx」 系列。

简单贴一下 macOS 下使用 Homebrew 安装所需环境的几条命令：

```bash
brew install rbenv
rbenv install 3.1.2
rbenv global 3.1.2
gem install rails
```

安装好 Rails 之后，使用如下命令创建项目：

```bash
rails new demo --api
cd demo
```

结尾的 `--api` 代表这是一个单纯的 API 项目，Rails 不会加载与前端相关的组件。

进入项目目录后，打开 `Gemfile` 文件，找到如下行：

```ruby
# Build JSON APIs with ease [https://github.com/rails/jbuilder]
# gem "jbuilder"
```

取消第二行的注释，`jbuilder` 是一个序列化 JSON 内容的工具，它实现了一个简单好用的 DSL ，可以减少我们的工作。

### 创建脚手架

回顾一下我们需要的模型，对它的描述如下：

- 节点 (node) 有自己的名字。
- 节点拥有一个父亲节点 (parent)，如果父亲为空，则该节点为根节点。
- 节点拥有许多子节点 (children)。

我们使用 Rails 的生成工具创建一个包含 CRUD 操作的模板：

```bash
rails generate scaffold node name:string parent:references
```

命令的前半部分告诉 Rails 我们需要生成一个名叫 `node` 的脚手架，后半部分指明我们的模型包含两部分内容：
一个字符串类型的 `name` 和一个引用类型 `parent` 。

顺利的话，你将会看到以下输出：

```bash
      invoke  active_record
      create    db/migrate/20220519154951_create_nodes.rb
      create    app/models/node.rb
      invoke    test_unit
      create      test/models/node_test.rb
      create      test/fixtures/nodes.yml
      invoke  resource_route
       route    resources :nodes
      invoke  scaffold_controller
      create    app/controllers/nodes_controller.rb
      invoke    resource_route
      invoke    test_unit
      create      test/controllers/nodes_controller_test.rb
      invoke    jbuilder
      create      app/views/nodes/index.json.jbuilder
      create      app/views/nodes/show.json.jbuilder
      create      app/views/nodes/_node.json.jbuilder
```

Rails 为我们创建了两个 Active Record 类，分别表示数据库迁移和模型，并创建了针对模型的单元测试。

Rails 还为我们添加了资源 `nodes` 的路由。

最后，Rails 为我们生成了 `nodes` 的 Controller ，对应的单元测试，以及 `jbuilder` 视图。

### 修改模型

自动生成的脚手架存在一些问题，首先，现在许多项目不允许在数据库中创建外键，虽然我并不能理解这种不分实际情况通通禁止的做法，但我们可以稍作修改，让 Rails 不要创建外键。

在 `db/migrate/` 目录下，找到生成的 `.rb` 文件，将其中的 `foreign_key: true` 修改为 `foreign_key: false`，这将会告诉 Rails 在该列建立一个简单的索引而不是外键约束。

顺便，由于我们的根节点不会有父亲，所以将 `null: false` 改为 `null: true` 来允许空值。

第二个问题是我们使用自关联，但指定了引用 `parent` ，我们需要告诉 Rails 它是什么。

打开 `app/models/node.rb` 文件，修改模型中的关系定义如下：

```ruby
class Node < ApplicationRecord
  belongs_to :parent, class_name: 'Node', optional: true
  has_many :children, class_name: 'Node', foreign_key: 'parent_id', dependent: :destroy
end
```

我们通过 `class_name` 告诉 Rails `parent` 使用哪一个模型，并通过 `has_many` 添加了反向关系，指定使用的字段名为 `parent_id` 。最后，我们规定在父亲节点被删除时，子节点也会被删除。

现在，我们可以使用 `rails db:migrate` 生成数据库。默认的数据库是 SQLite ，使用 Rails 提供的工具进入数据库查看生成的表结构（当然你也可以自己使用 `sqlite3` 查看，数据库在 `db/development.sqlite3` 文件中）：

```bash
rails dbconsole
sqlite> .schema nodes
CREATE TABLE IF NOT EXISTS "nodes" ("id" integer PRIMARY KEY AUTOINCREMENT NOT NULL, "name" varchar, "parent_id" integer, "created_at" datetime(6) NOT NULL, "updated_at" datetime(6) NOT NULL);
CREATE INDEX "index_nodes_on_parent_id" ON "nodes" ("parent_id");
sqlite> .exit
```

可以看到， Rails 创建了索引而不是外键约束，并默认添加了 `id` `created_at` `updated_at` 字段。

### 预设数据

为了方便接口测试，我们先生成一些预设数据。

打开 `db/seeds.rb` 文件，添加三条记录：

```ruby
root = Node.create name: 'root'
Node.create name: 'child1', parent: root
Node.create name: 'child2', parent: root
```

然后使用 `rails db:seed` 让它生效，这样我们的数据库中就有了三条记录。

### 查看接口

修改接口之前，我们先来看一看脚手架生成的方法是什么样子的。

使用 `rails server` 启动服务，然后访问 `http://localhost:3000/nodes.json`，你应该会看到刚才添加的三条记录：

```bash
$ http :3000/nodes.json  # use httpie
HTTP/1.1 200 OK
... # HTTP HEADERS

[
    {
        "created_at": "2022-05-19T15:18:07.798Z",
        "id": 1,
        "name": "root",
        "parent_id": null,
        "updated_at": "2022-05-19T15:18:07.798Z",
        "url": "http://localhost:3000/nodes/1.json"
    },
    {
        "created_at": "2022-05-19T15:18:07.803Z",
        "id": 2,
        "name": "child1",
        "parent_id": 1,
        "updated_at": "2022-05-19T15:18:07.803Z",
        "url": "http://localhost:3000/nodes/2.json"
    },
    {
        "created_at": "2022-05-19T15:18:07.804Z",
        "id": 3,
        "name": "child2",
        "parent_id": 1,
        "updated_at": "2022-05-19T15:18:07.804Z",
        "url": "http://localhost:3000/nodes/3.json"
    }
]
```

我们的目标就是将它修改为树状结构。

### 修改接口

为了修改接口，我们首先需要知道 Rails 是怎么处理我们的请求的。脚手架模板为我们生成了 MVC 中需要的全部组件以及一些路由，让我们使用 `rails routes` 命令查看路由，这里只列出第一条：

```bash
nodes GET /nodes(.:format) nodes#index
```

这条路由就是我们刚刚访问的地址，最后的部分由 `Controller#Action` 组成，也就是说，我们的目标是 `nodes_controller` 中的 `index` 方法。

打开 `app/controllers/nodes_controller.rb` 文件，找到 `index` 方法，将它修改为：

```ruby
def index
  @nodes = Node.where parent: nil
end
```

这样，接口将只返回根节点：

```bash
$ http :3000/nodes.json
HTTP/1.1 200 OK
...

[
    {
        "created_at": "2022-05-19T15:18:07.798Z",
        "id": 1,
        "name": "root",
        "parent_id": null,
        "updated_at": "2022-05-19T15:18:07.798Z",
        "url": "http://localhost:3000/nodes/1.json"
    }
]
```

`@nodes` 的渲染将委派给 `jbuilder` ，我们可以在 `app/views/nodes/_node.json.jbuilder` 文件中找到它的模板，为它添加一个 `children` 属性：

```ruby
json.extract! node, :id, :name, :parent_id, :created_at, :updated_at
json.url node_url(node, format: :json)
json.children do
  json.array! node.children, partial: 'nodes/node', as: :node
end
```

这段代码使得 `node` 递归地访问自己的孩子节点，最终渲染出整个树形结构，这样我们就完成了接口的修改。

最后看一看修改后的接口返回：

```bash
$ http :3000/nodes.json
HTTP/1.1 200 OK
...

[
    {
        "children": [
            {
                "children": [],
                "created_at": "2022-05-19T15:18:07.803Z",
                "id": 2,
                "name": "child1",
                "parent_id": 1,
                "updated_at": "2022-05-19T15:18:07.803Z",
                "url": "http://localhost:3000/nodes/2.json"
            },
            {
                "children": [],
                "created_at": "2022-05-19T15:18:07.804Z",
                "id": 3,
                "name": "child2",
                "parent_id": 1,
                "updated_at": "2022-05-19T15:18:07.804Z",
                "url": "http://localhost:3000/nodes/3.json"
            }
        ],
        "created_at": "2022-05-19T15:18:07.798Z",
        "id": 1,
        "name": "root",
        "parent_id": null,
        "updated_at": "2022-05-19T15:18:07.798Z",
        "url": "http://localhost:3000/nodes/1.json"
    }
]
```

完成！可以放下工作去享受生活了！

## 结语

这并不是一篇正经的教程文章，在 Ruby on Rails 的时代，人们遵从「宁花机器一分，不花程序员一秒」的 UNIX 哲学，希望大家也可以时常放下工作，享受生活。

最后，如果你对 Ruby on Rails 产生了兴趣，可以去它的官方网站学习。

[^1]: [Ruby on Rails](http://www.rubyonrails.org/)
