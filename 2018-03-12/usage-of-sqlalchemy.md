---
title: 普通的 SQLAlchemy ORM 使用姿势
author: MisLink
abbrlink: 39277c31
date: 2018-03-18 21:38:54
tags:
thumbnail: http://ourm7pfm2.bkt.clouddn.com/18-3-18/10592292.jpg
---

## 前言

SQLAlchemy 是 Python 世界中最常用的 SQL 工具之一，包含 SQL 渲染引擎和 ORM 两大部分。平时使用最多的就是 ORM 部分，这篇文章的重点也是 ORM。在我看来平时很多使用 ORM 的姿势是有问题的，或者说是不优雅的。所以这篇文章打算讲讲（搬运）其中一些普通的姿势和技巧（API 文档）。

## property 和混合属性

### property

下面是一个简单的用户表映射：

```python
class User(Base):
    __tablename__ = 'user'
    id = Column(Integer, primary_key=True)
    name = Column(String(64))
    password = Column(String(128))
```

通常情况下，我们会加密用户的密码，在数据库中保存密文，但是这里有一个问题，我们得这么写：

```python
# 创建用户
user = User(name='zhang', password=encrypt('123456'))
# 修改密码
user.password = encrypt('654321')
```

这意味着我们需要不断的重复书写 `encrypt` 函数来保证加密了用户密码。

有没有什么方法能省去这一步呢？答案是 `property`。

现在把用户表映射改成这样：

```python
class User(Base):
    __tablename__ = 'user'
    id = Column(Integer, primary_key=True)
    name = Column(String(64))
    _password = Column(String(128))

    @property
    def password(self):
        raise ValueError('write only!')

    @password.setter
    def password(self, value):
        self._password = encrypt(value)
```

现在只需要简单的写成：

```python
# 创建用户
user = User(name='zhang', password='123456')
# 修改密码
user.password = '654321'
```

就可以了。

关于 Python 中 `property `和描述符的使用值得再另写一篇文章描述，在这里就不详细说明了。

### 混合属性（hybrid_property）

上面的例子看上去让代码清爽了不少，但是有时候这种用法是无法满足需要的，譬如下面这个例子：

```python
class Student(Base):
    __tablename__ = 'student'
    id = Column(Integer, primary_key=True)
    name = Column(String(64))
    birthday = Column(DateTime)
```

这是一个学生表映射，增加了 `birthday ` 字段。通常我们会保存用户的生日，再通过生日获取用户年龄。有了上面的例子，很容易写出获取年龄的代码：

```python
class Student(Base):
    ...

    @property
    def age(self):
        return datetime.now().year - self.birthday.year
```

现在可以简单的使用 `student.age` 获取具体的生日。

这样做是有缺陷的：如果需要获取所有 18 岁的学生呢？我们希望可以这样写：

```python
session.query(Student).filter_by(age=18).all()
```

但是却没有任何结果返回。如果改成这样呢？

```python
now = datetime.now()
start = datetime(now.year - 18, 1, 1)
end = end = datetime(now.year + 1 - 18, 1, 1)
session.query(Student).filter(Student.birthday >= start, Student.birthday < end).all()
```

这样倒是可以获取正确的结果了，但是也太丑了点吧？难道没办法写出像第一条一样的既清晰又简洁的查询么？

答案自然是有的，SQLAlchemy 提供了混合属性（`hybrid_property`）来处理类似的情况，于是我们可以改写获取年龄的代码：

```python
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy import func

class Student(Base):
    ...

    @hybrid_property
    def age(self):
        return datetime.now().year - self.birthday.year

    @age.expression
    def age(self):
        return datetime.now().year - func.year(self.birthday)
```

这里将原本的 `property` 替换为 SQLAlchemy 中的 `hybrid_property`，同时提供了一个 `expression` 装饰器，在被装饰的方法中把 Python 代码翻译成 SQL（代码示例的目标数据库为 MySQL，获取日期中的年份的函数为`YEAR()`，使用其他数据库请查阅对应数据库的相关文档）。有了这个方法，SQLAlchemy 就知道如何在 SQL 语句中处理 `age` 属性了。

接下来稍微提一下 `hybrid_method`。

和 `hybrid_property` 类似，只不过可以给 `hybrid_method` 传参数。下面这个例子不太合适，只为了展示`hybrid_method` 的功能。

如何找到所有 90 后同学？当然我们可以复用上面的 `age` 属性，先计算一下 90 后的同学现在多少岁，然后直接写在查询里就好：

```python
session.query(Student).filter(Student.age >= now.year - 1990, Student.age < now.year - 2000).all()
```

如果要判断某个学生是否是 90 后的呢？又需要再写一遍：

```python
if now.year - 2000 > student.age >= now.year - 1990:
    ...
```

出现了很多不直观的代码，这时候可以使用 `hybrid_method` 简化：

```python
class Student(Base):
    ...
    @hybrid_method
    def born_after(self, years):
        return years + 10 > self.birthday.year >= years

    @born_after.expression
    def born_after(self, years):
        return and_(func.year(self.birthday) < years + 10, func.year(self.birthday) >= years)
```

于是现在可以这样做：

```python
session.query(Student).filter(Student.born_after(1990)).all()

if student.born_after(1990):
    ...
```

看上去好了一些（误

这一部分就到此为止，当然 hybrid 在 SQLAlchemy 中的用法不止上述这些，更详细和复杂的内容参见官方文档。

## 关联代理（association_proxy）

### 简化标量集合

关联代理用在有关联的表中，所以我们先创建如下映射关系：

```python
association = Table('association', Base.metadata,
                    Column('blog_id', Integer, ForeignKey('blog.id'), primary_key=True),
                    Column('tag_id', Integer, ForeignKey('tag.id'), primary_key=True))


class Blog(Base):
    __tablename__ = 'blog'
    id = Column(Integer, primary_key=True)
    name = Column(String(64))
    tags = relationship(
        'Tag', secondary=association, backref=backref('blogs', lazy='dynamic'), lazy='dynamic')


class Tag(Base):
    __tablename__ = 'tag'
    id = Column(Integer, primary_key=True)
    name = Column(String(64))
```

一个经常被拿出来作为演示的 Many-To-Many 模型。

先填充一些数据：

```python
In [1]: blog = Blog(name='first')
In [2]: blog.tags.append(Tag(name='t1'))
In [3]: blog.tags.append(Tag(name='t2'))
In [4]: session.add(blog)
In [5]: session.commit()
```

接下来就可以获取这些对象的所有信息了：

```python
In [4]: blog.tags.all()
Out[4]: [<Tag at 0x1fdbab6f198>, <Tag at 0x1fdbab6f208>]

In [5]: blog.tags.all()[0].name
Out[5]: 't1'

In [6]: [t.name for t in blog.tags]
Out[6]: ['t1', 't2']
```

上面的操作有点复杂。对我们而言，`Tag` 对象只有 `name` 字段是有用的，为了获取 `name` 字段，我们要写很多额外的代码把 `name` 字段从 `Tag` 对象中剥离出来。`association_proxy` 就可以用来简化这个操作。

现在修改一下上面的 `Blog` 映射：

```python
from sqlalchemy.ext.associationproxy import association_proxy


class Blog(Base):
    ...

    tag_objectss = relationship(
        'Tag', secondary=association, backref=backref('blogs', lazy='dynamic'), lazy='dynamic')
    tags = association_proxy('tag_objects', 'name')
```

增加了一行 `association_proxy` 对象的声明，现在我们可以这样做：

```python
In [7]: blog.tags
Out[7]: ['t1', 't2']
```

现在查询操作变得很简单了，但是新增标签的操作还是很麻烦：

```python
blog.tag_objects.append(Tag(name='t3'))
```

还是需要实例化一个 `Tag` 对象，能不能直接写：

```python
blog.tags.append('t4')
```

当然是可以的，只要再修改一下 `association_proxy` 的声明：

```python
class Blog(Base):
    ...

    tags = association_proxy('tag_objects', 'name', creator=lambda name: Tag(name=name))
```

参数 `creator` 接受一个可调用对象，它告诉 `association_proxy` 如何处理“新增”操作。

**注意**：`creator` 的默认参数是被代理对象的构造函数，如果提供了一个单参数的构造函数，那么可以省略 `creator` 参数。

### 简化关联对象

上面的例子里把 `association` 表作为一个普通的 `Table` 对象，是因为 `association` 中不需要保存额外信息，只需要作为 `Blog` 和 `Tag` 的中转。现在有了新的需求，我们需要知道每篇博客的标签是在什么时候加上的，这就需要在 `association` 表中增加一个额外的字段用来表示创建时间，同时为了获取这个时间，也要把 `association` 改造成一个真正的映射：

```python
class Association(Base):
    __tablename__ = 'association'

    blog_id = Column(Integer, ForeignKey('blog.id'), primary_key=True)
    tag_id = Column(Integer, ForeignKey('tag.id'), primary_key=True)
    created_at = Column(DateTime, default=datetime.now)

    blog = relationship('Blog', backref=backref('blog_tags', lazy='dynamic'), lazy='joined')
    tag = relationship('Tag', backref=backref('tag_blogs', lazy='dynamic'), lazy='joined')


class Tag(Base):
    __tablename__ = 'tag'
    id = Column(Integer, primary_key=True)
    name = Column(String(64))


class Blog(Base):
    __tablename__ = 'blog'
    id = Column(Integer, primary_key=True)
    name = Column(String(64))
```

这里实际上是把 Many-To-Many 拆成了两个 One-To-Many。

然后构造一些数据：

```python
In  [1]: blog = Blog(name='first')
    ...: tags = [Tag(name='t1'), Tag(name='t2')]
    ...: for tag in tags:
    ...:     session.add(Association(blog=blog, tag=tag))
    ...: session.add(blog)
    ...: session.add_all(tags)
    ...: session.commit()
```

现在就可以获取 `Tag ` 和被添加的时间了：

```python
In [2]: blog.blog_tags[0].tag.name
Out[2]: 't1'

In [3]: blog.blog_tags[0].created_at
Out[3]: datetime.datetime(2018, 3, 18, 16, 4, 17)
```

可以看到，给 `Blog` 增加标签要经过 `Association` 这个中间对象。虽然表结构的确如此，但是我们仍然希望 `Association` 表是透明的，仅当需要获取其中的创建时间时才明确获取 `Association` 对象。只需要在 `Blog` 中声明一个关联代理：

```python
class Blog(Base):
    ...

    tags = association_proxy('blog_tags', 'tag', creator=lambda tag: Association(tag=tag))
```

然后就可以这样写了：

```python
In [4]: blog.tags[0].name
Out[4]: 't1'
```

添加新的 `Tag` 也方便了很多：

```python
In  [3]: for tag in [Tag(name='t3'), Tag(name='t4')]:
    ...:     blog.tags.append(tag)
```

### 混合关联代理

现在回到了第一个问题的出发点，能不能在上一个例子的基础上简化 `tags` 的调用呢？同样没问题，只要在 `Association` 中加一个关联代理：

```python
class Association(Base):
    ...

    tag_objects = relationship('Tag', backref=backref('tag_blogs', lazy='dynamic'), lazy='joined')
    tags = association_proxy('tag_objects', 'name', creator=lambda name: Tag(name=name))
```

然后用起来就和第一个例子一样了：

```python
In [1]: blog.tags
Out[1]: ['t1', 't2']

In [2]: blog.tags.append('t3')
In [3]: blog.tags
Out[3]: ['t1', 't2', 't3']
```

## 结语

上述内容并没有很复杂的操作，都是一些易于实现并且可以改善日常使用体验的方法。SQLAlchemy 还有很多骚操作可以讲，但是受限于本人的姿势水平，很多并没有实际使用过，也谈不上有什么见解。那就这样吧~
