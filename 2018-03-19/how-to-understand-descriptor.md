---
title: 如何理解描述符
author: MisLink
abbrlink: 5dd0238f
date: 2018-03-25 23:52:32
tags:
thumbnail: 'https://user-images.githubusercontent.com/8280169/43658026-29c0aa14-978a-11e8-9df0-ddc0cf55a93b.png'
---

## 前言

上篇文章中挖了 property 和描述符的坑，这篇就把它填上好了\_(:з)∠)\_

property 是用描述符实现的，所以先说说 property。

## property

property 本身是一个实现了描述符协议的类，在不改变类接口的情况下，提供了一组对实例属性的读取、写入和删除操作。下面举个例子，一个银行账户的抽象，很容易实现：

```python
class Account:

    def __init__(self, name, balance):
        self.name = name
        self.balance = balance
```

银行账户最常见的操作就是存款和取款了：

```python
In [1]: account = Account('zhang', 100)  # 创建一个有 100 块存款的账户

In [2]: account.balance
Out[2]: 100

In [3]: account.balance -= 90  # 取 90 块

In [4]: account.balance  # 还剩 10 块
Out[4]: 10

In [5]: account.balance += 30  # 存 30 块

In [6]: account.balance  # 现在有 40 块
Out[6]: 40
```

但是这里有个问题：

```python
...

In [7]: account.balance -= 50  # 再取 50 块

In [8]: account.balance  # 存款变成了负数！
Out[8]: -10
```

当然这种操作是不该被允许的，我们需要对 `balance` 的写入做限制。Jawa 之类的语言会创建一组 getter、setter 方法来管理属性，但是这并不 Python，也对现有的代码不友好。正确的方式是使用 property。

```python
class Account:

    def __init__(self, name, balance):
        self.name = name
        self.balance = balance

    @property
    def balance(self):
        return self._balance

    @balance.setter
    def balance(self, value):
        if value < 0:
            raise ValueError('balance must greater than 0.')
        else:
            self._balance = value
```

现在 `balance` 被禁止设为小于 0 的数：

```python
In [1]: account = Account('zhang', 100)

In [2]: account.balance
Out[2]: 100

In [3]: account.balance += 40

In [4]: account.balance -= 200
---------------------------------------------------------------------------
ValueError                                Traceback (most recent call last)
...
ValueError: balance must greater than 0.

In [5]: account.balance
Out[5]: 140

In [6]: account = Account('zhang', -1)  # 初始化的时候也不行！
---------------------------------------------------------------------------
ValueError                                Traceback (most recent call last)
...
ValueError: balance must greater than 0.
```

可以看到我们使用 `balance` 的方式没有发生变化，但是对值的限制已经生效了。

property 还有一个 `deleter` 装饰器，处理应用于属性的 `del`；当然，`del` 本身用的也不多，大多数时候把销毁操作交给 Python 就可以了。不过如果涉及到复杂对象的引用，要做到 RAII（误，还是要手动实现的。

### property 是类

property 本身是用 C 实现的，[这里](https://docs.python.org/3/howto/descriptor.html#properties)有一个纯 Python 的实现。正如上文所说，它本身是一个类，构造方法的签名如下：

```python
class property(object):
    def __init__(self, fget=None, fset=None, fdel=None, doc=None):
        pass
```

熟悉一点装饰器用法的话就可以看出上面的

```python
class Account:
    ...

    @property
    def balance(self):
        pass
```

实际上就是

```python
class Account:
    ...

    def get_balance(self):
        pass

    balance = property(fget=get_balance)
```

如果不熟悉的话，下一篇就讲装饰器好了（误

### property 的实例是类属性

上面的代码段同时展示了这样一个事实：property 的实例是类属性。这就涉及到了属性查找顺序的问题，简单试一下：

```python
class Foo:
    data = 'data!'

    @property
    def bar(self):
        return 'bar!'
```

```python
In [1]: f = Foo()

In [2]: f.data
Out[2]: 'data!'

In [3]: f.data = 'f.data!'

In [4]: f.data
Out[4]: 'f.data!'

In [5]: Foo.data
Out[5]: 'data!'
```

实例属性覆盖了类属性，符合直觉。那么对 property 的实例来说呢？

```python
In [6]: f.bar
Out[6]: 'bar!'

In [7]: f.bar = 'bar'
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
...
AttributeError: can't set attribute
```

尝试给 `bar` 赋值，失败了，也符合 property 的工作方式：执行赋值时，如果没有 setter 方法就抛出异常。那么直接修改 `f.__dict__` 呢？

```python
In [8]: f.__dict__['bar'] = 'bar'

In [9]: f.bar
Out[9]: 'bar!'
```

也不行，property 的实例完全覆盖了实例属性。但是，它是一个类属性，所以我们可以这样做：

```python
In [10]: Foo.bar
Out[10]: <property at 0x29c44800408>

In [11]: Foo.bar = 'bar'

In [12]: f.bar
Out[12]: 'bar'

In [13]: f.bar = 'ba'

In [14]: f.bar
Out[14]: 'ba'
```

对类属性的覆盖使 `bar` 不再是一个 property 的实例，所以也就不会覆盖后续的赋值了。

当然我们仍然可以用一个 property 的实例再次覆盖 `Foo.bar`：

```python
In [15]: Foo.bar = property(fget=lambda self: 'bar!')

In [16]: f.bar
Out[16]: 'bar!'
```

恢复原样。 property 的实例这种先从类中开始属性查找的方式，是一类描述符的工作模式。接下来就说说描述符。

## 描述符

描述符是指实现了描述符协议的类，这个协议包含四个方法，分别是 `__get__`，`__set__`，`__delete__` 和 Python 3.6 新增的 `__set_name__`。通常，只要实现了 `__get__` 或 `__set__`，就可以被称之为描述符。在某个角度上说，描述符的作用相当于抽象的 property，可以为一组属性提供相同的读取、写入和删除逻辑。接下来，还是从数据验证的例子开始。

下面是商店中一项商品的抽象，包含商品名、数量和单价：

```python
class Item:
    amount = Storage('amount')
    price = Storage('price')

    def __init__(self, name, amount, price):
        self.name = name
        self.amount = amount
        self.price = price
```

其中的 `amount` 和 `price` 都必须大于 0，所以可以用统一的描述符实现：

```python
class Storage:

    def __init__(self, name):
        self.name = name

    def __set__(self, instance, value):
        if value > 0:
            instance.__dict__[self.name] = value
        else:
            raise ValueError(f'{self.name} must greater than 0.')
```

由于我们并没有对读取方法有特别的需求，所以不用实现 `__get__` 方法。

试一下：

```python
In [1]: item = Item('orange', 100, 0)
---------------------------------------------------------------------------
ValueError                                Traceback (most recent call last)
...
ValueError: price must greater than 0.

In [2]: item = Item('orange', 0, 100)
---------------------------------------------------------------------------
ValueError                                Traceback (most recent call last)
...
ValueError: amount must greater than 0.
```

如果 `amount` 或 `price` 中的任何一个不大于 0，都会被禁止。

这里需要解释一下 `__set__` 的签名中的 `instance`：

```python
def __set__(self, instance, value):
    pass
```

`instance` 是 `Item` 的实例。因为描述符应该管理实例的属性，所以需要额外的参数提供相应的实例。这也是为什么我们不能这样写：

```python
def __set__(self, instance, value):
    self.__dict__[self.name] = value
```

这实际上是为描述符实例设置了值，而描述符实例是`Item` 类的类属性，所有的 `Item` 实例都共享相同的描述符实例。修改了某个描述符实例，相当于修改了所有的 `Item` 实例。

上面的例子有个缺点，初始化描述符实例的时候需要重复属性的名字。我们希望可以简单的写成：

```python
class Item:
    amount = Storage()
    price = Storage()
    ...
```

而不需要在描述符的构造方法中重复属性名。这就是 Python 3.6 新增的 `__set_name__` 方法的作用。只要实现 `__set_name__` 方法：

```python
class Storage:
    ...

    def __set_name__(self, owner, name):
        self.name = name
```

同样解释一下函数签名：

```python
def __set_name__(self, owner, name):
    pass
```

`owner` 是 `Item` 类本身，`name` 是引用描述符实例的变量的名字。

如果使用的 Python 版本在 3.6 以下呢？有两个方法：第一个是用元类接管`Item`类的创建过程，这个不在这篇文章的内容之内（可能又挖了一个坑；第二个就是为每个描述符实例生成与属性名无关但是唯一字符串，用来代替属性名：

```python
class Storage:

    _counter = 0

    def __init__(self):
        cls = self.__class__
        self.name = f'_{cls.__name__}#{cls._counter}'
        cls._counter += 1

    def __get__(self, instance, owner):
        return getattr(instance, self.name)

    def __set__(self, instance, value):
        if value > 0:
            setattr(instance, self.name, value)
        else:
            raise ValueError('must greater than 0.')
```

由于 `Item` 中的属性名和我们实际保存的属性名不同，所以需要实现 `__get__` 方法。与 `__set_name__` 签名中的 `owner` 含义相同，`__get__` 方法签名中的 `owner` 也是 `Item` 类本身。

现在，我们使用 `_Storage#N` 这样的名称在 `Item` 实例中保存属性。当然，这样的名称会让人有点困惑，特别是以类属性访问的时候：

```python
In [1]: Item.amount
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
...
AttributeError: 'NoneType' object has no attribute '_Storage#0'
```

为了避免在如此明显的地方暴露我们的实现细节，我们可以修改异常的错误消息，或者，内省描述符实例：

```python
def __get__(self, instance, owner):
    if instance is None:
        return self
    else:
        return getattr(instance, self.name)
```

### 两类描述符

上述例子中对数据属性的控制和管理是描述符的典型用途之一。这种实现了 `__set__` 方法，接管了设置属性行为的描述符，被称为覆盖型描述符，没有定义 `__set__` 方法的描述符，被称为非覆盖型描述符。由于 Python 中对实例属性和类属性的处理方式不同，这两类描述符也有不同的行为。

#### 覆盖型描述符

实现了 `__set__` 方法的描述符就是覆盖型描述符。这类描述符虽然是类属性，但是会覆盖实例属性的赋值操作：

```python
class Override:

    def __get__(self, instance, owner):
        print('get!')

    def __set__(self, instance, value):
        print('set!')


class Manager:

    override = Override()
```

下面做一些实验：

```python
In [1]: m = Manager()

In [2]: m.override
get!

In [3]: m.override = 1
set!

In [4]: Manager.override
get!

In [5]: m.__dict__['override'] = 1

In [6]: m.__dict__
Out[6]: {'override': 1}

In [7]: m.override
get!

```

可以看出，无论以实例属性还是类属性访问 `override`，都会触发 `__get__` 方法；为实例属性 `override` 赋值会触发 `__set__` 方法；即使跳过描述符直接为 `m.__dict__` 赋值，读取 `override` 的操作仍然会被描述符覆盖。

##### 没有 `__get__` 方法的覆盖型描述符

如果只实现了 `__set__` 会发生什么呢？

```python
class OverrideNoGet:

    def __set__(self, instance, value):
        print('set!')


class Manager:

    override_no_get = OverrideNoGet()
```

```python
In [1]: m = Manager()

In [2]: m.override_no_get
Out[2]: <__main__.OverrideNoGet at 0x29c44a97668>

In [3]: Manager.override_no_get
Out[3]: <__main__.OverrideNoGet at 0x29c44a97668>

In [4]: m.override_no_get = 1
set!

In [5]: m.override_no_get
Out[5]: <__main__.OverrideNoGet at 0x29c44a97668>

In [6]: m.__dict__['override_no_get'] = 1

In [7]: m.override_no_get
Out[7]: 1

In [8]: m.override_no_get = 2
set!

In [9]: m.override_no_get
Out[9]: 1
```

可以看到，没实现 `__get__` 方法，无论以实例属性还是类属性访问 `override_no_get`，都会返回描述符实例；而赋值操作可以触发 `__set__` 方法；由于我们的 `__set__` 方法并没有真正修改实例属性，所以再次访问 `override_no_get` 仍然会得到描述符实例；通过 `m.__dict__` 修改实例属性后，实例属性就会覆盖描述符；不过只有访问实例属性时才是如此，赋值仍然由 `__set__` 处理。

#### 非覆盖型描述符

没有实现 `__set__` 方法的描述符就是非覆盖型描述符：

```python
class NonOverride:

    def __get__(self, instance, owner):
        print('get!')


class Manager:

    non_override = NonOverride()
```

```python
In [1]: m = Manager()

In [2]: m.non_override
get!

In [3]: Manager.non_override
get!

In [4]: m.non_override = 1

In [5]: m.non_override
Out[5]: 1

In [6]: Manager.non_override
get!

In [7]: del m.non_override

In [8]: m.non_override
get!
```

无论访问实例属性还是类属性，都会触发 `__get__` 方法；由于没有 `__set__` 方法，对属性的赋值不会被干涉；对属性复制之后，实例属性就会覆盖同名的描述符，但是访问类属性仍然可以触发 `__get__` 方法；如果把 `non_override` 从实例中删除，访问 `non_override` 的操作又会交给 `__get__`。

当然，描述符都是定义在类上的，如果对同名的类属性进行赋值，就会完全替换掉描述符。这里表现出读、写属性时的不对等：对类属性的读操作可以被 `__get__` 处理，但是写操作不会。当然，了解一些 Python 的话就会知道还存在着另一种不对等：读取实例属性时，会返回实例属性，如果实例属性不存在，会返回类属性；但是为实例属性赋值时，如果实例属性不存在，会在实例中创建属性，不会影响到类属性。

## 结语

描述符充斥在 Python 底层（举个例子：Python 中的方法是怎么实现的？）与各种框架中，理解描述符是体会 Python 世界工作原理和设计美学的重要方式。
