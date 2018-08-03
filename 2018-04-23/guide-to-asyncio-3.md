---
title: asyncio 不完全指北（三）
author: MisLink
thumbnail: 'https://user-images.githubusercontent.com/8280169/43657842-879ee0e8-9789-11e8-9f1b-3ebbf9ceadc7.png'
abbrlink: c99ebd1c
date: 2018-05-01 11:58:22
tags:
---

书接上文。

## 并行执行任务

任务是与事件循环交互的主要方式之一。任务包装协程并跟踪它们完成的时间。任务是 `future` 的子类，因此其它协程可以等待任务，并且每个任务都有一个结果，可以在任务完成后获取。

### 启动任务

使用 `create_task()` 创建任务实例。只要事件循环正在运行且协程不返回，生成的任务将作为事件循环管理的并发操作的一部分运行：

```python
import asyncio


async def task_func():
    print('in task_func')
    return 'the result'


async def main(loop):
    print('creating task')
    task = loop.create_task(task_func())
    print(f'waiting for {task!r}')
    return_value = await task
    print(f'task completed {task!r}')
    print(f'return value: {return_value!r}')


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

`main()` 函数在退出前等待任务返回结果：

```shell
creating task
waiting for <Task pending coro=<task_func() running at *.py:4>>
in task_func
task completed <Task finished coro=<task_func() done, defined at *.py:4> result='the result'>
return value: 'the result'
```

### 取消任务

通过保留 `create_task()` 返回的任务对象，可以在任务完成之前取消其操作：

```python
import asyncio


async def task_func():
    print('in task_func')
    return 'the result'


async def main(loop):
    print('creating task')
    task = loop.create_task(task_func())

    print('canceling task')
    task.cancel()

    print(f'canceled task {task!r}')
    try:
        await task
    except asyncio.CancelledError:
        print('caught error from canceled task')
    else:
        print(f'task result: {task.result()!r}')


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

在启动事件循环之前取消任务时，`await task` 会抛出 `CancelledError` 异常：

```shell
creating task
canceling task
canceled task <Task cancelling coro=<task_func() running at *.py:4>>
caught error from canceled task
```

如果某个任务在等待另一个并发操作时被取消，则会通过在其等待的位置抛出 `CancelledError` 异常来通知该任务：

```python
import asyncio


async def task_func():
    print('in task_func, sleeping')
    try:
        await asyncio.sleep(1)
    except asyncio.CancelledError:
        print('task_func was canceled')
        raise
    return 'the result'


def task_canceller(t):
    print('in task_canceller')
    t.cancel()
    print('canceled the task')


async def main(loop):
    print('creating task')
    task = loop.create_task(task_func())
    loop.call_soon(task_canceller, task)
    try:
        await task
    except asyncio.CancelledError:
        print('main() also sees task as canceled')


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

捕获该异常可以清理已完成工作：

```shell
creating task
in task_func, sleeping
in task_canceller
canceled the task
task_func was canceled
main() also sees task as canceled
```

### 从协程创建任务

`ensure_future()` 返回一个与协程的执行相关联的任务。然后，可以将该任务实例传递给其他代码，这些代码可以在不知道原始的协程是如何构造或调用的情况下等待它：

```python
import asyncio


async def wrapped():
    print('wrapped')
    return 'result'


async def inner(task):
    print('inner: starting')
    print(f'inner: waiting for {task!r}')
    result = await task
    print(f'inner: task returned {result!r}')


async def starter():
    print('starter: creating task')
    task = asyncio.ensure_future(wrapped())
    print('starter: waiting for inner')
    await inner(task)
    print('starter: inner returned')


event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    result = event_loop.run_until_complete(starter())
finally:
    event_loop.close()
```

可以注意到传入 `ensure_future()` 的协程不会马上启动，而是直到某个地方用 `await` 调用了用它创建的任务：

```shell
entering event loop
starter: creating task
starter: waiting for inner
inner: starting
inner: waiting for <Task pending coro=<wrapped() running at *.py:4>>
wrapped
inner: task returned 'result'
starter: inner returned
```

## 用控制结构组合协程

一系列线性执行的协程可以很方便的使用关键字 `await` 管理。对于复杂的控制结构，例如一个协程等待其他几个协程并行完成，也可以用 `asyncio` 中的工具实现。

### 等待多个协程

将一个操作分成许多部分并分别执行它们是很常见的场景。例如，下载多个远程资源，或查询远程 API。在执行顺序无关紧要，并且可能存在任意数量的操作的情况下，`wait()` 可以用于暂停一个协程，直到其他后台操作完成：

```python
import asyncio


async def phase(i):
    print(f'in phase {i}')
    await asyncio.sleep(0.1 * i)
    print(f'done with phase {i}')
    return f'phase {i} result'


async def main(num_phases):
    print('starting main')
    phases = [phase(i) for i in range(num_phases)]
    print('waiting for phases to complete')
    completed, pending = await asyncio.wait(phases)
    results = [t.result() for t in completed]
    print(f'results: {results!r}')


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(3))
finally:
    event_loop.close()
```

在内部，`wait()` 使用一个集合来保存它创建的任务实例，所以任务的执行顺序是无序的。`wait()` 的返回值是一个包含两个集合的元组，第一个保存了状态为 `done` 的任务，第二个保存了状态为 `pending` 的任务。

```shell
starting main
waiting for phases to complete
in phase 1
in phase 0
in phase 2
done with phase 0
done with phase 1
done with phase 2
results: ['phase 1 result', 'phase 0 result', 'phase 2 result']
```

调用 `wait()` 时如果指定了 `timeout` 参数，才会出现状态为 `pending` 的任务：

```python
import asyncio


async def phase(i):
    print(f'in phase {i}')
    try:
        await asyncio.sleep(0.1 * i)
    except asyncio.CancelledError:
        print(f'phase {i} canceled')
        raise
    else:
        print(f'done with phase {i}')
        return f'phase {i} result'


async def main(num_phases):
    print('starting main')
    phases = [phase(i) for i in range(num_phases)]
    print('waiting 0.1 for phases to complete')
    completed, pending = await asyncio.wait(phases, timeout=0.1)
    print(f'{len(completed)} completed and {len(pending)} pending')
    # 取消状态为 pending 的任务
    if pending:
        print('canceling tasks')
        for t in pending:
            t.cancel()
    print('exiting main')


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(3))
finally:
    event_loop.close()
```

这些状态为 `pending` 的任务应被取消或者继续等待它们完成。事件循环继续运行时这些任务将继续执行，如果`wait()` 函数的完成被认为所有操作都已经终止了，那这样的结果是不正确的；如果在事件循环结束时仍未完成这些任务，则会生成警告。所以有必要在 `wait()` 函数结束后取消所有状态为 `pending` 的任务。

```shell
starting main
waiting 0.1 for phases to complete
in phase 0
in phase 1
in phase 2
done with phase 0
1 completed and 2 pending
canceling tasks
exiting main
phase 1 canceled
phase 2 canceled
```

### 收集协程的结果

如果要执行的多个协程已经被定义好，并且只关心它们的结果，那么 `gather()` 是一种比较好的收集结果的方法：

```python
import asyncio


async def phase1():
    print('in phase1')
    await asyncio.sleep(2)
    print('done with phase1')
    return 'phase1 result'


async def phase2():
    print('in phase2')
    await asyncio.sleep(1)
    print('done with phase2')
    return 'phase2 result'


async def main():
    print('starting main')
    print('waiting for phases to complete')
    results = await asyncio.gather(
        phase1(),
        phase2(),
    )
    print(f'results: {results!r}')


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main())
finally:
    event_loop.close()
```

`gather()` 创建的任务不会公开，因此无法取消。返回值是一个结果列表，顺序与传递给 `gather()` 的参数的顺序相同，与实际完成的顺序无关：

```shel
starting main
waiting for phases to complete
in phase2
in phase1
done with phase2
done with phase1
results: ['phase1 result', 'phase2 result']
```

### 在任务完成后做一些事

`as_completed()` 是一个生成器，它管理当作参数传递给它的协程列表的执行，每次迭代都会产生一个执行完的协程。与 `wait()` 一样，`as_completed()` 也不保证顺序；与 `wait()` 不同的是它不会等到所有后台操作完成后才可以执行其它操作：

```python
import asyncio


async def phase(i):
    print(f'in phase {i}')
    await asyncio.sleep(0.5 - (0.1 * i))
    print(f'done with phase {i}')
    return f'phase {i} result'


async def main(num_phases):
    print('starting main')
    phases = [phase(i) for i in range(num_phases)]
    print('waiting for phases to complete')
    results = []
    for next_to_complete in asyncio.as_completed(phases):
        print(f'start {next_to_complete}')
        answer = await next_to_complete
        print(f'received answer {answer!r}')
        results.append(answer)
    print(f'results: {results!r}')
    return results


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(3))
finally:
    event_loop.close()
```

这个例子启动了几个协程，这些协程以它们开始顺序的相反顺序结束。当生成器被消耗时，循环使用 `await `等待协程的结果：

```shell
starting main
waiting for phases to complete
in phase 0
in phase 1
in phase 2
done with phase 2
received answer 'phase 2 result'
done with phase 1
received answer 'phase 1 result'
done with phase 0
received answer 'phase 0 result'
results: ['phase 2 result', 'phase 1 result', 'phase 0 result']
```

## 参考资料

* [Executing Tasks Concurrently](https://pymotw.com/3/asyncio/tasks.html)
* [Composing Coroutines with Control Structures](https://pymotw.com/3/asyncio/control.html)
