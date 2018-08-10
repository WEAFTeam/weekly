---
title: Python 类型系统的下一步
author: MisLink
thumbnail: >-
  https://user-images.githubusercontent.com/8280169/43972489-202ddfbc-9d07-11e8-80f9-bba30285b5b9.png
abbrlink: 684b8f9
date: 2018-08-11 01:35:33
tags:
---
## 写在前面

本文翻译自 [Next Steps with Python Type System](https://blog.daftcode.pl/next-steps-with-python-type-system-efc4df5251c9)。

自豪的采用搜狗翻译。

这是 Python 类型系统的第二篇文章。这篇文章中，我将展示 Python 类型的一些更高级的特性。此外也会包括一些关于使用特定类型功能的提示，和一个如何将类型系统引入你的代码库中的简短指南。

## 1. 约束类型

在上一篇博客文章中描述了 `Optional` 类型。让我们回到展示其用法的片段：

```python
from typing import Optional


def get_user_id() -> Optional[int]:
    pass


def process_user_id(user_id: int):
    pass


user_id = get_user_id()
process_user_id(user_id)  # 错误:
# Argument 1 to "process_user_id" has incompatible type "Optional[int]"; expected "int"
```

mypy 类型检查器报告的错误确实是正确和有用的。但是如果你真的知道在当前上下文中 `get_user_id()` 会返回一个 `int`，而你只想把它传递给 `process_user_id()` 怎么办？首先，考虑一下你的程序结构是否不复杂，是否需要重构。你还想这么做吗？嗯，我们需要以某种方式通知mypy 类型已经改变了。在我们的例子中，这种变化实际上是限制性的：从 `Optional[int]` （即 `Union[int, None]`）到 `int`。首先，让我们尝试最显而易见的方法来实现它。

### 1.1 具有新的类型注解的类型约束 [不正确]

最简单的方法似乎是用更严格的类型注解变量。

```python
user_id: int = get_user_id()  # 错误:
# Incompatible types in assignment (expression has type "Optional[int]",
#   variable has type "int")

process_user_id(user_id)
```

然而我们不能这样做。为什么？因为类型注解不会强制变量使用类型，它告知类型。如果有任何不一致，类型检查器会报告它。事实上，如果这种方法是正确的，整个类型检查思想就会崩溃。

类型检查思想会崩溃，特别是当一个新类型不是旧类型的子类型时（如 `int` 和 `str`）。我们可以想象一种假设的情况，在这种情况下，mypy 会接受类型约束（通过使用注解将类型从更一般的类型变为不太一般的类型）。在我们的例子中，它将会从 `Union[int, None]` 约束到 `int`。但是，目前它不受支持。

至少有两种正确的方法可以通知 mypy 类型检查器不同于预期的类型。

### 1.2 具有类型检查的类型约束 [正确]

更改类型的正确方法是确保新类型的 `isinstance` 返回 `True`：

```python
user_id = get_user_id()

if isinstance(user_id, int):
    process_user_id(user_id)  # 没有错误

# 或者
assert isinstance(user_id, int)
process_user_id(user_id)  # 没有错误

# 在我们的例子中可以
if user_id is not None:
    process_user_id(user_id)  # 没有错误
```

现在，mypy 确信 `user_id` 具有正确的类型——否则，将不会执行对 `process_user_id` 的调用。

请注意，使用 `isinstance` 会带来少量运行时开销。另外，我们还会得到额外的运行时检查，这可能会很有用。

### 1.3 具有强制类型转换的类型约束 [正确]

告诉 mypy 该类型受到约束（或以其他方式更改）的另一个正确方法是使用 `cast` 函数。这在 [PEP 484](https://www.python.org/dev/peps/pep-0484/#casts) 中有明确描述。

```python
from typing import cast

user_id = cast(int, get_user_id())
process_user_id(user_id)  # 没有错误
```

正如我们所见，这个函数是在 `typing` 模块中定义的。类型系统不应对运行时产生任何影响，这个函数保持了这个承诺（除了空函数调用）——在 Python 的源代码中，它被定义为一个身份函数 （删除了 docstring）：

```python
def cast(typ, val):
    return val
```

因此不执行运行时检查。当使用 `cast` 时，对于类型检查者来说，盲目相信这种新类型是一条命令。

请注意，使用 `cast` 可能会掩盖错误：前一种类型——可能是正确的——会被忽略。因此，在某种程度上，它的工作方式类似于 `Any` 和 `#type: ignore`（见下文），因此要小心使用。

## 2. 组合类型并定义类型别名

Python 的类型可以自由组合。想要一个整数、浮点数、字符串或 `None` 的列表吗？只需使用：

```python
List[Union[int, float, str, None]]
```

或者

```python
List[Optional[Union[int, float, str]]]
```

随便你。

由字符串组成的元组和由整数组成的元组列表以及由整数、字符串和字符串列表组成的元组列表怎么样？也很直白😅：

```python
Tuple[str, List[Tuple[int, List[Tuple[int, str, List[str]]]]]]
```

看起来很有趣！不是吗？事实上，我们有时会在程序中使用这些复杂的数据类型。如何将类型注解与它们一起使用，并且不要失去理智？

要使类型更易于管理和阅读，请使用类型别名。要创建别名，只需为变量分配一个类型：`alias = T`。现在我们可以使用 `alias` 代替 `T`。

这里的关键是正确命名别名。大多数情况下，创建反映命名类型结构的别名，如 `ListOfListsOfDictsFromStrToIntOrFloat`，并没有真正意义。要使用内部反映“业务对象”的名称。同样，相应地嵌套别名。像这样：

```python
from typing import List, Tuple

ItemId = int
ItemName = str
ItemTag = str
ItemTags = List[ItemTag]
Item = Tuple[ItemId, ItemName, ItemTags]
Items = List[Item]

OrderId = int
Order = Tuple[OrderId, Items]
Orders = List[Order]

ShipmentId = str
Shipment = Tuple[ShipmentId, Orders]
```

使用 `NamedTuple` 来定义 `Item`、`Order` 和 `Shipment` 将会进一步提高我们代码的可读性。另外，在现实的代码中，我们可能会使用自定义类来代替。尽管如此，类型别名仍然是有用的。

`Shipment` 看起来比 `Tuple[str, List[Tuple[int, List[Tuple[int, str, List[str]]]]]]` 好得多，不是吗？需要键入的字符也少得多。

现在，我们的代码可以用业务相关的类型注解，如下所示：

```python
def generate_item_id() -> ItemId:
    pass

def create_item(name: ItemName, tags: ItemTags) -> Item:
    pass

def create_order(items: Items) -> Order:
    pass

def make_order(order: Order):
    pass

# 等等
```

代码更清晰，业务逻辑明显。此外，类型注解和类型本身的错误更容易被发现。

## 3. NewType 函数

在最后一节中，我们看到为一个简单的类型分配了别名，如 `ItemId = int`。即使是这个简单的别名也有意义，因为它表示这个特定整数的“意义”。尽管如此，它并不能保护我们免受以下错误的影响：

```python
ItemId = int
OrderId = int


def get_last_item_id() -> ItemId:
    pass


def get_last_order_id() -> OrderId:
    pass


def get_order(id_: OrderId):
    pass


order_id = get_last_item_id()
order = get_order(order_id)  # 没有错误
```

mypy 很开心，IDE 很开心，我们也很开心。让我们祈祷进行代码审查的人会发现错误！

为了防止这种情况，我们可以额外定义直接从 `int` 继承的子类（子类型）：

```python
class ItemId(int):
    pass


class OrderId(int):
    pass


def get_last_item_id() -> ItemId:
    pass


def get_last_order_id() -> OrderId:
    pass


def get_order(id_: OrderId):
    pass


order_id = get_last_item_id()
order = get_order(order_id)  # 错误:
# Argument 1 to "get_order" has incompatible type "ItemId"; expected "OrderId"
```

很好，错误被发现了。不幸的是，通过额外的构造函数传递值会带来运行时开销。当我们必须处理许多实例时，这尤其痛苦。为了解决这个问题，`typing` 模块具有 `NewType` 函数。它用于定义不同的子类型：

```python
from typing import NewType

ItemId = NewType("ItemId", int)
OrderId = NewType("OrderId", int)


def get_last_item_id() -> ItemId:
    pass


def get_last_order_id() -> OrderId:
    pass


def get_order(id_: OrderId):
    pass


order_id = get_last_item_id()
order = get_order(order_id)  # 错误:
# Argument 1 to "get_order" has incompatible type "ItemId"; expected "OrderId"
```

`NewType` 只是返回一个身份函数，因此在运行时没有定义子类。此外，这只会带来最小的开销。请参见[这里](https://github.com/python/typing/blob/master/src/typing.py#L2210-L2234)的源代码。

使用用 `NewType` 定义的类型，我们可以在代码中添加额外的类型“跟踪”。这在安全环境中可能很方便——例如区分安全和（潜在的）不安全字符串。

```python
from typing import NewType

SafeStr = NewType('SafeStr', str)

safe_code = SafeStr('2 + 2')
user_provided_code = 'import sys; sys.melt_cpu()'

def exec_code(string: SafeStr):
    exec(string)

exec_code(safe_code)
exec_code(user_provided_code)  # 错误:
# Argument 1 to "exec_code" has incompatible type "str"; expected "SafeStr"
```

请注意，获取 `user_provided_code` 的值可能离对 `exec_code` 的调用很远，因此如果没有 mypy 帮助，很难发现它是不安全的。

在运行时，通过 `NewType`——在我们的例子中是 `SafeStr('2 + 2')`（第 5 行）——传递值几乎没有开销，实际上没有任何变化。对 mypy 来说，它的工作方式就像 `cast(SafeStr, '2 + 2')`。

## 4. 可调用类型

到目前为止，我们定义了函数的参数类型和返回类型。但是，如果我们想将一个函数本身传递给另一个函数（在 Python 中，函数是第一类对象，所以可以这样做）怎么办——我们如何表达传递函数的类型？

让我们离题一会儿……在 Python 的类型系统中，函数的类型就像任何其他类型一样（记住：函数是第一类对象，就像整数或字符串一样）。起初这可能令人惊讶，但如果你仔细想想，这是很自然的。例如，这样注解的函数类型：

```python
def fun(arg: str) -> int: ...
```

可以被认为是“`str` 到 `int`”。事实上，函数的类型在某种程度上非常类似于 `Dict` 类型，它将一个值“映射”到另一个值；像 `Dict[str, int]` 将 `str` 映射到 `int`。从类型的角度来看，函数更复杂——可以没有、一个或多个参数，字典只有一个键。然而，映射的思想是一样的。

在 Python 中，描述函数类型（和其他可调用类型）时使用 `Callable` 类型。它的定义如下：

> `Callable[[t1, t2, …, tn], tr]` 具有位置参数类型 `t1` 等的函数，返回类型 `tr`。[[来源](https://www.python.org/dev/peps/pep-0483/#fundamental-building-blocks)]

让我们看看：

```python
from typing import Callable, List


def apply_function_on_value(func: Callable[[str], int], value: str) -> int:
    return func(value)


def text_length(text: str) -> int:
    return len(text)


result1 = apply_function_on_value(
    func=text_length, value="I know a dead parrot when I see one."
)  # 没有错误
```

`apply_function_on_value` 将一个函数 `func` 作为它第一个参数并将它应用于其第二个参数 `value`。这个函数的类型为“`str` 到 `int`”，或者 `Callable[[str], int]`。所以将 `text_length` 函数（它只是定义在字符串上的 `len` 函数）传递给它是正确的，因为 `func` 被定义为获取 `str` 并返回 `int`。

```python
def append_parrot(text: str) -> str:
    return text + ' Parrot!'

result2 = apply_function_on_value(
    func=append_parrot,
    value="Now that's what I call a dead parrot.",
)  # 错误:
# Argument "func" to "apply_function_on_value" has incompatible type
#   "Callable[[str], str]"; expected "Callable[[str], int]"
```

`append_parrot` 无法正确传递给 `apply_function_on_value`，因为它与 `func` 类型不兼容：返回一个 `str`，而不是 `int`。

## 5. Any 类型与关闭 mypy 检查

Python 的类型规则非常严格，但是 Python 为你不想让 mypy 抱怨某个类型的情况提供了漏洞。这个漏洞是 `Any` 类型。`Any` 与每种类型都一致，每种类型都与 `Any` 一致。

> 当一个值具有类型 `Any` 时，类型检查器将允许对其进行所有操作，并且类型 `Any` 的值可以分配给一个更受约束类型的变量（或者用作返回值）。[[来源](https://www.python.org/dev/peps/pep-0484/#the-any-type)]

让我们证明一下：

```python
from typing import Any

class Dog: ...

# 检查类型
lassie: Dog
anything1: Any
lassie = anything1  # 没有错误

scooby: Dog
anything2: Any
anything2 = scooby  # 没有错误

# 检查属性
anything3: Any
anything3.enter_hiperspace()  # 没有错误
```

正如文档所述：

> `Any` 可以被认为是具有所有值和所有方法的类型。结合上面的子类型定义，这将 `Any` 部分的放在类型层次结构的顶部（它有所有值）和底部（它有所有方法）。[来源](https://www.python.org/dev/peps/pep-0483/#summary-of-gradual-typing)

可以这样表示：

```
                      Any                     <- 所有类型都是 Any
                      / \                        (就像 `object`
                     /*  \*                       -- 所有对象都是 `object`)
                    /     \
            SomeType1     SomeType2
           /       |       |       \
          /        |       |        \
         /         |       |         \
 Subtype1_1  Subtype1_2  Subtype2_1  Subtype2_2
     |          |           |           |
     |*         |*          |*          |*
     |          |           |           |
    Any        Any         Any         Any       <- Any 有所有属性
                                                     (不像 `object`
                                                      -- `object` 没有属性)
* 一致性关系，不是子类型，见下文
```

请注意，严格来说，`Any` 类型和其他类型之间的关系不是微妙的关系，而是保持一致的关系。有关正式定义和更多上下文，请参见[这里](https://www.python.org/dev/peps/pep-0483/#summary-of-gradual-typing)。

实际上，`Any` 只是关闭它正在注解的项目的 mypy 检查。禁用 mypy 检查的另一种更残酷的方法是使用 `# type: ignore`。在你想让 mypy 错误消失的行上使用它：

```python
height: int
height = '25'  # type: ignore
# 没有错误

class Dog: pass

lassie: Dog
lassie.fly()  # type: ignore
# 没有错误
```

`Any` 的使用应该尽可能少，而使用 `# type: ignore` 应该是最后一招。尤其是当你认真对待整个类型工作的时候。

你绝对不应该在下列情况下使用它们：

- 你想推迟注解一些东西。如果你现在不想添加一个类型，就让这个东西没有类型。Python 类型系统是完全可选的——您可以自由注解代码的一部分，而不注解另一部分。
- 你不理解 mypy 的错误信息，只想摆脱它。有时候它可能是太神秘或者太普通了，但是真的值得深入研究这个问题。在我的经验中，几乎每次 mypy 都发现了什么。

`Any` 在哪可能有用？

- 当你真的不知道变量的类型时，使用 `Any`。典型的例子是从外部提供的 JSON 创建的数据。这样创建的变量的类型是什么？它是一个 `list`，`dict`，`int`，`float`，`bool` 还是 `None`？也可以是 `Any`。请注意，使用嵌套类型（`Dict`、`List` 等）正确定义 JSON 格式是不可能的（或者至少非常困难），因为 [JSON 的递归结构](https://www.json.org/)。

`# type: ignore` 在哪可能有用？

- 当 mypy 不知道发生了什么。例如，运行时发生了一些变化——就像方法被动态添加到类中或者类层次结构发生了变化。如果 mypy 的错误真的不太胜任，并且你无法想出另一种方法来处理这个问题，使用 `# type: ignore` 。但是首先，试着理解为什么 mypy 在抱怨。事实上，mypy 是一个非常好的坏味道代码检测器。
- 当某些东西还不兼容 mypy 时（比如 [enums](https://docs.python.org/3.6/library/enum.html)，一开始不支持）。
- mypy 中有 bug。记得在 [github](https://github.com/python/mypy/issues) 上提出这件事！

## Q&A

如果你还不能被说服开始使用 Python 类型系统，请继续阅读。

### 我喜欢我的 Python 代码具有动态性和鸭子类型。类型会毁了这一切。是吗？

首先，mypy 确实不会理解一些与语言的动态特性相关的运行时黑科技。然而，作为回报，你会得到更可靠的代码。

其次，Python 类型系统[支持协议](https://www.python.org/dev/peps/pep-0544/)（也称为“鸭子类型”），既支持内置协议，也支持用户定义的协议 （本主题不在此讨论）。所以不用担心。

### 坦白地说，我不喜欢这整件类型系统的事情。是不是慢慢地将 Python 变成了 Java？

不要担心，与 Java 不同，Python 的类型系统：

- 完全可选（*），
- 不会影响运行时（**）。

（*）好吧，不完全是，`dataclasses` [强迫你使用类型](https://docs.python.org/3/library/dataclasses.html)。希望这是一个异端。

（**）好吧，不完全是…… 首先，从 `typing` 中导入会影响。其次，一些值——例如 `cast` 和 `NewType` ——通过身份函数传递。第三，[泛型类型](https://docs.python.org/3/library/typing.html#typing.Generic)（本文中未涵盖）使用自定义元类，这可能与用户定义的元类冲突；[在 Python 3.7 中，这不再是一个问题](https://www.python.org/dev/peps/pep-0560/)。

### 没错，但是看看类型化的 Python 代码：更像 Java，而不是好的老 Python。

这可能是第一印象，但是你真的看过企业 Java 代码库吗？我不否认类型化的 Python 代码看起来与非类型化代码不同，你需要习惯阅读它。但是当你这样做的时候——并且代码本身是以一种聪明的方式具有类型的（例如，通过使用别名）——它会变得比添加类型之前更加易读易懂。它仍然是 Pythonic 的，因为它增加了代码的清晰性和可读性。

### 好吧，让我试试这个… 我如何开始在代码库中使用类型注解？

我推荐以下步骤：

1. 从在你的非类型化代码上运行 mypy 开始，看看会发生什么。你可能会惊讶于它已经理解了这么多。（您可能希望首先使用 `--ignore-missing-imports` 和 `--follow-imports skip` 参数来运行它。）
2. 修复所有初始的 mypy 错误。
3. 现在，只需开始向代码中添加类型注解。你可以从任何地方开始，但是我认为最好从最重要的代码开始。这样，你的代码库将会更可靠。不要担心错误的类型注解会破坏你的代码：类型会尽可能少地影响 Python 的运行时。
4. 继续添加类型注解。过了一段时间，添加类型注解将开始见效。你可能会收到越来越多的捕捉到真正错误的 mypy 错误。（只是不要忘记运行 mypy。）
5. 下一步是将 mypy 检查添加到 CI 流程（也许是预提交钩子）中，并使代码始终更加可靠。

一般来说，如果你致力于类型，最好注解所有新代码。另一种方法是，当您只想快速原型化功能时，跳过注解阶段。当你知道代码会稳定下来，你可以添加类型注解。

您也可以尝试从使用自动工具添加类型注解开始。[pyannotate](https://github.com/dropbox/pyannotate) 库从运行时观察实际类型中获取类型信息。类似地，[pytest-annotate](https://github.com/kensho-technologies/pytest-annotate) 通过运行测试和观察类型添加类型注解。另外，[pytype](https://github.com/google/pytype) 可以基于代码的静态分析添加注解。我没有使用这些工具的经验，但是我认为仔细验证其中任何一个工具给出的注解是一个好主意。研究这些注解甚至可能导致发现代码中的一些错误。

### 当我没有时间理解 mypy 的错误时该怎么办？

找到时间，它可能会有回报。不要用 `# type: ignore`，但是先试着理解这个问题（有时在 [mypy 问题页面](https://github.com/python/mypy/issues)上搜索是唯一的方法），只有当你确信没有好的选择时，才使用它。mypy 可能，很可能，发现了什么东西。

另一方面，无论代价是什么，不要试图取悦 mypy。经验法则是：不要仅仅为了抑制 mypy 错误而让代码变得更糟。它仍然只是一种工具。

请记住，mypy 正在不断发展。它的错误信息越来越好，误报（不时被发现）也越来越少。所以，记得在新版本发布时更新 mypy。
