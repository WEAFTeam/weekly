---
title: asyncio 不完全指北（一）
abbrlink: b496f296
date: 2018-04-15 03:37:56
tags:
author: MisLink
thumbnail: http://ourm7pfm2.bkt.clouddn.com/18-4-16/9617370.jpg
---

## 前言

众所周知，Python 的并发编程主要由线程、进程和协程三个组件组成，我们可以使用 Python 模块 `threading`、`multiprocessing` 和 `yield` 句法去操纵它们。后来，又有了更高层的封装：`concurrent.futures` 和 `asyncio` 模块。`concurrent.futures`是对 `threading` 和 `multiprocessing` 的封装，不是这篇文章的重点；`asyncio` 是 Python 中的重大变化，也代表了未来的发展趋势，所以这篇文章打算讲讲 `asyncio`。

## 什么是 asyncio

`asyncio` 一开始是 Python 的作者 Guido van Rossum 在 Python 仓库之外开发的，代号为“[Tulip](https://code.google.com/p/tulip/)”，在 Python 3.4 时加入标准库。`asyncio `模块使用事件循环驱动的协程实现并发，提供了基于协程来构建并发程序的工具。作为对比，`threading` 模块通过应用级线程实现并发；`multiprocessing` 模块使用系统级进程实现并发；而 `asyncio `使用单线程、单进程，其中应用程序的各个部分在事件循环的驱动下进行协作，在最佳时间显式切换任务。`asyncio` 不但支持通常情况下出现阻塞型 IO 时的上下文切换，还支持调度，来让代码在指定的将来时间运行，并且还可以让一个协程等待另一个协程完成。

## asyncio 中的几个概念

事件循环：`asyncio ` 提供的框架以事件循环为中心，它是负责有效处理 I / O 事件、系统事件和应用程序上下文切换的第一类对象。Python 提供了几个循环实现，通常会自动选择合理的默认值，但也可以选择特定的事件循环实现。同样也有一些第三方的实现，例如 [uvloop](https://github.com/MagicStack/uvloop)。应用程序将要执行的代码注册到事件循环中，代表允许事件循环在必要时对代码进行调用。当调用结束，或无法继续时，应用程序会让出控制权，交还给事件循环。

协程（coroutine ）：将控制权交还给事件循环的机制来自于 Python 的协程，这是一种特殊的函数，它将控制权交还给调用方而不会丢失本身状态。协程类似于生成器函数，事实上可以在 Python 3.5 之前的版本中用生成器实现协程。`asyncio` 还为 `Protocols` 和 `Transports` 提供了基于类的抽象层，使用回调的代码风格。在基于类的模型和协程模型中，通过重新进入事件循环来显式更改上下文将取代 Python 线程实现中的隐式上下文更改。

future：future 是一种对象，表示待完成的操作。事件循环可以监视将要完成的 `future` 对象，从而允许应用程序的一部分等待另一部分完成某些工作。除了 future，`asyncio` 还包括其他并发原语，例如锁和信号量。通常情况下，我们不应该自行创建 future，只能通过并发框架（例如 `asyncio`）来实例化。原因是 future 代表终将发生的事情，而某件事的发生，是通过安排好这件事的执行时间来确定的。只有当我们把某件事交给事件循环处理时，事件循环才会给这件事排期，从而创建一个 future 对象。

任务（Task）：任务是 future 的子类，它包装并管理协程的执行。任务可以通过事件循环进行调度，以便在它们需要的资源可用时运行，并生成可由其他协程使用的结果。

## 使用协程处理多任务协作

协程是为并发设计的语言概念。协程函数在调用时创建协程对象，然后调用方可以使用协程的 `send ()` 方法运行函数。协程可以在另一个协程中使用 `await `关键字暂停执行。当它被暂停时，协程的状态被保持，允许它在下次被唤醒时恢复到它停止的位置。

### 启动协程

启动一个协程最简单的方式是将一个协程传递给事件循环的 `run_until_complete()` 方法：

```python
import asyncio


async def coroutine():
    print('in coroutine')


event_loop = asyncio.get_event_loop()
try:
    print('starting coroutine')
    print('entering event loop')
    event_loop.run_until_complete(coroutine())
finally:
    print('closing event loop')
    event_loop.close()
```

首先，我们通过 `asyncio.get_event_loop()` 获取了一个默认事件循环的引用。`run_until_complete()` 方法接受一个协程对象，并用它启动事件循环，然后在协程通过 `return` 结束时停止事件循环。

### 协程的返回值

协程的返回值返回给启动并等待它的程序：

```python
import asyncio


async def coroutine():
    print('in coroutine')
    return 'result'


event_loop = asyncio.get_event_loop()
try:
    return_value = event_loop.run_until_complete(coroutine())
    print(f'it returned: {return_value!r}')
finally:
    event_loop.close()
```

### 链式调用协程

一个协程可以启动另一个协程并等待它返回结果。下面的示例包含两个阶段，它们必须按顺序执行，但可以与另外的操作同时运行：

```python
import asyncio


async def phase1():
    print('in phase1')
    return 'result1'


async def phase2(arg):
    print('in phase2')
    return f'result2 derived from {arg}'


async def main():
    print('in main')
    print('waiting for result1')
    result1 = await phase1()
    print('waiting for result2')
    result2 = await phase2(result1)
    return (result1, result2)


event_loop = asyncio.get_event_loop()
try:
    return_value = event_loop.run_until_complete(main())
    print(f'return value: {return_value!r}')
finally:
    event_loop.close()
```

在这里使用了 `await` 关键字，并没有将新的协程添加到事件循环中。因为控制流已经在由事件循环管理的协程内部，所以不需要通知事件循环管理新的协程。

### 使用生成器语法

`async` 和 `await` 关键字出现于 Python 3.5，对于 Python 3.5 之前的版本，可以使用 `asyncio.coroutine` 装饰器和 `yield from` 来实现相同的功能：

```python
import asyncio


@asyncio.coroutine
def phase1():
    print('in phase1')
    return 'result1'


@asyncio.coroutine
def phase2(arg):
    print('in phase2')
    return f'result2 derived from {arg}'


@asyncio.coroutine
def main():
    print('in main')
    print('waiting for result1')
    result1 = yield from phase1()
    print('waiting for result2')
    result2 = yield from phase2(result1)
    return (result1, result2)


event_loop = asyncio.get_event_loop()
try:
    return_value = event_loop.run_until_complete(main())
    print(f'return value: {return_value!r}')
finally:
    event_loop.close()
```

## 参考资料

* [Asynchronous Concurrency Concepts](https://pymotw.com/3/asyncio/concepts.html)
* [Cooperative Multitasking with Coroutines](https://pymotw.com/3/asyncio/coroutines.html)
