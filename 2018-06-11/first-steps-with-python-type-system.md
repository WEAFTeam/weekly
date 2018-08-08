---
title: Python 类型系统的第一步
abbrlink: cde2bcff
date: 2018-08-08 21:52:35
tags:
author: MisLink
thumbnail: https://user-images.githubusercontent.com/8280169/43841718-6529b0e4-9b56-11e8-8e2f-f944101765ab.png
---

## 写在前面

本文翻译自 [First Steps with Python Type System](https://blog.daftcode.pl/first-steps-with-python-type-system-30e4296722af)。

自豪的采用搜狗翻译。

## 这是什么玩意？

在过去几年中，类型注解的语法和语义逐渐被引入 Python 语言。类型在 Python 中仍然是一个非常新颖并且经常被误解的主题。在这篇文章中，我会介绍它的基础知识，而一些更高级的功能将出现在本文的后续部分中。

这篇文章基于 [PEP 483: The Theory of Type Hints](https://www.python.org/dev/peps/pep-0483)，[PEP 484: Type Hints](https://www.python.org/dev/peps/pep-0484)，Python 类型[文档](https://docs.python.org/3/library/typing.html)，mypy 的 [github issues](https://github.com/python/mypy/issues)，和我个人对于类型在现实代码方面的经验。我在使用的版本为 Python 3.6 和 mypy 0.620。

## 1. 类型和类

 为了更好地掌握 Python 的类型系统，我们需要区分类型和类。在发布 PEP 483 和 484 之前，这两个概念有些混乱。现在，一般来说，类型是类型检查器概念，类是运行时概念。在这两者的基本描述中，我们已经获得了关键信息：类型不在运行时的 “领域” 中。实际上，类型是程序的另一个 “层” 上的东西，一个用于类型检查的层。

但是什么是类型检查器？这是一个分析我们代码的工具（类似于 flake8，但更智能）。它不以任何方式运行我们的代码，只会静态检查代码的类型一致性。Python 社区的官方类型检查器是 mypy。此外还有 Facebook 的 [pyre-check](https://github.com/facebook/pyre-check) 和谷歌的 [pytype](https://github.com/google/pytype)。

### 1.1 如何定义类型？

类型检查器需要关于类型的信息来检查我们的程序。定义类型有三种基本方法：

1. 通过定义一个类，
2. 通过指定使用类型变量的函数，
3. 通过使用更基本的类型来创建更复杂的类型。

第一种情况中，语句 `class Animal: …` 同时定义了 `Animal` 类和 `Animal` 类型。这也适用于内置类型：`int `，`float`，`str`，`list`，`dict` 等等，它们也同时是类和类型。在这种情况下，类之间的继承关系被一一映射到子类型关系。因此，如果 `Dog` 是 `Animal` 的一个子类，那么 `Dog` 就是 `Animal` 的一个子类型，等等。这种处理类型的方法被称为“命名子类型”。一会儿，我将展示它在类型检查的情境中是如何工作的。

第二种情况是鸭子类型的精神：我们通过指定哪些函数/方法使用此类型的变量来定义类型。例如，如果一个对象有 `__len__` 方法，那么它具有 `Sized` 类型。这种处理类型的方法被称为“结构子类型”。这本身就是一个话题，在这篇文章中只会略加讨论。

在第三种情况中，我们使用早期定义的类型（以任何方式）来定义更复杂的类型。例如，我们可以定义以下类型：“仅包含整数或字符串实例的列表”。稍后将介绍这些类型。

## 2. 类型注解语法

 为了用类型信息给我们的代码做注解，我们需要一种特殊的语法。这种语法逐渐被引入到语言中，但是现在只关注它的当前（也很可能是最终）状态。

### 2.1 注解变量

要注解变量，我们使用变量名后跟冒号和类型名。变量初始化是可选的：

```python
name: Type
```

类型注解与初始化变量，如下所示：

```python
name: Type = initial_value
```

所以从现在开始，mypy 知道，在这个范围内，`name` 应该具有 `Type` 类型，并将检查是否确实如此。事实上，第一次检查是在赋值阶段进行的：`initial_value` 是 `Type` 类型的吗？

如果一个变量没有初始化，我们就不能使用它（会引发 `NameError`），但是稍后，在我们初始化它之后，mypy 会用声明的变量类型来检查值的类型。

让我们看看它是什么样的。为了方便起见，我会把所有的 mypy 错误放在注释中。

```python
width: int
width = 15  # 没有错误

height: int
height = "25"  # 错误:
# Incompatible types in assignment (expression has type "str", variable has type "int")

depth: int = 15.5  # 错误:
# error:Incompatible types in assignment (expression has type "float", variable has type "int")
```

所以即使在这些简单的情况下，mypy 也已经很有用了。

### 2.2 注解函数

我们还可以注解函数的参数类型和返回值类型。使用以下语法：

```python
def function(arg1: Type1, arg2: Type2) -> ReturnType:
    ...
```

让我们看看：

```python
def add_ints(x: int, y: int) -> int:
    return x + y  # 没有错误


add_ints(1, 2)  # 没有错误
add_ints(1, 2.0)  # 错误:
# Argument 2 to "add_ints" has incompatible type "float"; expected "int"


def broken_add(x: int, y: int) -> str:
    return x + y  # 错误:
    # Incompatible return value type (got "int", expected "str")
```

在第二个例子中，mypy 通过检查 `+` 运算符（实际上是 `__add__` 方法）的返回类型知道 `broken_add` 在两个 `int` 上使用时有错误的返回类型：它是一个 `int`，而不是 `str`，所以函数的返回类型声明不正确。

## 3. 子类型

在我们开始尝试所有关于 Python 类型的东西之前，我们需要更好地理解基本的子类型关系。

让我们来看看这两个类：

```python
class Animal:
    ...

class Dog(Animal):
    ...
```

基本上，子类型是一种不太普遍的类型。在我们的例子中，`Dog` 没有 `Animal` 一般，所以它是 `Animal` 的一个子类型。但是让我们深入一点，看看 Python 中如何定义子类型关系。这个定义将决定赋值规则和属性规则的使用，这些规则是 mypy 在代码上强制执行的。

### 3.1 定义

我们用 `<:` 表示子类型关系。（例如 `B <: A` 表示 `B` 是 `A` 的子类型。）

现在，`B <: A` 当且仅当：

1. 类型 `B` 的每个值也在类型 `A` 的值的集合中；并且
2. 类型 `A` 的每个函数也在类型 `B` 的函数的集合中。

（“类型 `A` 的函数” 基本上意味着“接受类型 `A` 的对象作为其参数的函数”。因此，它可以是一个具有 `A` 类型参数的独立函数，也可以是在 `A` 类上定义的方法。）

因此，在子类型化过程中，值集合变得更小，而函数集合变得更大（参见[文档](https://www.python.org/dev/peps/pep-0483/#subtype-relationships)和[这个博客](https://www.stephanboyer.com/post/132/what-are-covariance-and-contravariance)，它启发了符号和示例）。

就我们的两种类型而言，`Dog <: Animal` 意味着：

1. 一组 `Dog` 是 `Animal` 的子集（每只 `Dog` 都是 `Animal`，但不是每只 `Animal` 都是 `Dog`）。这基本上意味着 `Dog` 比 `Animal` 少。
2. `Animal` 的一组函数是 `Dog` 的函数的子集（`Dog` 可以做任何 `Animal` 能做的事情，但是 `Animal` 不能做任何 `Dog` 能做的事情）。基本上，`Animal` 能做的比 `Dog` 少。

### 3.2 赋值规则

这个定义决定了哪些赋值是可接受的，哪些是不可接受的。让我们尝试将一种类型的变量分配给另一种类型的变量。

```python
# Dog <: Animal
scooby: Dog
an_animal: Animal

an_animal = scooby  # 没有错误
```

将 `scooby` 赋值给 `an_animal` 是类型安全的，因为 `scooby` 被保证是一只 `Animal`。

```python
# Dog <: Animal
scooby: Dog
an_animal: Animal

scooby = an_animal  # 错误:
# Incompatible types in assignment (expression has type "Animal", variable has type "Dog")
```

把 `an_animal` 赋值给 `scooby` 不是类型安全的，因为 `an_animal` 可能不是 `Dog`。

检查继承关系是命名子类型化方法的一部分。

### 3.3 属性规则

mypy 不仅关注赋值，还关注属性的使用。更准确地说，它检查属性是否实际定义在对象上。

```python
class Animal:
    def eat(self): ...

class Dog(Animal):
    def bark(self): ...
```

现在 `Animal` 可以 `eat`，`Dog` 可以 `eat` （通过继承）和 `bark`。

```python
# Dog <: Animal
an_animal: Animal
snoopy: Dog

an_animal.eat()  # 没有错误
snoopy.eat()  # 没有错误

snoopy.bark()  # 没有错误
an_animal.bark()  # 错误: "Animal" has no attribute "bark"
```

mypy 确保确实在相关对象上定义了方法。`an_animal` 没有定义 `bark` 方法，因此会报告一个错误。

检查属性，特别是方法，是结构子类型化方法的一部分。在这种方法中，“子类型关系是从声明的方法中推导出来的”[[来源](https://www.python.org/dev/peps/pep-0483/#background)]。

## 4. 定义复杂类型

让我们看看如何使用更基本的类型来创建更复杂的类型。这是在 Python 中定义类型的第三种方法。我将集中讨论几个典型的复杂类型，而其余的工作方式基本相同。

### 4.1 列表

其中最基本的是 `List`。除了大写字母 `L` 之外，它的拼写与内置 `list` 相同。语法如下：`List[TypeOfElements]`。所以整数列表是 `List[int]`，字符串列表是 `List[str]` 等等。让我们来看看代码:

```python
from typing import List

my_list: List[int] = [1, 2, 3]  # 没有错误

my_other_list: List[int] = [1, 2, "3"]  #  错误:
# List item 2 has incompatible type "str"; expected "int"


class Animal:
    pass


class Dog(Animal):
    pass


scooby = Dog()
lassie = Dog()
pinky = Animal()

my_dogs: List[Dog] = [scooby, lassie, pinky]  # 错误:
# List item 2 has incompatible type "Animal"; expected "Dog"

```

这是有道理的，但是我们都知道，在 Python 列表中，我们可以放置多种类型的项目： `[1, 2, '3']` 仍然是有效的列表。一会儿，我们将看到如何表达像 “整数或字符串” 这样的类型。但是首先，让我们看看元组和字典类型。

### 4.2 元组

在 Python 语言中，元组传统上有两个用途。首先，这是一个 “不可改变的列表”。第二，它是 “记录” 或 “一行值”，其中每个位置上的值通常都有特定定义的类型；把它想象成 SQL 数据库中的一行。`Tuple`（大写 `T` ）类型支持这两种方法。

要将元组定义为记录，使用以下语法：`Tuple[Type1, Type2, Type3]`（等等）。

要将元组定义为不可变列表，使用元组与省略号对象（用三个点拼写：`...`）：`Tuple[TypeOfAllElements, ...]`。

```python
from typing import Tuple

bob: Tuple[str, str, int] = ("Bob", "Smith", 25)  # 没有错误

frank: Tuple[str, str, int] = ("Frank", "Brown", 43.4)  # 错误:
# Incompatible types in assignment (expression has type "Tuple[str, str, float]",
#   variable has type "Tuple[str, str, int]")

ann: Tuple[str, str, int] = ("Ann", "X", 1, 2)  # 错误:
# Incompatible types in assignment (expression has type "Tuple[str, str, int, int]",
#   variable has type "Tuple[str, str, int]")

scores1: Tuple[int, ...] = (5, 8, 4, -1)  # 没有错误

scores2: Tuple[int, ...] = (5, 8, 4, -1, None, 7)  # 错误:
# Incompatible types in assignment (expression has type
#   "Tuple[int, int, int, int, None, int]", variable has type "Tuple[int, ...]")
```

### 4.3 命名元组

在 Python 中也有 `namedtuple`。即使不涉及类型，这也是一个非常方便的元组扩展。它增加了字段名查找和一个漂亮的字符串表示：

```python
>>> from collections import namedtuple

    Person = namedtuple('Person', 'first_name last_name age')

    bob = Person(first_name='Bob', last_name='Smith', age=41)

>>> bob.age
41

>>> bob
Person(first_name='Bob', last_name='Smith', age=41)

>>> # 依然可以使用旧的访问方式：
    bob[0]
'Bob'

>>> bob[1:]
('Smith', 41)

>>> list(bob)
['Bob', 'Smith', 41]
```

在 Python 3 中，`namedtuple` 有一个更年轻的类型兄弟：`NamedTuple`（同样，用大写字母 `N` 和 `T` 拼写）。在运行时，它具有与 `namedtuple` 完全相同的 API，但是另外，它支持类型注解：

```python
from typing import NamedTuple


class Person(NamedTuple):
    first_name: str
    last_name: str
    age: int


Person("Bob", "Smith", 41)  # 没有错误

Person("Kate", "Smith", "32")  # 错误:
# Argument 3 to "Person" has incompatible type "str"; expected "int"
```

不错，易读，方便！

### 4.4 字典

另一种重要的 Python 类型是字典。其类型的定义类似于元组：`Dict[KeyType, ValueType]`。因此，将整数键映射到字符串值的字典是 `Dict[int, str]`。

```python
from typing import Dict

id_to_name: Dict[int, str] = {1: "Bob", 23: "Ann", 7: "Kate"}  # 没有错误
id_to_age: Dict[int, int] = {"1": 41, 2: 22}  # 错误:
# Dict entry 0 has incompatible type "str": "int"; expected "int": "int"

name_to_phone_no: Dict[str, str] = {"Bob": "55534534", "Ann": 55599412}  # 错误:
# Dict entry 1 has incompatible type "str": "int"; expected "str": "str"
```

还有其他 Python 集合的类型：`Set`、`FrozenSet`、`DefaultDict`、`Counter`、`Deque` 和许多其他类型。有关完整列表，请参见[文档](https://docs.python.org/3/library/typing.html)。

现在让我们关注另一种创建复杂类型的方法。

### 4.5 Union

假设我们有一个变量可以有 `str` 类型或 `int` 类型，这取决于情况（像不同的数据源）。为了定义这种类型，我们可以使用 `Union`。在我们的例子中，它将是`Union[str, int]`。在方括号中，我们可以根据需要放置任意多的类型：`Union[Type1, Type2, Type3, Type4]`（等等）。形式上：

> `Union[t1, t2, ...]`.  `t1, ...` 中至少一个的子类型是这个类型的子类型。[[来源](https://www.python.org/dev/peps/pep-0483/#fundamental-building-blocks)]

一些例子：

```python
from typing import Union

width1: Union[int, float] = 20  # 没有错误
width2: Union[int, float] = 20.5  # 没有错误
width3: Union[int, float] = "44"  # 错误:
# Incompatible types in assignment (expression has type "str",
#   variable has type "Union[int, float]")


class Animal:
    def eat(self):
        pass


class Dog(Animal):
    pass


class Cat(Animal):
    pass


class Lizard(Animal):
    pass


def restricted_eat(animal: Union[Dog, Cat]) -> None:
    animal.eat()


a_dog: Dog
restricted_eat(a_dog)  # 没有错误

a_cat: Cat
restricted_eat(a_cat)  # 没有错误

a_lizard: Lizard
restricted_eat(a_lizard)  # 错误:
# Argument 1 to "restricted_eat" has incompatible type "Lizard";
#   expected "Union[Dog, Cat]"

```

### 4.6 None 类型和可选类型

一种常见的编程模式是使用一个变量来表示具体的值，或者用一个符号来表示没有值（当值丢失、损坏、还不可用、在当前上下文中不充分等等）。在 Python 中，为了表示没有值，`None` 是最常用的对象。

None 的类型是 `NoneType`，但是在类型系统中，它有一个别名，即... `None` 本身。别名非常有用，因为它不涉及导入任何内容。因此，表达 `T` 类型或 `None` 类型的值的最自然方式是 `Union[T, None]`。所以整型或无应该是 `Union[int, None]`。

这种类型或无的模式非常普遍，以至于在 Python 的类型系统中 `Union[T, None]` 有一个别名：`Optional[T]`。例如，要想表达 `Union[int, None]` 应该使用 `Optional[int]`。

忘记变量的“可选性”常常会导致错误。mypy 真的可以帮到我们。
