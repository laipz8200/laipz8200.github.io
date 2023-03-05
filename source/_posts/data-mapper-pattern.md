---
title: 数据映射器和模型-数据源分离
date: 2022-07-07 22:53:32
category: Programming
tags:
  - data mapper
  - 数据映射器
  - 领域模型
  - 企业应用架构模式
---


> 使用 Python 进行 web 开发时，很多人都会首先接触到 Django 这样的框架，Django ORM 提供了一种极为简单明了的与关系型数据库交互的方式。但当业务逻辑变得复杂时，关系模型和对象模型的差异总会给项目带来一些麻烦，
>
> 在这种情况下，可以使用一种叫做数据映射器 (Data Mapper) 的模式将对象模型和数据源隔离，使他们能够各自演变，这样一来，软件便可以健康地发展下去。

<!-- more -->

设想这样一种常见的情况：你需要建立一个「用户」模型，「用户」具有「真实姓名」和「昵称」，出于隐私考虑，一些「用户」可能不太愿意提供自己的「真实姓名」。在「用户」提供了「真实姓名」时，软件需要保证「用户」同时提供「姓氏」和「名字」，不能只有其中一样。

聪明的你应该很快可以设计出这样一个模型:

```python
class User:
    first_name: str
    last_name: str
    nickname: str
```

乍一看，这个模型可以很好的工作，但我们很容易预见到在不久的将来，代码中将会充满着 `if first_name == '' or last_name == ''` 之类的判断，当这样的判断分布在程序各个地方时，代码很快就会变得难以维护，继而分崩离析。

出现这种问题的根源在于这个模型中丢失了需求里的一个重要概念（如果你注意到了我在上面那段话中的括号）「真实姓名」。让我们试着改进这个模型:

```python
class Name:
    first_name: str
    last_name: str


class User:
    realname: Name
    nickname: str
```

此时我们可以仅通过 `if realname is None` 完成同样的判断了。

很可惜的是，即使是这样一个简单的需求，在使用 Django ORM 时也会遇到问题，Django ORM 和 Ruby on Rails 一样，在模型层使用 「活动记录模式」 ，这种模式的特点便是模型的一个属性正好对应到关系型数据库中表的一列，在关系模型可以描述的领域中，这种方式能够极大简化模型和存储层的关系，使得软件开发变得非常迅速，可是当模型中包含复杂的领域知识（特别是涉及到继承、组合等关系）时，问题就会变得难以处理，为此我们需要采用一种更有效的方式，将领域模型和存储层隔离，「数据映射器」便是一种常用的模式。

数据映射器使用一个单独的映射器对象来处理领域模型和数据库内容的相互转换，将转换的过程封装，由此隔离领域层和存储层，在这种情况下，领域模型不需要知道存储层的任何信息，你可以轻易将 MySQL 替换为 PostgreSQL ，甚至换成 XML 或 MongoDB 等非关系型存储。

在开始实现映射器前，让我们先完善一下这个例子中需要使用的简单模型:

> 这里声明一下: 为了用尽可能少的代码进行说明，我不会采用严格的编码规范，也尽量不做提前设计，因此请不要试图将这里的示例代码直接在生产中使用。

```python
# data_mapper/domain/models.py
from datetime import datetime
from uuid import uuid4


class Name:
    def __init__(self, first_name: str, last_name: str):
        self.first_name = first_name
        self.last_name = last_name

    def __str__(self):
        return f"{self.first_name}·{self.last_name}"


class User:
    def __init__(self, id_: str, nickname: str, realname: Name | None):
        self._id = id_
        self.nickname = nickname
        self.realname = realname

    @property
    def id(self) -> str:
        return self._id

    @property
    def name(self) -> str:
        return str(self.realname) if self.realname else self.nickname

    def __str__(self):
        return "<User '%s'>" % self.name


def new_user(nickname: str, realname: Name | None) -> User:
    id_ = "USER" + datetime.utcnow().strftime("-%y-%m-%d-") + str(uuid4())[:8]
    return User(id_, nickname, realname)
```

为了给「用户」生成标识，额外添加了一个工厂方法，这会为我们之后的工作提供许多便利。

接下来我们设置数据库，同样，为了保证代码的简单，这里不会引入标准库以外的内容:

```python
# data_mapper/db/connections.py
import sqlite3

conn = sqlite3.connect(":memory:")


def dict_factory(cursor, row) -> dict:
    d = {}
    for idx, col in enumerate(cursor.description):
        d[col[0]] = row[idx]
    return d


conn.row_factory = dict_factory


def init_db(conn):
    ddl = """CREATE TABLE IF NOT EXISTS users(
    id         TEXT PRIMARY KEY,
    first_name TEXT,
    last_name  TEXT,
    nickname   TEXT NOT NULL
);"""
    conn.execute(ddl)


conn.set_trace_callback(print)
init_db(conn)
```

接下来终于可以开始编写我们的映射器了，我们首先实现一个能够将数据保存到数据库的方法:

```python
# data_mapper/db/mappers.py
from data_mapper.domain.models import User

from .connections import conn


class UserMapper:
    tablename: str = "users"
    columns: str = "id, first_name, last_name, nickname"
    insert_statement: str = (
        f"INSERT INTO {tablename}({columns}) VALUES ($1, $2, $3, $4)"
    )

    def insert(self, user: User) -> str:
        statement = self.insert_statement
        cursor = conn.cursor()
        try:
            cursor.execute(statement, self.insert_data(user))
            conn.commit()
            return user.id
        finally:
            cursor.close()

    def insert_data(self, user: User) -> tuple:
        if user.realname:
            return (
                user.id,
                user.realname.first_name,
                user.realname.last_name,
                user.nickname,
            )
        return (user.id, None, None, user.nickname)
```

我们可以运行一个简单的测试来看看它是否能够工作:

```python
from data_mapper.db.connections import conn
from data_mapper.db.mappers import UserMapper
from data_mapper.domain.models import Name, new_user

mapper = UserMapper()

user = new_user("Tom", Name("Tom", "Jackson"))
mapper.insert(user)

cursor = conn.cursor()
rows = cursor.execute("SELECT * FROM users;").fetchall()
print(rows)
cursor.close()
```

得到的结果是:

```
--------------------
CREATE TABLE IF NOT EXISTS users(
    id         TEXT PRIMARY KEY,
    first_name TEXT,
    last_name  TEXT,
    nickname   TEXT NOT NULL
);
BEGIN
INSERT INTO users(id, first_name, last_name, nickname) VALUES ($1, $2, $3, $4)
COMMIT
SELECT * FROM users;
[{'id': 'USER-22-07-07-11dd6b13', 'first_name': 'Tom', 'last_name': 'Jackson', 'nickname': 'Tom'}]

[Done] exited with code=0 in 0.025294 seconds
```

一切顺利，接下来我们实现根据主键获取「用户」的方法:

```python
class UserMapper:

    ...

    find_statement: str = f"SELECT {columns} FROM {tablename} WHERE id = $1 LIMIT 1"

    def find(self, id_: str) -> User:
        statement = self.find_statement
        cursor = conn.cursor()
        try:
            rs = cursor.execute(statement, (id_,)).fetchone()
            if rs is None:
                raise KeyError(f"{id_} is not exists")
            return self.load(rs)
        finally:
            cursor.close()

    def load(self, rs: dict) -> User:
        id_ = rs["id"]
        return self.do_load(id_, rs)

    def do_load(self, id_: str, rs: dict) -> User:
        first_name = rs["first_name"]
        last_name = rs["last_name"]
        nickname = rs["nickname"]
        if first_name and last_name:
            return User(id_, nickname, Name(first_name, last_name))
        return User(id_, nickname, None)
```

测试一下效果:

```python
mapper = UserMapper()

user = new_user("Tom", Name("Tom", "Jackson"))
mapper.insert(user)

user = mapper.find(user.id)
print(user)
```

结果:

```
--------------------
CREATE TABLE IF NOT EXISTS users(
    id         TEXT PRIMARY KEY,
    first_name TEXT,
    last_name  TEXT,
    nickname   TEXT NOT NULL
);
BEGIN
INSERT INTO users(id, first_name, last_name, nickname) VALUES ($1, $2, $3, $4)
COMMIT
SELECT id, first_name, last_name, nickname FROM users WHERE id = $1 LIMIT 1
<User 'Tom·Jackson'>

[Done] exited with code=0 in 0.028210 seconds
```

对于「用户」模型，到这里其实就可以结束了，但你可能注意到在 `find()` 方法的下面有几个看起来有些多余的方法，这是为了接下来的抽象预留的方法，下面我们开始对可以在多个模型间共享的方法进行抽象。

首先对模型本身提取出 `id` 标识:

```python
# data_mapper/domain/models.py
class DomainObject:
    def __init__(self, id_: str):
        self._id = id_

    @property
    def id(self) -> str:
        return self._id


class User(DomainObject):
    def __init__(self, id_: str, nickname: str, realname: Name | None):
        super().__init__(id_)
        ...

    ...
```

然后是映射器:

```python
# data_mapper/db/mappers.py
...

ModelType = TypeVar("ModelType", bound=DomainObject)


class AbstractMapper(ABC, Generic[ModelType]):
    loaded_map: dict[str, ModelType] = {}

    @property
    @abstractmethod
    def find_statement(self) -> str:
        ...

    @property
    @abstractmethod
    def insert_statement(self) -> str:
        ...

    def insert(self, subject: ModelType) -> str:
        statement = self.insert_statement
        cursor = conn.cursor()
        try:
            cursor.execute(statement, self.insert_data(subject))
            conn.commit()
            self.loaded_map[subject.id] = subject
            return subject.id
        finally:
            cursor.close()

    @abstractmethod
    def insert_data(self, subject: ModelType) -> tuple:
        ...

    def find(self, id_: str) -> ModelType:
        if id_ in self.loaded_map:
            return self.loaded_map[id_]
        statement = self.find_statement
        cursor = conn.cursor()
        try:
            rs = cursor.execute(statement, (id_,)).fetchone()
            if rs is None:
                raise KeyError(f"{id_} is not exists")
            return self.load(rs)
        finally:
            cursor.close()

    def load(self, rs: dict) -> ModelType:
        id_ = rs["id"]
        if id_ in self.loaded_map:
            return self.loaded_map[id_]
        return self.do_load(id_, rs)

    @abstractmethod
    def do_load(self, id_: str, rs: dict) -> ModelType:
        ...


class UserMapper(AbstractMapper[User]):
    tablename: str = "users"
    columns: str = "id, first_name, last_name, nickname"

    @property
    def insert_statement(self) -> str:
        return f"INSERT INTO {self.tablename}({self.columns}) VALUES ($1, $2, $3, $4)"

    @property
    def find_statement(self) -> str:
        return f"SELECT {self.columns} FROM {self.tablename} WHERE id = $1 LIMIT 1"

    def insert_data(self, user: User) -> tuple:
        if user.realname:
            return (
                user.id,
                user.realname.first_name,
                user.realname.last_name,
                user.nickname,
            )
        return (user.id, None, None, user.nickname)

    def do_load(self, id_: str, rs: dict) -> User:
        first_name = rs["first_name"]
        last_name = rs["last_name"]
        nickname = rs["nickname"]
        if first_name and last_name:
            return User(id_, nickname, Name(first_name, last_name))
        return User(id_, nickname, None)
```

经过提取后，大部分查询逻辑和数据库交互均转移到了 `AbstractMapper` ，具体的映射器只需要实现部分与具体模型相关的功能即可。此外，我在映射器中添加了一个标识映射，用来避免重复的数据库交互。

我们重新运行代码，得到的结果是:

```python
--------------------
CREATE TABLE IF NOT EXISTS users(
    id         TEXT PRIMARY KEY,
    first_name TEXT,
    last_name  TEXT,
    nickname   TEXT NOT NULL
);
BEGIN
INSERT INTO users(id, first_name, last_name, nickname) VALUES ($1, $2, $3, $4)
COMMIT
<User 'Tom·Jackson'>

[Done] exited with code=0 in 0.030054 seconds
```

观察软件执行的 SQL 语句，可以发现 `find()` 方法并没有真的执行查询，这是由于调用 `insert()` 方法时对象已经被保存到标识映射中了。
