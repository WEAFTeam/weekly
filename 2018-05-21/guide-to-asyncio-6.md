---
title: asyncio 不完全指北（六）
author: MisLink
thumbnail: 'https://user-images.githubusercontent.com/8280169/43657842-879ee0e8-9789-11e8-9f1b-3ebbf9ceadc7.png'
abbrlink: 74232ae7
date: 2018-05-27 18:39:08
tags:
---

## 前言

前五篇文章介绍了 `asyncio` 的 API，从这篇开始，就要讲一些 Real World（并不）的东西了。

## 使用 aiohttp 作为 HTTP 客户端

[aiohttp](https://docs.aiohttp.org/en/stable/index.html) 是一个基于 `asyncio` 的异步 HTTP 客户端和服务器库，也是 `asyncio` 生态中发展最迅速的第三方库之一。在这一节，我们使用 aiohttp 作为 HTTP 客户端来比较一下同步、基于线程的异步和基于 `asyncio` 的异步的差别。

### 准备工作

首先我们安装好所需的第三方库：

```shell
pip install requests
pip install aiohttp
```

准备一些用于并发请求的 url，共 45 个：

```python
# url.py
urls = [
    'http://caipiao.hao123.com/', 'http://game.hao123.com/',
    'http://mail.10086.cn/', 'http://mail.126.com/', 'http://mail.163.com/',
    'http://mail.aliyun.com/', 'http://mail.qq.com/',
    'http://mail.sina.com.cn/', 'http://music.163.com/',
    'http://tuijian.hao123.com/', 'http://www.12306.cn/', 'http://www.163.com/',
    'http://www.37.com/', 'http://www.4399.com/', 'http://www.abchina.com/',
    'http://www.baidu.com/', 'http://www.bankcomm.com/', 'http://www.boc.cn/',
    'http://www.ccb.com/', 'http://www.chsi.com.cn/',
    'http://www.cmbchina.com/', 'http://www.cnki.net/',
    'http://www.eastmoney.com/', 'http://www.fang.com/',
    'http://www.icbc.com.cn/icbc/', 'http://www.ifeng.com',
    'http://www.iqiyi.com/', 'http://www.psbc.com/', 'http://www.qq.com/',
    'http://www.sina.com.cn/', 'http://www.sohu.com/', 'http://www.tianya.cn/',
    'http://www.zhihu.com/', 'http://wyyx.hao123.com/', 'https://mail.qq.com/',
    'https://mail.sohu.com/', 'https://tieba.baidu.com/', 'https://weibo.com/',
    'https://www.autohome.com.cn/', 'https://www.bilibili.com/',
    'https://www.booking.com/', 'https://www.douyu.com/',
    'https://www.qunar.com/', 'https://www.suning.com/',
    'https://www.taobao.com/'
]
```

### 同步的请求

首先导入所需的库：

```python
import time
import requests
from url import urls
```

完成请求单个 url 的函数，这个函数会以 `bytes` 形式返回网站内容：

```python
def fetch(session, url):
    resp = session.get(url)
    return resp.content
```

同步请求所有的 url，打印出字节的长度：

```python
def main():
    session = requests.Session()
    for url in urls:
        data = fetch(session, url)
        print(f'{url}: {len(data)}')
```

记录完成请求所需的时间：

```python
if __name__ == '__main__':
    start = time.time()
    main()
    print(time.time() - start)
```

上述代码的结果：

```shell
http://caipiao.hao123.com/: 109299
http://game.hao123.com/: 179707
http://mail.10086.cn/: 52500
http://mail.126.com/: 13063
http://mail.163.com/: 137118
http://mail.aliyun.com/: 725
http://mail.qq.com/: 8206
http://mail.sina.com.cn/: 2837
http://music.163.com/: 92606
...
https://www.autohome.com.cn/: 656200
https://www.bilibili.com/: 26642
https://www.booking.com/: 457842
https://www.douyu.com/: 75286
https://www.qunar.com/: 140898
https://www.suning.com/: 188523
https://www.taobao.com/: 126283
24.28531312942505
```

可以看出打印的顺序是和 url 列表的顺序完全一致的，同步的代码耗时约 24s。

### 基于线程的请求

首先导入所需的库：

```python
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests
from url import urls
```

完成单个请求的函数：

```python
def fetch(session, url):
    resp = session.get(url)
    return resp.content
```

使用线程池请求所有的 url：

```python
def main():
    session = requests.Session()
    with ThreadPoolExecutor() as executor:
        tasks = {executor.submit(fetch, session, url): url for url in urls}
        for task in as_completed(tasks.keys()):
            data = task.result()
            print(f'{tasks[task]}: {len(data)}')
```

记录完成所需的时间：

```python
if __name__ == '__main__':
    start = time.time()
    main()
    print(time.time() - start)
```

上述代码的结果：

```shell
http://www.ccb.com/: 276
http://www.12306.cn/: 1480
http://www.cnki.net/: 59235
http://mail.aliyun.com/: 725
http://www.tianya.cn/: 7867
http://www.icbc.com.cn/icbc/: 157227
http://www.bankcomm.com/: 3473
http://www.chsi.com.cn/: 34188
http://mail.sina.com.cn/: 2837
...
http://www.sina.com.cn/: 584540
https://tieba.baidu.com/: 137714
https://www.autohome.com.cn/: 656154
https://www.taobao.com/: 126283
https://www.booking.com/: 457849
http://www.iqiyi.com/: 599940
http://tuijian.hao123.com/: 511465
9.722297191619873
```

可以看到返回结果的顺序并不和 url 列表一致，准确的说，是按照请求完成的顺序排列的。同时，请求所需的时间大幅缩短，降到了约 9s。

### 基于 `asyncio` 的请求

首先导入所需的库：

```python
import asyncio
import time
import aiohttp
from url import urls
```

完成单个请求的函数：

```python
async def fetch(session, url):
    async with session.get(url) as resp:
        return url, await resp.read()
```

这里同时返回了请求的 url 和网站内容，是因为后面的代码不容易在请求完成后获得请求的 url。

使用 `aiohttp` 请求所有的 url：

```python
async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        for task in asyncio.as_completed(tasks):
            url, data = await task
            print(f'{url}: {len(data)}')
```

开启事件循环，并记录所需的时间：

```python
if __name__ == '__main__':
    start = time.time()
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
    print(time.time() - start)
```

上述代码的结果：

```shell
http://www.psbc.com/: 404
http://www.eastmoney.com/: 392883
http://www.icbc.com.cn/icbc/: 157227
http://www.ifeng.com: 438464
http://mail.aliyun.com/: 725
http://www.tianya.cn/: 7867
...
http://tuijian.hao123.com/: 510052
http://mail.qq.com/: 8023
http://game.hao123.com/: 179707
https://tieba.baidu.com/: 137723
http://www.iqiyi.com/: 599830
https://www.taobao.com/: 126283
https://weibo.com/: 6117
http://www.zhihu.com/: 22696
https://www.booking.com/: 457851
2.0516560077667236
```

和使用线程一样，返回结果是按照请求完成顺序排列的。请求的时间比线程更短，只用了约 2s 就完成了所有的请求。和使用线程的方式相比，`asyncio` 避免了创建线程的开销。

#### 保存请求的结果

需要注意的是，上述请求只是简单的获取了内容，这些 bytes 只在内存中存在。一旦我们需要把结果保存到磁盘，就会有另一个会导致异步代码退化到同步的地方：磁盘 I / O。

现在我们增加一个保存请求内容到磁盘的函数：

```python
from urllib.parse import quote_plus

def save_to_file(filename, data):
    with open(f'async_data/{quote_plus(filename)}.html', 'wb') as f:
        f.write(data)
```

同时增加一个函数，用来同时发起请求并把结果保存到文件：

```python
async def fetch_and_save(session, url):
    url, data = await fetch(session, url)
    save_to_file(url, data * 500)  # 把文件大小扩大 500 倍，使结果更明显
    return url
```

同时更新一下 `main()` 函数：

```python
async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_and_save(session, url) for url in urls]
        for task in asyncio.as_completed(tasks):
            url = await task
            print(f'save: {url}')
```

上述代码的结果：

```shell
save: http://www.psbc.com/
save: http://www.cnki.net/
save: http://www.eastmoney.com/
save: http://tuijian.hao123.com/
save: http://caipiao.hao123.com/
save: http://www.ifeng.com
save: http://www.fang.com/
save: http://www.qq.com/
...
save: https://www.bilibili.com/
save: https://www.autohome.com.cn/
save: https://www.booking.com/
save: https://mail.sohu.com/
save: https://weibo.com/
save: http://mail.qq.com/
save: http://mail.10086.cn/
save: http://www.zhihu.com/
10.63579511642456
```

可以看到消耗的时间增加到了约 10s。

有没有什么方法可以将同步的文件系统操作变为异步的呢？答案就是结合使用线程和 `asyncio`。修改一下 `fetch_and_save()` 函数，使其在其他线程中执行保存操作：

```python
async def fetch_and_save(session, url):
    url, data = await fetch(session, url)
    loop = asyncio.get_event_loop()
    loop.run_in_executor(None, save_to_file, url, data * 500)  # 默认使用 ThreadPoolExecutor
    return url
```

修改后的结果：

```shell
save: http://www.psbc.com/
save: http://www.12306.cn/
save: http://www.ccb.com/
save: http://www.cnki.net/
save: http://www.sohu.com/
save: http://www.37.com/
save: http://www.icbc.com.cn/icbc/
save: http://mail.sina.com.cn/
save: http://www.ifeng.com
...
save: https://mail.qq.com/
save: http://mail.163.com/
save: https://mail.sohu.com/
save: https://weibo.com/
save: http://mail.qq.com/
save: http://mail.10086.cn/
save: http://www.zhihu.com/
save: https://www.booking.com/
4.817075967788696
```

效果很明显，所需的时间缩短到了约 5s。

**NOTE：**需要注意的是，大多数操作系统上并未提供文件系统的异步 I / O 操作（Linux kernel 提供了文件系统异步 I / O，不过它需要一个额外的库 [aio](http://lse.sourceforge.net/io/aio.html)），大部分的异步框架都是使用线程处理文件系统 I / O 的。如果需要统一的 API，可以选择 [aiofiles](https://github.com/Tinche/aiofiles/)。
