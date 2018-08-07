---
title: asyncio 不完全指北（五）
author: MisLink
thumbnail: 'https://user-images.githubusercontent.com/8280169/43657842-879ee0e8-9789-11e8-9f1b-3ebbf9ceadc7.png'
abbrlink: 112d613e
date: 2018-05-13 23:09:56
tags:
---


书接上文。

## 用协程和流实现异步 I / O

本节将重新实现 echo 服务器和客户端的两个示例程序，只不过会使用协程和 `asyncio` 流 API 而不是 `Protocol` 和 `Transport` 类抽象。这些示例在比前面讨论的`Protocol` API 更低的抽象级别上操作，但是处理的事件是相似的。

### Echo 服务器

服务器程序首先导入所需的 `asyncio` 和 `logging` 模块，然后创建事件循环对象：

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

然后定义一个协程来处理通信。每次客户端连接时，都会调用协程的新实例，从而在该函数中的代码一次只能与一个客户端通信。Python 的语言运行时管理每个协程实例的状态，因此应用程序代码不需要管理任何额外的数据结构来跟踪单独的客户端。

协程接受的参数是与新连接关联的 `StreamReader`  和 `StreamWriter` 实例。与 `Transport` 一样，可以通过 `writer` 的  `get_extra_info()`  方法访问客户端地址：

```python
async def echo(reader, writer):
    address = writer.get_extra_info('peername')
    log = logging.getLogger('echo_{}_{}'.format(*address))
    log.debug('connection accepted')
```

虽然在建立连接时调用协程，但可能还没有任何要读取的数据。为了避免在读取时阻塞，协程使用 `await read()` 来允许事件循环继续处理其他任务，直到有数据要读取：

```python
    while True:
        data = await reader.read(128)
```

如果客户端发送了数据，则从 `await` 返回数据，并可通过将其传递给 `writer` 发送回客户端。对 `write()` 的多个调用可用于缓冲传出的数据，然后使用 `drain() ` 刷新结果。由于刷新网络 I / O 可能会阻塞，因此再次使用 `await` 来恢复对事件循环的控制，事件循环监视写入 socket，并在可能发送更多数据时调用 `writer`：

```python
        if data:
            log.debug(f'received {data}')
            writer.write(data)
            await writer.drain()
            log.debug(f'sent {data}')
```

如果客户端未发送任何数据，`read()` 将返回一个空字节串，以指示连接已关闭。服务器需要关闭 socket 以写入客户端，然后 协程可以返回以指示它已完成：

```python
        else:
            log.debug('closing')
            writer.close()
            return
```

启动服务器有两个步骤。首先，应用程序告诉事件循环要监听的主机名和 socket，使用协程创建新的服务器对象。 `start_server()`  方法本身就是一个协程，因此必须由事件循环处理结果才能实际启动服务器。完成协程产生了绑定到事件循环的 `asyncio.Server ` 实例：

```python
factory = asyncio.start_server(echo, *SERVER_ADDRESS)
server = event_loop.run_until_complete(factory)
log.debug('starting up on {} port {}'.format(*SERVER_ADDRESS))
```

需要运行事件循环以处理事件和客户端请求。对于长期运行的服务，`run_forever()` 方法是最简单的方法。当事件循环停止时，无论是通过应用程序代码还是通过发信号通知进程，服务器都可以关闭以正确清理 socket，然后可以关闭事件循环以在程序退出之前完成对任何其他事务的处理：

```python
try:
    event_loop.run_forever()
except KeyboardInterrupt:
    pass
finally:
    log.debug('closing server')
    server.close()
    event_loop.run_until_complete(server.wait_closed())
    log.debug('closing event loop')
    event_loop.close()
```

### Echo 客户端

使用协程构建客户端非常类似于构建服务器。代码再次开始于导入 `asyncio` 和 `logging` 模块，然后创建事件循环对象：

```python
import asyncio
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

`echo_client` 协程接受两个参数，告诉它服务器在哪里以及要发送什么消息：

```python
async def echo_client(address, messages):
```

 当任务启动时调用协程，但它没有可用的活动连接。因此，第一步是让客户端建立自己的连接。它使用 `await` 来避免在 `open_connection() ` 协程运行时阻塞其他活动：

```python
    log = logging.getLogger('echo_client')

    log.debug('connecting to {} port {}'.format(*address))
    reader, writer = await asyncio.open_connection(*address)
```

`open_connection()` 协程返回与新 socket 关联的 `StreamReader ` 和 `StreamWriter ` 实例。下一步是使用 `writer` 向服务器发送数据。与服务器一样，`writer` 将缓冲传出的数据，直到 socket 就绪或使用 `drain() ` 刷新结果。由于刷新网络 I / O 可能会阻塞，因此再次使用 `await` 来恢复对事件循环的控制，事件循环监视写入 socket，并在可能发送更多数据时调用 `writer`：

```python
    for msg in messages:
        writer.write(msg)
        log.debug(f'sending {msg}')
    if writer.can_write_eof():
        writer.write_eof()
    await writer.drain()
```

接下来，客户端通过尝试读取数据直到没有要读取的内容来获取来自服务器的响应。为了避免阻塞单个 `read()` 调用，`await` 将控制权交还给事件循环。如果服务器已发送数据，则会记录数据。如果服务器未发送任何数据，`read() ` 将返回一个空字节串，指示连接已关闭。客户端需要关闭 socket 以发送到服务器，然后返回以指示已完成：

```python
    log.debug('waiting for response')
    while True:
        data = await reader.read(128)
        if data:
            log.debug(f'received {data}')
        else:
            log.debug('closing')
            writer.close()
            return
```

要启动客户端，使用协程调用事件循环以创建客户端。使用 `run_until_complete()`  可避免客户端程序中出现无限循环。与`Protocol` 示例不同，协程完成时不需要单独的 `future` 发出信号，因为 `echo_client() ` 包含所有客户端逻辑本身，并且在收到响应并关闭服务器连接之前不会返回：

```python
try:
    event_loop.run_until_complete(echo_client(SERVER_ADDRESS, MESSAGES))
finally:
    log.debug('closing event loop')
    event_loop.close()
```

### 输出

在一个窗口中运行服务器而在另一个窗口中运行客户端。

客户端将产生以下输出：

```shell
asyncio: Using selector: SelectSelector
echo_client: connecting to localhost port 10000
echo_client: sending b'This is the message. '
echo_client: sending b'It will be sent '
echo_client: sending b'in parts.'
echo_client: waiting for response
echo_client: received b'This is the message. It will be sent in parts.'
echo_client: closing
main: closing event loop
```

虽然客户端总是单独发送消息，但客户端第一次运行时，服务器会收到一条大消息，并将该消息返回给客户端。根据网络的繁忙程度以及是否在准备所有数据之前刷新网络缓冲区，这些结果在后续运行中会有所不同：

```shell
asyncio: Using selector: SelectSelector
main: starting up on localhost port 10000
echo_::1_11075: connection accepted
echo_::1_11075: received b'This is the message. It will be sent in parts.'
echo_::1_11075: sent b'This is the message. It will be sent in parts.'
echo_::1_11075: closing
```

```shell
echo_::1_11200: connection accepted
echo_::1_11200: received b'This is the message. It will be sent '
echo_::1_11200: sent b'This is the message. It will be sent '
echo_::1_11200: received b'in parts.'
echo_::1_11200: sent b'in parts.'
echo_::1_11200: closing
```

## 与子进程协作

为了利用现有代码而不重写，或者访问 Python 中不可用的库或功能，我们经常需要使用其他程序或进程。与网络 I / O 一样，`asyncio` 包括两个抽象，用于启动另一个程序，然后与它交互。

### 使用子进程的 Protocol 抽象

这个例子使用协程启动一个进程来运行 Unix 命令 `df`，以便查看在本地磁盘上的可用空间。它使用 `subprocess_exec() ` 启动进程，并将其绑定到知道如何读取 `df` 命令输出并对其进行分析的 `Protocol` 类。`Protocol` 类的方法是根据子进程的 I / O 事件自动调用的。因为 `stdin` 和 `stderr` 参数都设置为 `None`，所以这些通信通道不会连接到新进程：

```python
import asyncio
import functools


async def run_df(loop):
    print('in run_df')

    cmd_done = asyncio.Future(loop=loop)
    factory = functools.partial(DFProtocol, cmd_done)
    proc = loop.subprocess_exec(
        factory,
        'df',
        '-hl',
        stdin=None,
        stderr=None,
    )
    try:
        print('launching process')
        transport, protocol = await proc
        print('waiting for process to complete')
        await cmd_done
    finally:
        transport.close()

    return cmd_done.result()
```

类 `DFProtocol` 继承自 `SubprocessProtocol `，该 `Protocol` 定义了类通过管道与另一进程通信的 API。`done` 参数是调用者用来监视进程是否完成的 `future`：

```python
class DFProtocol(asyncio.SubprocessProtocol):

    FD_NAMES = ['stdin', 'stdout', 'stderr']

    def __init__(self, done_future):
        self.done = done_future
        self.buffer = bytearray()
        super().__init__()
```

与 socket 通信一样，在设置新进程的输入通道时调用 `connection_made() `。`transport` 参数是 `BaseSubprocessTransport` 子类的一个实例。如果进程被配置为接收输入，则它可以读取进程输出的数据并将数据写入进程的输入流：

```python
    def connection_made(self, transport):
        print(f'process started {transport.get_pid()}')
        self.transport = transport
```

当进程生成输出时，`pipe_data_received() ` 将使用发送数据的文件描述符和从管道读取的实际数据作为参数调用。`Protocol`类将进程的标准输出通道的输出保存在缓冲区中，以供以后处理：

```python
    def pipe_data_received(self, fd, data):
        print(f'read {len(data)} bytes from {self.FD_NAMES[fd]}')
        if fd == 1:
            self.buffer.extend(data)
```

当进程终止时，`process_exited() ` 将被调用。通过调用 `get_returncode() ` 可以从 `transport` 对象获得进程的退出代码。在这种情况下，如果没有报告错误，则可以在通过 `future` 实例返回可用输出之前对其进行解码和分析。如果出现错误，则结果为空。设置 `future` 的结果会告诉 `run_df() ` 进程已退出，因此它会清理并返回结果：

```python
    def process_exited(self):
        print('process exited')
        return_code = self.transport.get_returncode()
        print(f'return code {return_code}')
        if not return_code:
            cmd_output = bytes(self.buffer).decode()
            results = self._parse_results(cmd_output)
        else:
            results = []
        self.done.set_result((return_code, results))
```

命令的输出被解析成一系列字典，将每行输出的标题名称映射到值，并返回结果列表：

```python
    def _parse_results(self, output):
        print('parsing results')
        if not output:
            return []
        lines = output.splitlines()
        headers = lines[0].split()
        devices = lines[1:]
        results = [dict(zip(headers, line.split())) for line in devices]
        return results
```

`run_df() ` 协程使用 `run_until_complete() ` 运行，然后检查结果并打印每个设备上的可用空间：

```python
event_loop = asyncio.get_event_loop()
try:
    return_code, results = event_loop.run_until_complete(run_df(event_loop))
finally:
    event_loop.close()

if return_code:
    print(f'error exit {return_code}')
else:
    print('\nFree space:')
    for r in results:
        print(f'{r["Mounted"]:25}: {r["Avail"]}')
```

下面的输出显示了执行步骤的顺序，以及系统中驱动器的可用空间：

```shell
in run_df
launching process
process started 6170
waiting for process to complete
read 375 bytes from stdout
process exited
return code 0
parsing results

Free space:
/                        : 41G
...
```

### 用协程和流调用子进程

若要使用协程直接运行进程，而不是通过 `Protocol` 子类访问进程，请调用 `create_subprocess_exec() `，并指定一个连接到管道的标准输出、标准错误和标准输入。产生子进程的协程的结果是一个 `Process` 实例，可用于操作子进程或与其通信：

```python
import asyncio
import asyncio.subprocess


async def run_df():
    print('in run_df')

    buffer = bytearray()

    create = asyncio.create_subprocess_exec(
        'df',
        '-hl',
        stdout=asyncio.subprocess.PIPE,
    )
    print('launching process')
    proc = await create
    print(f'process started {proc.pid}')
```

 在这个例子中，`df` 除了命令行参数之外不需要任何输入，因此下一步是读取所有输出。对于 `Protocol`，无法控制一次读取多少数据。这个例子中使用了 `readline() `，但也可以直接调用 `read() ` 读取不是按行组织的数据。命令的输出被缓冲，就像 `Protocol` 示例一样，因此稍后可以对其进行分析：

```python
    while True:
        line = await proc.stdout.readline()
        print(f'read {line!r}')
        if not line:
            print('no more output from command')
            break
        buffer.extend(line)
```

`readline() ` 方法在程序已完成不再有输出时返回空字节串。为确保正确清除进程，下一步是等待进程完全退出：

```python
    print('waiting for process to complete')
    await proc.wait()
```

此时可以检查退出状态，以确定是解析输出还是将错误视为未生成输出。解析逻辑与前面的示例相同，但处于独立函数中，因为没有可以包装它的 `Protocol` 类。解析数据后，结果和退出代码将返回给调用方：

```python
    return_code = proc.returncode
    print(f'return code {return_code}')
    if not return_code:
        cmd_output = bytes(buffer).decode()
        results = _parse_results(cmd_output)
    else:
        results = []

    return (return_code, results)
```

主程序看起来类似于基于 `Protocol` 的示例，因为实现的改变被隔离在 `run_df() ` 中：

```python
event_loop = asyncio.get_event_loop()
try:
    return_code, results = event_loop.run_until_complete(run_df())
finally:
    event_loop.close()

if return_code:
    print(f'error exit {return_code}')
else:
    print('\nFree space:')
    for r in results:
        print(f'{r["Mounted"]:25}: {r["Avail"]}')
```

由于 `df` 的输出可以一次读取一行，因此它将显示程序的进度。否则，输出看起来与前面的示例类似：

```shell
in run_df
launching process
process started 7354
read b'Filesystem      Size  Used Avail Use% Mounted on\n'
read b'/dev/vda1        50G  6.0G   41G  13% /\n'
...
read b''
no more output from command
waiting for process to complete
return code 0
parsing results

Free space:
/                        : 41G
...
```

### 向子进程发送数据

前面的两个示例都仅使用单个通信信道来从子进程读取数据。通常需要将数据发送到命令中进行处理。下面将定义一个协程，用于执行 Unix 命令 `tr` 以转换其输入流中的字符。这个例子中`tr` 用于将小写字母转换为大写字母。

`to_upper() ` 协程将输入字符串作为参数。它产生运行 `tr [:lower:] [:upper:] ` 的子进程：

```python
import asyncio
import asyncio.subprocess


async def to_upper(input):
    print('in to_upper')

    create = asyncio.create_subprocess_exec(
        'tr',
        '[:lower:]',
        '[:upper:]',
        stdout=asyncio.subprocess.PIPE,
        stdin=asyncio.subprocess.PIPE,
    )
    print('launching process')
    proc = await create
    print(f'pid {proc.pid}')
```

接下来 `to_upper() ` 使用 `Process` 的 `communicate()` 方法将输入字符串发送到命令，并异步读取所有生成的输出。与 `subprocess.Popen ` 版本的方法相同，`communicate()`  返回完整的输出字节串。如果一个命令可能产生的数据超出了可以充裕的放入内存的范围，或者无法一次产生输入，或者必须增量处理输出，则可以直接使用进程的 `stdin`、`stdout` 和 `stderr` 句柄，而不是调用 `communicate()` ：

```python
    print('communicating with process')
    stdout, stderr = await proc.communicate(input.encode())
```

I / O 完成后，等待进程完全退出可确保进程得到正确清理：

```python
    print('waiting for process to complete')
    await proc.wait()
```

然后可以检查返回代码，并对输出字节串进行解码，以准备协程的返回值：

```python
    return_code = proc.returncode
    print(f'return code {return_code}')
    if not return_code:
        results = bytes(stdout).decode()
    else:
        results = ''

    return (return_code, results)
```

 程序的主要部分构建要转换的消息字符串，然后设置事件循环以运行 `to_upper()` 并打印结果：

```python
MESSAGE = """
This message will be converted
to all caps.
"""

event_loop = asyncio.get_event_loop()
try:
    return_code, results = event_loop.run_until_complete(to_upper(MESSAGE))
finally:
    event_loop.close()

if return_code:
    print(f'error exit {return_code}')
else:
    print(f'Original: {MESSAGE!r}'.format(MESSAGE))
    print(f'Changed : {results!r}')
```

输出显示操作序列，然后显示如何转换简单文本消息：

```shell
in to_upper
launching process
pid 12428
communicating with process
waiting for process to complete
return code 0
Original: '\nThis message will be converted\nto all caps.\n'
Changed : '\nTHIS MESSAGE WILL BE CONVERTED\nTO ALL CAPS.\n'
```

## 接收 Unix 信号

UNIX 系统事件通知通常会中断应用程序，从而触发其处理程序。当与 `asyncio` 一起使用时，信号处理程序回调与事件循环管理的其他协程和回调交错执行。这导致中断函数较少，因此需要提供安全防护来清理不完整的操作。

信号处理程序必须是常规的可调用程序，而不是协程：

```python
import asyncio
import functools
import os
import signal


def signal_handler(name):
    print(f'signal_handler({name!r})')
```

信号处理程序是使用 `add_signal_handler() ` 注册的。第一个参数是信号，第二个参数是回调。回调不传递参数，因此如果需要参数，可以使用 `functools.partical()` 包装函数：

```python
event_loop = asyncio.get_event_loop()
event_loop.add_signal_handler(
    signal.SIGHUP,
    functools.partial(signal_handler, name='SIGHUP'),
)
event_loop.add_signal_handler(
    signal.SIGUSR1,
    functools.partial(signal_handler, name='SIGUSR1'),
)
event_loop.add_signal_handler(
    signal.SIGINT,
    functools.partial(signal_handler, name='SIGINT'),
)
```

本示例程序使用协程通过 `os.kill() ` 向自身发送信号。在发送每个信号之后，协程将让出控制权以允许处理程序执行。在一个正常的应用程序中，会有很多应用程序代码让步给事件循环的地方，而不需要这样的人工让步：

```python
async def send_signals():
    pid = os.getpid()
    print(f'starting send_signals for {pid}')

    for name in ['SIGHUP', 'SIGHUP', 'SIGUSR1', 'SIGINT']:
        print(f'sending {name}')
        os.kill(pid, getattr(signal, name))
        print('yielding control')
        await asyncio.sleep(0.01)
    return
```

 主程序运行 `send_signals() `，直到它发送完所有信号：

```python
try:
    event_loop.run_until_complete(send_signals())
finally:
    event_loop.close()
```

输出显示当 `send_signals() ` 在发送信号后让出控制时如何调用处理程序：

```shell
starting send_signals for 23185
sending SIGHUP
yielding control
signal_handler('SIGHUP')
sending SIGHUP
yielding control
signal_handler('SIGHUP')
sending SIGUSR1
yielding control
signal_handler('SIGUSR1')
sending SIGINT
yielding control
signal_handler('SIGINT')
```

## 将协程与线程和进程相结合

许多现有库尚未准备好与 `asyncio` 配合使用。它们可能会阻塞或依赖模块中不可用的并发功能。通过使用来自 `concurrent.futures` 的 `executor` 在单独的线程或单独的进程中运行代码，仍然可以在基于 `asyncio` 的应用程序中使用这些库。

### 线程

事件循环的 `run_in_executor() ` 方法接受的参数为 `executor` 实例，要调用的常规可调用对象以及要传递给可调用对象的任何参数。它返回一个可用于等待函数完成其工作并返回某些内容的 `future`。如果没有传入 `executor`，则会创建 `ThreadPoolExecutor `。此示例显式创建一个 `executor `，以限制可用的工作线程数。

`ThreadPoolExecutor `启动其工作线程，然后在线程中调用每个提供的函数一次。此示例说明如何将 `run_in_executor() ` 和 `wait() ` 组合起来，以便在阻塞单独线程中运行的函数的同时，对事件循环具有协程让步控制，然后在这些函数完成时将其唤醒：

```python
import asyncio
import concurrent.futures
import logging
import sys
import time


def blocks(n):
    log = logging.getLogger(f'blocks({n})')
    log.info('running')
    time.sleep(0.1)
    log.info('done')
    return n**2


async def run_blocking_tasks(executor):
    log = logging.getLogger('run_blocking_tasks')
    log.info('starting')

    log.info('creating executor tasks')
    loop = asyncio.get_event_loop()
    blocking_tasks = [loop.run_in_executor(executor, blocks, i) for i in range(6)]
    log.info('waiting for executor tasks')
    completed, pending = await asyncio.wait(blocking_tasks)
    results = [t.result() for t in completed]
    log.info(f'results: {results!r}')

    log.info('exiting')


if __name__ == '__main__':
    logging.basicConfig(
        level=logging.INFO,
        format='%(threadName)10s %(name)18s: %(message)s',
        stream=sys.stderr,
    )

    executor = concurrent.futures.ThreadPoolExecutor(max_workers=3,)

    event_loop = asyncio.get_event_loop()
    try:
        event_loop.run_until_complete(run_blocking_tasks(executor))
    finally:
        event_loop.close()
```

这个程序使用 `logging` 来方便地指示哪些线程和函数正在生成的日志消息。因为每次调用 `blocks() ` 时使用单独的 `Logger`，所以输出清楚地显示了相同的线程被重用，以调用具有不同参数的函数的多个副本：

 ```shell
MainThread run_blocking_tasks: starting
MainThread run_blocking_tasks: creating executor tasks
ThreadPoolExecutor-0_0          blocks(0): running
ThreadPoolExecutor-0_1          blocks(1): running
ThreadPoolExecutor-0_2          blocks(2): running
MainThread run_blocking_tasks: waiting for executor tasks
ThreadPoolExecutor-0_0          blocks(0): done
ThreadPoolExecutor-0_0          blocks(3): running
ThreadPoolExecutor-0_1          blocks(1): done
ThreadPoolExecutor-0_2          blocks(2): done
ThreadPoolExecutor-0_1          blocks(4): running
ThreadPoolExecutor-0_2          blocks(5): running
ThreadPoolExecutor-0_0          blocks(3): done
ThreadPoolExecutor-0_1          blocks(4): done
ThreadPoolExecutor-0_2          blocks(5): done
MainThread run_blocking_tasks: results: [16, 25, 1, 4, 0, 9]
MainThread run_blocking_tasks: exiting
 ```

### 进程

`ProcessPoolExecutor ` 的工作方式大致相同，它创建一组工作进程而不是线程。使用单独的进程需要更多的系统资源，但是对于计算密集型操作，在每个 CPU 内核上运行单独的任务是有意义的：

```python
...

if __name__ == '__main__':
    logging.basicConfig(
        level=logging.INFO,
        format='PID %(process)5s %(name)18s: %(message)s',
        stream=sys.stderr,
    )

    executor = concurrent.futures.ProcessPoolExecutor(max_workers=3,)

    event_loop = asyncio.get_event_loop()
    try:
        event_loop.run_until_complete(run_blocking_tasks(executor))
    finally:
        event_loop.close()
```

从线程转移到进程所需的唯一更改是创建不同类型的 `executor`。本示例还将日志记录格式更改为包含进程 id 而不是线程名称，以证明任务实际上正在单独的进程中运行：

```shell
PID 24417 run_blocking_tasks: starting
PID 24417 run_blocking_tasks: creating executor tasks
PID 24417 run_blocking_tasks: waiting for executor tasks
PID 24461          blocks(0): running
PID 24460          blocks(1): running
PID 24459          blocks(2): running
PID 24460          blocks(1): done
PID 24459          blocks(2): done
PID 24460          blocks(3): running
PID 24461          blocks(0): done
PID 24459          blocks(4): running
PID 24461          blocks(5): running
PID 24460          blocks(3): done
PID 24459          blocks(4): done
PID 24461          blocks(5): done
PID 24417 run_blocking_tasks: results: [16, 1, 25, 0, 4, 9]
PID 24417 run_blocking_tasks: exiting
```

## 调试

`asyncio` 内置了几个有用的调试功能。

首先，事件循环使用 `logging` 在运行时发出状态消息。如果在应用程序中启用了日志记录，则其中一些是可用的。其他的可以通过告诉循环发出更多调试消息来打开。调用 `set_debug() `，传递一个布尔值，指示是否应启用调试。

由于基于 `asyncio` 构建的应用程序对无法让出控制的“贪婪”协程非常敏感，因此支持检测事件循环中的缓慢回调。通过启用调试将其打开，并通过将循环的 `slow_callback_duration ` 属性设置为应发出警告的秒数来定义 “缓慢”。

最后，如果使用 `asyncio` 的应用程序在不清理某些协程或其他资源的情况下退出，这可能意味着存在逻辑错误，无法运行某些应用程序代码。启用 `ResourceWarning` 警告会在程序退出时报告这些情况：

```python
import argparse
import asyncio
import logging
import sys
import time
import warnings

parser = argparse.ArgumentParser('debugging asyncio')
parser.add_argument(
    '-v',
    dest='verbose',
    default=False,
    action='store_true',
)
args = parser.parse_args()

logging.basicConfig(
    level=logging.DEBUG,
    format='%(levelname)7s: %(message)s',
    stream=sys.stderr,
)
LOG = logging.getLogger('')


async def inner():
    LOG.info('inner starting')
    # 模拟缓慢的任务
    time.sleep(0.1)
    LOG.info('inner completed')


async def outer(loop):
    LOG.info('outer starting')
    await asyncio.ensure_future(loop.create_task(inner()))
    LOG.info('outer completed')


event_loop = asyncio.get_event_loop()
if args.verbose:
    LOG.info('enabling debugging')

    # 启用调试
    event_loop.set_debug(True)

    # 定义一个很小阈值表示“缓慢”
    event_loop.slow_callback_duration = 0.001

    # 报告管理异步资源的所有错误
    warnings.simplefilter('always', ResourceWarning)

LOG.info('entering event loop')
event_loop.run_until_complete(outer(event_loop))
```

在未启用调试的情况下运行时，此应用程序的所有内容看起来都很好：

```shell
  DEBUG: Using selector: SelectSelector
   INFO: entering event loop
   INFO: outer starting
   INFO: inner starting
   INFO: inner completed
   INFO: outer completed
```

开启调试会暴露出一些问题，包括 `inner() ` 完成所花的时间比设定的 `slow_callback_duration` 还要长，而且当程序结束时，事件循环并未正确关闭：

```shell
  DEBUG: Using selector: SelectSelector
   INFO: enabling debugging
   INFO: entering event loop
   INFO: outer starting
   INFO: inner starting
   INFO: inner completed
WARNING: Executing <Task finished coro=<inner() done, defined at *.py:25> result=None created at *.py:33> took 0.093 seconds
   INFO: outer completed
```
