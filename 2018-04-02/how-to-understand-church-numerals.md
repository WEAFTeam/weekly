---
title: 如何理解丘奇计数
author: MisLink
thumbnail: 'http://ourm7pfm2.bkt.clouddn.com/18-4-9/67032850.jpg'
abbrlink: cd89a3b2
date: 2018-04-08 03:39:29
tags:
mathjax: true
---


## 前言

不想写 Python 了，这次换个主题：丘奇计数，又名 lambda 演算的自然数表示法。

## 什么是 lambda 演算

lambda 演算（也称为 λ 演算）是数学逻辑中的一种形式系统，它基于函数抽象和应用，使用变量绑定和替换来表示计算。

没错，上面这句话来自维基百科，基本上是一句正确的废话，看完了也不知道什么是 lambda 演算。不过这篇文章的重点不在 lambda 演算上，希望你已经了解了一些关于 lambda 演算的知识。如果有机会下一篇再展开说（可能

## 什么是自然数

在计算机科学和集合论中，我们把非负整数 $(0, 1, 2, 3, 4...)$ 称为自然数。皮亚诺给出了自然数的严格定义：

1. $0$ 是自然数；
2. 如果 $n$ 是自然数，那么 $n+1$ 也是自然数（$n+1$ 代表 $n$ 的后继）；
3. $0$ 不是任何一个数的后继；
4. 如果 $m$ 与 $n$ 都是自然数且 $m\neq n$，那么 $n+1 \neq m+1$；
5. 设 $P(n)$ 为关于自然数 $n$ 的一个性质，如果 $P(0)$ 正确， 且假设 $P(n)$ 正确，则 $P(n+1)$ 亦正确。那么 $P(n)$ 对一切自然数 $n$ 都正确。

存在一个集合 $N$，称其元素为自然数，当且仅当这些元素满足公理 1 - 5（也就是皮亚诺公理）。

在自然数集合上可以定义一组运算：加法、乘法等等，这里用加法举个例子：

```python
def add(m, n):
    if n == 0:
        return m
    return add(m, n - 1) + 1
```

可以看出加法是由两条规则递归定义的：

1. $ m + 0 = m$
2. $m + (n + 1) = (m + n) + 1$

## lambda 演算的自然数表示法

自然数当然不止可以用皮亚诺公理定义，丘奇首先把自然数和自然数上的运算定义在了 lambda 演算上，所以称之为丘奇计数。

下面就要开始丘奇计数的定义了，由于 lambda 演算的样子不太友好，所以还是用 Python 表示。

首先定义 0：

```python
zero = lambda f: lambda x: x
```

先不管它为什么是 0，让我们看看这个语句。它定义了一个接受一个参数 `f` 的函数，返回一个函数，这个函数接受一个参数 `x`，然后返回它。似乎很简单，但是它为什么是 0？

把它放在一边，看看 1 的定义：

```python
one = lambda f: lambda x: f(x)
```

这个语句是什么意思呢？定义了一个接受一个参数 `f` 的函数，返回一个函数，这个函数接受一个参数 `x`，然后返回 `f(x)` 的调用结果。好像有些规律了，再看看 2：

```python
two = lambda f: lambda x: f(f(x))
```

定义了一个接受一个参数 `f` 的函数，返回一个函数，这个函数接受一个参数 `x`，然后返回 `f(f(x))` 的调用结果。

现在可以清楚的看到，每个自然数的后继都多调用了一次 `f`，自然数被表示为 `f` 的调用次数。

于是，我们可以很轻易的写出后继函数：

```python
succ = lambda n: lambda f: lambda x: f(n(f)(x))
```

这个函数接受一个参数 `n`（也就是上面被定义的 0，1，2 等等），返回一个函数，这个函数在 `n` 的基础上多执行了一次 `f`，达到了求 `n` 的后继的目的。

现在让我们忘记 1 的定义，用 0 和后继重新定义一次：

```python
zero = lambda f: lambda x: x
succ = lambda n: lambda f: lambda x: f(n(f)(x))

one = succ(zero)
    = (lambda n: lambda f: lambda x: f(n(f)(x)))(lambda f: lambda x: x)
    = lambda f: lambda x: f(((lambda f: lambda x: x)(f))(x))
    = lambda f: lambda x: f((lambda x: x)(x))
    = lambda f: lambda x: f(x)
```

和我们预想的完全一致。

### 加法

接下来试着定义一下加法，加法是两个数相加返回一个数（也就是说，加法是定义在自然数上的幺半群），所以签名长这样：

```python
add = lambda m: lambda n: lambda f: lambda x: ...
```

函数体呢？我们推广一下后继函数：后继函数在`n`的基础上多调用了一次 `f`，相当于 `+1`；那加法相当于 `+m`，也就是多调用 `m` 次 `f`，于是可以得出：

```python
add = lambda m: lambda n: lambda f: lambda x: m(f)(n(f)(x))
```

### 乘法

乘法的签名也是一样：

```python
mul = lambda m: lambda n: lambda f: lambda x: ...
```

我们都知道乘法是从加法推广而来的：`m * n` 相当于加 `m` 次 `n`，所以可以使用加法的定义：

```python
mul = lambda m: lambda n: lambda f: lambda x: m(add(n))(zero)(f)(x)
```

上面的写法正确，不过太丑了，可以化简为：

```python
mul = lambda m: lambda n: lambda f: lambda x: m(n(f))(x)
```

### 求幂

求幂的签名也一样：

```python
pow = lambda m: lambda n: lambda f: lambda x: ...
```

求幂是由乘法推广而来的：m^n^   相当于乘 `n` 次 `m`，所以可以使用乘法的定义：

```python
pow = lambda m: lambda n: lambda f: lambda x: n(mul(m))(one)(f)(x)
```

同样可以化简为：

```python
pow = lambda m: lambda n: lambda f: lambda x: n(m)(f)(x)
```

## 结语

机智的同学一定发现我们并没有实现减法，这是因为减法的实现太复杂了。至于为什么减法的实现很复杂，以及如何实现减法，这里有一篇[参考资料](http://gettingsharper.de/2012/08/30/lambda-calculus-subtraction-is-hard/) ，有兴趣的话可以自行了解一下。
