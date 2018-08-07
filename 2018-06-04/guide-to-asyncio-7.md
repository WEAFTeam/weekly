---
title: asyncio 不完全指北（七）
author: MisLink
thumbnail: 'https://user-images.githubusercontent.com/8280169/43657842-879ee0e8-9789-11e8-9f1b-3ebbf9ceadc7.png'
abbrlink: a6235d78
date: 2018-06-09 17:35:25
tags:
---

书接上文。

## 使用 aiohttp 作为 Web 服务器

上篇文章中提到，aiohttp 不仅仅是一个 http 客户端，同时也是一个 Web 服务器。在这一节，我们使用 aiohttp 实现一个简单的 Web 程序，同时与 flask 比较一下性能上的差别。

### 准备工作

首先安装我们需要的第三方库：

```shell
pip install aiohttp
pip install flask
```

然后准备好要使用的 Web 容器，这里我们使用对 aiohttp 和 flask 都很友好的 gunicorn。为了让 flask 得到异步支持， 需要同时安装 gevent：

```shell
pip install gunicorn[gevent]
```

安装 wrk，它是一个简单的性能测试工具：

```shell
brew install wrk
```

### Hello, world

我们的起手式当然是 Hello, world。这里，我们分别使用 flask 和 aiohttp 实现一个返回 Hello, world 的 Web 服务。

#### flask

```python
# flask_app.py
from flask import Flask

app = Flask("flask_app")


@app.route("/")
def hello():
    return "Hello, world!"
```

非常简单！

接下来让我们看一下性能测试的结果。首先用 gunicorn 启动应用，将 socket 绑定到 localhost:5000，打开访问日志，使用 4 个 worker，并使用 gevent 作为 worker 的类型：

```shell
gunicorn -b localhost:5000 --access-logfile - -w 4 -k gevent flask_app:app
```

然后就可以进行性能测试了，这里我们使用 8 个线程，每个线程负责 200 个请求，共测试 10 秒，并开启详细日志：

```shell
wrk -t8 -c200 -d10s --latency http://localhost:5000
```

我们会得到这样的结果：

```shell
Running 10s test @ http://localhost:5000
  8 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     9.80ms   94.91ms   1.67s    99.06%
    Req/Sec     1.07k   555.92     2.14k    63.76%
  Latency Distribution
     50%    1.55ms
     75%    1.82ms
     90%    2.27ms
     99%   20.89ms
  23325 requests in 10.01s, 3.96MB read
  Socket errors: connect 0, read 0, write 0, timeout 8
Requests/sec:   2330.74
Transfer/sec:    405.20KB
```

这里只关注几个重要的信息：

- Latency，可以理解为响应时间，wrk 提供了平均值，标准差，最大值，以及正负一个标准差的占比；
- Req/Sec，每个线程每秒钟的完成的请求数，同样有以上数据类型；
- Latency Distribution，响应时间的分布情况，50%、75%、90%、99%的请求在多长时间内结束；
- Socket errors，在测试中有多少错误发生；
- Requests/sec，每秒钟完成多少请求；
- Transfer/sec，每秒钟产生的数据量；

知道了上述信息的含义，就可以对应用程序的性能有一个大概的了解了。

#### aiohttp

同样我们用 aiohttp 实现一个 Hello world 应用：

```python
# aio_app.py
from aiohttp import web

routes = web.RouteTableDef()


@routes.get("/")
async def hello(request):
    return web.Response(text="Hello, world!")


app = web.Application()
app.add_routes(routes)
```

然后用 gunicorn 启动它：

```shell
gunicorn -b localhost:5000 --access-logfile - -w 4 -k aiohttp.GunicornWebWorker aio_app:app
```

大部分参数都相同，唯一的区别是使用了 aiohttp 的 wroker 类型。

也用同样的参数启动 wrk，得到以下结果：

```shell
Running 10s test @ http://localhost:5000
  8 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    32.58ms   15.57ms 120.43ms   68.40%
    Req/Sec   773.91    167.40     1.16k    67.25%
  Latency Distribution
     50%   36.83ms
     75%   42.40ms
     90%   46.73ms
     99%   66.48ms
  61709 requests in 10.03s, 9.65MB read
Requests/sec:   6150.18
Transfer/sec:      0.96MB
```

#### 对比

我们把性能测试的结果放在一块对比一下，左边是 flask，右边是 aiohttp：

```shell
Running 10s test @ http://localhost:5000                    Running 10s test @ http://localhost:5000
  8 threads and 200 connections                               8 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev           Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     9.80ms   94.91ms   1.67s    99.06%              Latency    32.58ms   15.57ms 120.43ms   68.40%
    Req/Sec     1.07k   555.92     2.14k    63.76%              Req/Sec   773.91    167.40     1.16k    67.25%
  Latency Distribution                                        Latency Distribution
     50%    1.55ms                                               50%   36.83ms
     75%    1.82ms                                               75%   42.40ms
     90%    2.27ms                                               90%   46.73ms
     99%   20.89ms                                               99%   66.48ms
  23325 requests in 10.01s, 3.96MB read                       61709 requests in 10.03s, 9.65MB read
  Socket errors: connect 0, read 0, write 0, timeout 8
Requests/sec:   2330.74                                     Requests/sec:   6150.18
Transfer/sec:    405.20KB                                   Transfer/sec:      0.96MB
```

可以看出 flask 在单个请求的耗时上明显胜于 aiohttp，但是标准差巨大，在压力场景下最大耗时长达 1.67s，甚至出现了 8 个超时的连接，而 aiohttp 的请求耗时比较稳定；最重要的区别在于，aiohttp 每秒完成了多达 6150.18 个请求，是 flask 的近 3 倍！flask 中 1% 超过 20.89ms 的请求严重影响了整体的性能。

### 点击计数

通过上面的 Hello, world 程序我们可以发现，使用 aiohttp 可以显著提升 Web 程序的性能。当然 Web 程序并不止于此，它还需要数据库、缓存、消息队列等等组件协同工作。asyncio 的周边虽然在迅速发展，不过仍不完善。好消息是 RabbitMQ 的 Python 驱动在下一个版本也加入了 asyncio 支持，基本组件大部分都支持了 asyncio。在这一节，我们增加 redis 支持，制作一个简单的点击计数器。

#### 准备工作

在本节，我们需要安装 redis，并启动它：

```shell
brew install redis
brew services start redis
```

准备好支持 asyncio 的 redis 驱动：

```shell
pip install aioredis
```

#### 定义 App

```python
from aiohttp import web

app = web.Application()
```

#### 初始化 redis

在 aiohttp 中使用 redis 需要在应用启动前初始化连接池，并在应用退出后关闭连接池：

```python
import aioredis

async def setup_redis(app):
    redis_url = "redis://@localhost/0"
    app["redis"] = await aioredis.create_redis_pool(redis_url)
    yield
    app["redis"].close()
    await app["redis"].wait_closed()
```

并将初始化函数注册到 app 的清理上下文：

```python
app.cleanup_ctx.append(setup_redis)
```

#### 编写路由

```python
routes = web.RouteTableDef()

@routes.get("/hit")
async def hello(request):
    redis = request.app["redis"]
    return web.json_response({"hit": await redis.incr("hit")})

app.add_routes(routes)
```

这样，一个点击计数器就完成了。我们试着请求一下：

```shell
$ http localhost:5000/hit
HTTP/1.1 200 OK
Content-Length: 10
Content-Type: application/json; charset=utf-8
Date: Sat, 09 Jun 2018 08:37:33 GMT
Server: Python/3.6 aiohttp/3.3.1

{
    "hit": 1
}
```

之后每次请求 hit 的值都会 +1。

#### 对比

同样，我也编写了一个 flask 版本的点击计数器，在这里只展示一下对比结果：

aiohttp 版本：

```shell
Running 10s test @ http://localhost:5000/hit
  8 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    35.83ms    7.18ms  90.80ms   70.37%
    Req/Sec   699.60     86.70     0.92k    68.62%
  Latency Distribution
     50%   35.01ms
     75%   40.39ms
     90%   44.61ms
     99%   58.03ms
  55816 requests in 10.04s, 9.10MB read
Requests/sec:   5559.97
Transfer/sec:      0.91MB
```

flask 版本：

```shell
Running 10s test @ http://localhost:5000/hit
  8 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   123.99ms  171.06ms   1.89s    95.32%
    Req/Sec   262.03    163.41   670.00     69.57%
  Latency Distribution
     50%  106.59ms
     75%  156.67ms
     90%  179.60ms
     99%    1.05s
  20484 requests in 10.07s, 3.33MB read
  Socket errors: connect 0, read 19, write 0, timeout 0
Requests/sec:   2033.33
Transfer/sec:    338.51KB
```

可以看出增加了 redis 的内存 io 操作后 aiohttp 的领先优势巨大，而 Web 应用大多是重 io 的，可以预见到 asyncio 在未来会占据更重要的地位。

## 结语

写完这篇，这个系列的文章就到此为止了。整体上 asyncio 仍在继续发展，有越来越多的基础组件已经有了可用的 asyncio 版本，一些大厂已经有转向 asyncio 的趋势了。asyncio 是未来，是一定要了解的~
