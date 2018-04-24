---
title: asyncio 不完全指北（二）
author: MisLink
thumbnail: 'http://ourm7pfm2.bkt.clouddn.com/18-4-16/9617370.jpg'
abbrlink: 84801e4e
date: 2018-04-24 23:43:20
tags:
---

书接上文。

## 调度常规函数

除了管理协程和 I / O 回调之外，`asyncio` 事件循环还可以根据循环中的计时器调度常规函数。

### 立即调度

如果函数执行的时机无关紧要，`call_soon()` 可以用于在事件循环的下一次迭代中调度函数。

```python
import asyncio
import functools


def callback(arg, *, kwarg='default'):
    print(f'callback invoked with {arg} and {kwarg}')


async def main(loop):
    print('registering callbacks')
    loop.call_soon(callback, 1)
    wrapped = functools.partial(callback, kwarg='not default')
    loop.call_soon(wrapped, 2)

    await asyncio.sleep(0.1)


event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    event_loop.run_until_complete(main(event_loop))
finally:
    print('closing event loop')
    event_loop.close()
```

`call_soon()` 的第一个参数为回调函数，剩下的位置参数都会被传递给回调函数。如果想传入关键字参数，就需要用到  `functools.partical()`。

回调函数按照调度顺序被依次调用：

```shell
entering event loop
registering callbacks
callback invoked with 1 and default
callback invoked with 2 and not default
closing event loop
```

### 有延迟的调度

要将回调函数的执行推迟到将来的某个时间，可以使用 `call_later()`。它的第一个参数是以秒为单位的延迟，第二个参数是回调函数。

```python
import asyncio


def callback(n):
    print(f'callback {n} invoked')


async def main(loop):
    print('registering callbacks')
    loop.call_later(0.2, callback, 1)
    loop.call_later(0.1, callback, 2)
    loop.call_soon(callback, 3)

    await asyncio.sleep(0.4)


event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    event_loop.run_until_complete(main(event_loop))
finally:
    print('closing event loop')
    event_loop.close()
```

在这个例子中，相同的回调函数参与了三次调度，分别使用了几个不同的延迟和不同的参数。最后使用了 `call_soon()`，它使回调函数在所有延迟调度之前调用，这表明 `call_soon()` 通常意味着最小延迟：

```shell
entering event loop
registering callbacks
callback 3 invoked
callback 2 invoked
callback 1 invoked
closing event loop
```

### 在特定时间调度

还可以在特定时间调度函数。事件循环使用单调时钟，而不是 Unix 时钟，以确保当前的值永不回归。要在特定的时间调度，必须使用事件循环的` time() `方法。

```python
import asyncio
import time


def callback(n, loop):
    print(f'callback {n} invoked at {loop.time()}')


async def main(loop):
    now = loop.time()
    print(f'clock time: {time.time()}')
    print(f'loop  time: {now}')

    print('registering callbacks')
    loop.call_at(now + 0.2, callback, 1, loop)
    loop.call_at(now + 0.1, callback, 2, loop)
    loop.call_soon(callback, 3, loop)

    await asyncio.sleep(1)


event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    event_loop.run_until_complete(main(event_loop))
finally:
    print('closing event loop')
    event_loop.close()
```

可以注意到事件循环内的时间与 `time.time()` 不一致：

```shell
entering event loop
clock time: 1524502404.7036376
loop  time: 4562.515
registering callbacks
callback 3 invoked at 4562.515
callback 2 invoked at 4562.625
callback 1 invoked at 4562.718
closing event loop
```

## 异步产生结果

`future`是尚未完成的工作的结果。事件循环可以监视`future`对象的状态直到它完成，从而允许应用程序的一部分等待另一部分完成某些工作。

### 等待 `future`

`future`就像协程，所以任何用于处理协程的方法也可以用来处理`future`。这个例子将`future`传递给事件循环的`run_until_complete()`方法。记住我们在上一篇中提到的，通常我们不应该自行创建 `future` 对象，这里只为演示。

```python
import asyncio


def mark_done(future, result):
    print(f'setting future result to {result!r}')
    future.set_result(result)


event_loop = asyncio.get_event_loop()
try:
    all_done = asyncio.Future()

    print('scheduling mark_done')
    event_loop.call_soon(mark_done, all_done, 'the result')

    print('entering event loop')
    result = event_loop.run_until_complete(all_done)
    print(f'returned result: {result!r}')
finally:
    print('closing event loop')
    event_loop.close()

print(f'future result: {all_done.result()!r}')
```

调用 `set_result()` 时，`future` 的状态会被更改为已完成，`future`的实例将保存结果，并返回：

```shell
scheduling mark_done
entering event loop
setting future result to 'the result'
returned result: 'the result'
closing event loop
future result: 'the result'
```

`future` 也可以与 `await` 一起使用：

```python
import asyncio


def mark_done(future, result):
    print(f'setting future result to {result!r}')
    future.set_result(result)


async def main(loop):
    all_done = asyncio.Future()

    print('scheduling mark_done')
    loop.call_soon(mark_done, all_done, 'the result')

    result = await all_done
    print(f'returned result: {result!r}')


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

`future` 的结果由 `await` 返回：

```python
scheduling mark_done
setting future result to 'the result'
returned result: 'the result'
```

### `future` 的回调

除了像协程一样工作之外，`future`还可以在完成时调用回调函数。

```python
import asyncio
import functools


def callback(future, n):
    print(f'{n}: future done: {future.result()}')


async def register_callbacks(all_done):
    print('registering callbacks on future')
    all_done.add_done_callback(functools.partial(callback, n=1))
    all_done.add_done_callback(functools.partial(callback, n=2))


async def main(all_done):
    await register_callbacks(all_done)
    print('setting result of future')
    all_done.set_result('the result')


event_loop = asyncio.get_event_loop()
try:
    all_done = asyncio.Future()
    event_loop.run_until_complete(main(all_done))
finally:
    event_loop.close()
```

添加回调的函数只需要一个参数，即回调函数；回调函数也只接受一个参数，即 `future` 实例。若要传递其他参数给回调函数，要使用 `functools.partical()` 。回调函数按注册顺序被调用：

```shell
registering callbacks on future
setting result of future
1: future done: the result
2: future done: the result
```

## 参考资料

* [Scheduling Calls to Regular Functions](https://pymotw.com/3/asyncio/scheduling.html)
* [Producing Results Asynchronously](https://pymotw.com/3/asyncio/futures.html)
