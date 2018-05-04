---
title: asyncio 不完全指北（四）
author: MisLink
thumbnail: 'http://ourm7pfm2.bkt.clouddn.com/18-4-16/9617370.jpg'
abbrlink: 7eb3a479
date: 2018-05-04 22:32:16
tags:
---

书接上文。

## 同步原语

虽然使用 `asyncio` 的程序通常都以单线程运行，但仍然可以作为并发程序。每个协程或任务可以根据来自 I / O 或其他外部事件的延迟和中断以不可预测的顺序执行。为了支持安全并发，和 `threading`和 `multiprocessing` 模块一样，`asyncio` 包含了一些相同的低级原语的实现。

### 锁

锁对共享资源的访问提供了保护。只有锁的持有者才能使用资源。第二次及以上获取锁的尝试将被阻止，因此每次只有一个持有者：

```python
import asyncio
import functools


def unlock(lock):
    print('callback releasing lock')
    lock.release()


async def coro1(lock):
    print('`coro1` waiting for the lock')
    with await lock:
        print('`coro1` acquired lock')
    print('`coro1` released lock')


async def coro2(lock):
    print('`coro2` waiting for the lock')
    await lock
    try:
        print('`coro2` acquired lock')
    finally:
        print('`coro2` released lock')
        lock.release()


async def main(loop):
    # 创建并持有一个锁
    lock = asyncio.Lock()
    print('acquiring the lock before starting coroutines')
    await lock.acquire()
    print(f'lock acquired: {lock.locked()}')

    # 安排一个回调释放锁
    loop.call_later(0.1, functools.partial(unlock, lock))

    # 运行希望持有锁的协程
    print('waiting for coroutines')
    await asyncio.wait([coro1(lock), coro2(lock)])


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

可以使用 `await` 持有一个锁，并用  `release()` 释放，就像`coro2()` 的做法一样；同时也可以像 `coro1()` 一样，用带有 `await` 的异步上下文处理器来持有并释放一个锁：

```shell
acquiring the lock before starting coroutines
lock acquired: True
waiting for coroutines
`coro1` waiting for the lock
`coro2` waiting for the lock
callback releasing lock
`coro1` acquired lock
`coro1` released lock
`coro2` acquired lock
`coro2` released lock
```

### Event

`asyncio.Event` 与 `threading.Event` 类似，用于允许多个协程等待某个事件发生，而不需要监听一个特定值来实现类似通知的功能：

```python
import asyncio
import functools


def set_event(event):
    print('setting event in callback')
    event.set()


async def coro1(event):
    print('coro1 waiting for event')
    await event.wait()
    print('coro1 triggered')


async def coro2(event):
    print('coro2 waiting for event')
    await event.wait()
    print('coro2 triggered')


async def main(loop):
    event = asyncio.Event()
    print(f'event start state: {event.is_set()}')

    loop.call_later(0.1, functools.partial(set_event, event))

    await asyncio.gather(coro1(event), coro2(event))
    print(f'event end state: {event.is_set()}')


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

与锁一样，`coro1()` 和 `coro2()` 都会等待 `event` 被设置。不同之处在于，它们可以在 `event` 状态发生变化时立即启动，并且它们不需要获取 `event` 对象的唯一使用权：

```shell
event start state: False
coro2 waiting for event
coro1 waiting for event
setting event in callback
coro2 triggered
coro1 triggered
event end state: True
```

### Condition

`Condition`的作用类似于 `Event`，不同之处在于，`Condition` 不会唤醒所有等待中的协程，唤醒的数量由 `notify() ` 的参数控制：

```python
import asyncio


async def consumer(condition, n):
    with await condition:
        print(f'consumer {n} is waiting')
        await condition.wait()
        print(f'consumer {n} triggered')
    print(f'ending consumer {n}')


async def manipulate_condition(condition):
    print('starting manipulate_condition')

    await asyncio.sleep(0.1)

    for i in range(1, 3):
        with await condition:
            print(f'notifying {i} consumers')
            condition.notify(n=i)
        await asyncio.sleep(0.1)

    with await condition:
        print('notifying remaining consumers')
        condition.notify_all()

    print('ending manipulate_condition')


async def main(loop):
    condition = asyncio.Condition()

    consumers = [consumer(condition, i) for i in range(5)]

    loop.create_task(manipulate_condition(condition))

    await asyncio.gather(*consumers)


event_loop = asyncio.get_event_loop()
try:
    result = event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

这个例子中启动了五个 `Condition` 的消费者。每个都使用 `wait()` 方法等待通知它们继续的消息。`manipulate_condition()` 通知一个消费者，然后通知两个消费者，最后通知所有剩余的消费者：

```shell
starting manipulate_condition
notifying 1 consumers
consumer 2 is waiting
consumer 3 is waiting
consumer 0 is waiting
consumer 4 is waiting
consumer 1 is waiting
notifying 2 consumers
consumer 2 triggered
ending consumer 2
consumer 3 triggered
ending consumer 3
notifying remaining consumers
ending manipulate_condition
consumer 0 triggered
ending consumer 0
consumer 4 triggered
ending consumer 4
consumer 1 triggered
ending consumer 1
```

### Queue

`asyncio.Queue` 为协程提供了一个先进先出的数据结构，类似于与多线程中的 `queue.Queue`，多进程中的 `multiprocessing.Queue`：

```python
import asyncio


async def consumer(n, q):
    print(f'consumer {n}: starting')
    while True:
        print(f'consumer {n}: waiting for item')
        item = await q.get()
        print(f'consumer {n}: has item {item}')
        # None 表示终止信号
        if item is None:
            q.task_done()
            break
        else:
            await asyncio.sleep(0.01 * item)
            q.task_done()
    print(f'consumer {n}: ending')


async def producer(q, num_workers):
    print('producer: starting')
    # 向队列中添加一些数据
    for i in range(num_workers * 3):
        await q.put(i)
        print(f'producer: added task {i} to the queue')
    # 传入终止信号
    print('producer: adding stop signals to the queue')
    for i in range(num_workers):
        await q.put(None)
    print('producer: waiting for queue to empty')
    await q.join()
    print('producer: ending')


async def main(loop, num_consumers):
    # 创建指定大小的队列
    # 超过队列大小时生产者会阻塞，直到有消费者取出数据
    q = asyncio.Queue(maxsize=num_consumers)

    # 调度消费者
    consumers = [loop.create_task(consumer(i, q)) for i in range(num_consumers)]

    # 调度生产者
    prod = loop.create_task(producer(q, num_consumers))

    # 等待所有任务完成
    await asyncio.gather(*consumers, prod)


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop, 2))
finally:
    event_loop.close()
```

使用 `put()` 添加项或使用 `get()` 获取并删除项都是异步操作，因为队列大小可能是固定的（阻塞添加操作），或者队列可能是空的（阻塞获取项的操作）：

```shell
consumer 0: starting
consumer 0: waiting for item
consumer 1: starting
consumer 1: waiting for item
producer: starting
producer: added task 0 to the queue
producer: added task 1 to the queue
consumer 0: has item 0
consumer 1: has item 1
producer: added task 2 to the queue
producer: added task 3 to the queue
consumer 0: waiting for item
consumer 0: has item 2
producer: added task 4 to the queue
consumer 1: waiting for item
consumer 1: has item 3
producer: added task 5 to the queue
producer: adding stop signals to the queue
consumer 0: waiting for item
consumer 0: has item 4
consumer 1: waiting for item
consumer 1: has item 5
producer: waiting for queue to empty
consumer 0: waiting for item
consumer 0: has item None
consumer 0: ending
consumer 1: waiting for item
consumer 1: has item None
consumer 1: ending
producer: ending
```

## 用 Protocol 抽象类实现异步 I / O

到目前为止，这些示例都避免了将并发和 I / O 操作混合在一起，一次只关注一个概念。但是，在 I / O 阻塞时切换上下文是 `asyncio` 的主要使用情形之一。在已经介绍的并发概念的基础上，本节将实现简单的 echo 服务器程序和客户端程序。客户端可以连接到服务器，发送一些数据，然后接收与响应相同的数据。每次启动 I / O 操作时，执行代码都会放弃对事件循环的控制，从而允许其他任务运行，直到 I / O 操作就绪。

### Echo 服务器

服务器首先导入所需的 `asyncio` 和 `logging` 模块，然后创建事件循环对象：

```python
import asyncio
import logging
import sys

SERVER_ADDRESS = ('localhost', 10000)

logging.basicConfig(
    level=logging.DEBUG,
    format='%(name)s: %(message)s',
    stream=sys.stderr,
)
log = logging.getLogger('main')

event_loop = asyncio.get_event_loop()
```

然后定义了一个 `asyncio.Protocol` 的子类，用来处理与客户端的通信。`Protocol` 对象的方法是基于与服务器 socket 关联的事件调用的：

```python
class EchoServer(asyncio.Protocol):
```

每个新的客户端连接都会触发对 `connection_made() ` 的调用。`transport` 参数是`asyncio.Transport` 的实例，它提供了使用 socket 进行异步 I / O 的抽象。不同类型的通信提供不同的 `tansport` 实现，所有这些实现都具有相同的 API。例如，有单独的 `transport` 类用于与 socket 通信、与子进程通过管道通信。传入客户端的地址可以通过 `transport` 的 `get_extra_info()` 获取，这是一种特定于实现的方法：

```python
    def connection_made(self, transport):
        self.transport = transport
        self.address = transport.get_extra_info('peername')
        self.log = logging.getLogger('EchoServer_{}_{}'.format(*self.address))
        self.log.debug('connection accepted')
```

建立连接后，当数据从客户端发送到服务器时，将调用协议的  `data_received()` 方法将数据传入以进行处理。数据以字节串的形式传递，由应用程序以适当的方式对其进行解码。在这里记录结果，然后通过调用 `transport.write()` 立即将响应发送回客户端：

```python
    def data_received(self, data):
        self.log.debug('received {!r}'.format(data))
        self.transport.write(data)
        self.log.debug('sent {!r}'.format(data))
```

某些 `transport` 支持特殊的文件结束标识符（EOF）。遇到 EOF 时，将调用 `eof_received()` 方法。在这个实现中，EOF 被发送回客户端来表示它已被接收。由于并非所有 `transport` 都支持显式 EOF，因此 `protocol` 首先询问 `transport` 发送 EOF 是否安全：

```python
    def eof_received(self):
        self.log.debug('received EOF')
        if self.transport.can_write_eof():
            self.transport.write_eof()
```

当连接关闭时，无论是正常关闭还是错误关闭，都会调用 `protocol` 的  `connection_lost()`  方法。如果发生错误，参数会包含适当的异常对象，否则为 `None`：

```python
    def connection_lost(self, error):
        if error:
            self.log.error(f'ERROR: {error}')
        else:
            self.log.debug('closing')
        super().connection_lost(error)
```

启动服务器有两个步骤。首先，应用程序告诉事件循环要使用的 `protocol` 类以及要侦听的主机名和 socket，用来创建新的服务器对象。`create_server()` 方法是协程，因此必须由事件循环处理结果，才能真正的启动服务器。然后，协程完成后产生了一个绑定到事件循环的 `asyncio.Server` 实例：

```python
factory = event_loop.create_server(EchoServer, *SERVER_ADDRESS)
server = event_loop.run_until_complete(factory)
log.debug('starting up on {} port {}'.format(*SERVER_ADDRESS))
```

然后，需要运行事件循环以处理事件和客户端请求。对于长期运行的服务，`run_forever()` 方法是最简单的方法。当事件循环停止时，无论是通过应用程序代码还是通过发信号通知进程，服务器都可以关闭，以便正确清理 socket，然后可以关闭事件循环，以便在程序退出之前完成对任何其他事务的处理：

```python
try:
    event_loop.run_forever()
finally:
    log.debug('closing server')
    server.close()
    event_loop.run_until_complete(server.wait_closed())
    log.debug('closing event loop')
    event_loop.close()
```

### Echo 客户端

使用 `protocol` 类构造客户端非常类似于构造服务器。首先导入所需的 `asyncio` 和 `logging` 模块，然后创建事件循环对象：

```python
import asyncio
import functools
import logging
import sys

MESSAGES = [
    b'This is the message. ',
    b'It will be sent ',
    b'in parts.',
]
SERVER_ADDRESS = ('localhost', 10000)

logging.basicConfig(
    level=logging.DEBUG,
    format='%(name)s: %(message)s',
    stream=sys.stderr,
)
log = logging.getLogger('main')

event_loop = asyncio.get_event_loop()
```

客户端 `protocol` 类定义了与服务器相同的方法，但实现方式不同。类构造函数接受两个参数，一个是要发送的消息列表，另一个是 `future` 的实例，用于通过接收来自服务器的响应来表明客户端已经完成了一个工作周期：

```python
class EchoClient(asyncio.Protocol):

    def __init__(self, messages, future):
        super().__init__()
        self.messages = messages
        self.log = logging.getLogger('EchoClient')
        self.f = future
```

当客户端成功连接到服务器时，它将立即开始通信。消息序列一次发送一条，尽管底层网络代码可以将多个消息组合成一个传输。当所有消息都用尽时，将发送 EOF。

虽然看起来数据都是立即发送的，但实际上 `transport` 对象缓冲传出的数据，并在当 socket 的缓冲区准备好接收数据时设置回调来进行实际的传输。所有这些都是透明处理的，因此可以编写应用程序代码，就好像 I / O 操作正在立即发生一样：

```python
    def connection_made(self, transport):
        self.transport = transport
        self.address = transport.get_extra_info('peername')
        self.log.debug('connecting to {} port {}'.format(*self.address))

        # 这里可以是 transport.writelines()
        # 但这会使显示要发送的消息的每个部分变得更加困难
        for msg in self.messages:
            transport.write(msg)
            self.log.debug(f'sending {msg!r}')
        if transport.can_write_eof():
            transport.write_eof()
```

收到来自服务器的响应时，将记录该响应：

```python
    def data_received(self, data):
        self.log.debug(f'received {data!r}')
```

当从服务器端接收到 EOF 或者连接被关闭时，本地 `transport` 对象被关闭，并通过设置结果将 `future` 对象标记为完成：

```python
    def eof_received(self):
        self.log.debug('received EOF')
        self.transport.close()
        if not self.f.done():
            self.f.set_result(True)

    def connection_lost(self, exc):
        self.log.debug('server closed connection')
        self.transport.close()
        if not self.f.done():
            self.f.set_result(True)
        super().connection_lost(exc)
```

通常，`protocol` 类被传递到事件循环以创建连接。在这种情况下，由于事件循环没有向 `protocol` 构造函数传递额外参数的工具，因此需要 `functools.partial()` 来包装客户端类，并传递要发送的消息列表和 `future` 的实例。然后，在调用 `create_connection()` 建立客户端连接时，将使用该新的可调用对象代替 `protocol` 类：

```python
client_completed = asyncio.Future()

client_factory = functools.partial(
    EchoClient,
    messages=MESSAGES,
    future=client_completed,
)
factory_coroutine = event_loop.create_connection(
    client_factory,
    *SERVER_ADDRESS,
)
```

为了触发客户端运行，事件循环将调用一次创建客户端的协程，然后再调用一次指定给客户端的 `future` 实例，以便在完成后进行通信。使用这样的两个调用避免了客户端程序中的无限循环，客户端程序可能希望在完成与服务器的通信后退出。如果仅使用第一个调用来等待协程创建客户端，那它可能无法处理所有响应数据并正确清理与服务器的连接：

```python
log.debug('waiting for client to complete')
try:
    event_loop.run_until_complete(factory_coroutine)
    event_loop.run_until_complete(client_completed)
finally:
    log.debug('closing event loop')
    event_loop.close()
```

### 输出

在一个窗口中运行服务器而在另一个窗口中运行客户端。

客户端将产生以下输出：

```shell
asyncio: Using selector: KqueueSelector
main: waiting for client to complete
EchoClient: connecting to ::1 port 10000
EchoClient: sending b'This is the message. '
EchoClient: sending b'It will be sent '
EchoClient: sending b'in parts.'
EchoClient: received b'This is the message. It will be sent in parts.'
EchoClient: received EOF
EchoClient: server closed connection
main: closing event loop
```

虽然客户端总是单独发送消息，但客户端第一次运行时，服务器会收到一条大消息，并将该消息返回给客户端。根据网络的繁忙程度以及是否在准备所有数据之前刷新网络缓冲区，这些结果在后续运行中会有所不同：

```shell
asyncio: Using selector: KqueueSelector
main: starting up on localhost port 10000
EchoServer_::1_55307: connection accepted
EchoServer_::1_55307: received b'This is the message. It will be sent in part
s.'
EchoServer_::1_55307: sent b'This is the message. It will be sent in parts.'
EchoServer_::1_55307: received EOF
EchoServer_::1_55307: closing
```

```shell
EchoServer_::1_55309: connection accepted
EchoServer_::1_55309: received b'This is the message. '
EchoServer_::1_55309: sent b'This is the message. '
EchoServer_::1_55309: received b'It will be sent in parts.'
EchoServer_::1_55309: sent b'It will be sent in parts.'
EchoServer_::1_55309: received EOF
EchoServer_::1_55309: closing
```
