---
title: TensorRT-量化指北
tags: Quantization
category: Quantization
mathjax: true
thumbnail: 'https://s2.ax1x.com/2019/03/17/AeYUk4.png'
author: Milittle
abbrlink: c316071b
date: 2019-03-16 15:51:02
---

### TensorRT量化指北

1. 对称的线性量化：

$$
TensorValues = FP32\,scale\,factor\,*int8\,array
$$

One FP32 scale factor for the entire int8 tensor

Q: 怎么设置scale factor？

非饱和方式：映射|max|到127 下图所示

![Quantization](https://s2.ax1x.com/2019/03/17/AeJsoQ.jpg)

- 一般上面的方式映射就会出现精确率严重降低。
- 那么我们需要找一种饱和的方式去映射，如下图右边所示那样，你可以左右对比一下。

![Quantization](https://s2.ax1x.com/2019/03/17/AeJ2zq.jpg)

- 以上右边映射的方式，对于权重映射的方式精确率影响不大，但是对于激活值影响很大。（那么就会面临一个问题，就是这样一个饱和的T该怎么选择）

2. 怎么选择一个合适的threshold呢？

![Quantization](https://s2.ax1x.com/2019/03/17/AeJgWn.md.jpg)

- 我们通过转换要将int8数据的转换信息丢失最少，那么怎么度量信息的loss，答案是使用KL散度，也就是相对熵，也叫Kullback-Leibler divergence。下图是一个描述，详细请看信息熵，交叉熵和相对熵的关系以及计算方式，我们这里处理的是离散的问题。

![Quantization](https://s2.ax1x.com/2019/03/17/AeJcJs.md.jpg)

- 下面展示一张在标定过程中，找到threshold的过程图：

![](https://s2.ax1x.com/2019/03/17/AeJ6ij.md.jpg)

上图描述了一个标定，也就是使用FP32精度，通过标定数据集来找到threshold的过程：

对于每一层来说：

第一步：收集统计激活的直方图

第二步：对于不同的饱和threshold生成很多的量化分布

第三步：挑一个threshold以后最小化KL_divergence(ref_distr, quant_distr)

这样的过程只需要在一台桌面工作站几分钟就搞定了（完全是离线的方式）

下图展示了标定数据集的规模以及注意事项：

![](https://s2.ax1x.com/2019/03/17/AeJfyV.md.jpg)

需要有代表性，需要不同的样本，理想情况当然是验证集的一部分数据，大约需要1000左右的样本就够了。

3. 下面讲讲TensorRT的量化工作流程。

![](https://s2.ax1x.com/2019/03/17/AeJWQ0.md.jpg)

上图展示了TensorRT的量化模型过程：你需要一个FP32的预训练模型和标定数据集：

第一，你需要以FP32的方式运行标定数据集

第二，你需要收集和统计数据，包括每一层的激活值

第三，运行标定的算法，优化scale factor

第四，量化FP32权重到INT8

第五，生成标定表，然后使用INT8执行引擎执行推理。

原理就是这么简单，下面展示一些benchmark：

4. benchmark

![](https://s2.ax1x.com/2019/03/17/AeJhLT.md.jpg)

![](https://s2.ax1x.com/2019/03/17/AeJoo4.md.jpg)

5. 挑战和展望,结论，这里面说了 对于Relu激活的量化，还有一些RNN的问题，结论中讲到这种方法是对称的线性量化方式。基本上通过这种量化性能是不会降低的。

![](https://s2.ax1x.com/2019/03/17/AeJ5eU.md.jpg)

![](https://s2.ax1x.com/2019/03/17/AeJIwF.md.jpg)

7. 伪代码，伪代码一出来，我想大家就会更清楚。展示了相对熵的求法，以及最后确定threshold的方法。

![](https://s2.ax1x.com/2019/03/17/AeJHY9.md.jpg)

![](https://s2.ax1x.com/2019/03/17/AeJbWR.md.jpg)

![](https://s2.ax1x.com/2019/03/17/AeJXy6.md.jpg)

最后的最后，加油执行，一致前行。fighting。

QQ: 329804334

Website:  www.weaf.top

Mail: air@weaf.top

Github: https://github.com/Milittle