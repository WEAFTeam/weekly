---
title: 量化算法的一个总结
date: 2019-03-17 17:06:17
tags: Quantization
category: Quantization
mathjax: true
thumbnail: 'https://s2.ax1x.com/2019/03/17/AeYUk4.png'
author: Milittle
---

> https://nervanasystems.github.io/distiller/algo_quantization/index.html

### 量化算法

#### 基于范围线性量化

分解以上专业术语：

1. 线性：Means a float value is quantized by multiplying with a numeric constant (the **scale factor**).（意思是一个浮点数通过和一个算术常数相乘得到量化值，这个算术常数也被叫做scale factor。）
2. 基于范围：Means that in order to calculate the scale factor, we look at the actual range of the tensor's values. In the most naive implementation, we use the actual min/max values of the tensor. Alternatively, we use some derivation based on the tensor's range / distribution to come up with a narrower min/max range, in order to remove possible outliers. This is in contrast to the other methods described here, which we could call **clipping-based**, as they impose an explicit clipping function on the tensors (using either a hard-coded value or a learned value).（）

##### 非对称和对称

#### Asymmetric Mode： 非对称 如下图所示

![](https://s2.ax1x.com/2019/03/17/AeJLS1.png)

In **asymmetric** mode, we map the min/max in the float range to the min/max of the integer range. This is done by using a **zero-point** (also called *quantization bias*, or *offset*) in addition to the scale factor.

Let us denote the original floating-point tensor by $x_f$, the quantized tensor by $x_q$, the scale factor by $q_x$, the zero-point by $zp_x$ and the number of bits used for quantization by $n$. Then, we get:
$$
x_q = round\left ((x_f - min_{x_f})\underbrace{\frac{2^n - 1}{max_{x_f} - min_{x_f}}}_{q_x} \right) = round(q_x x_f - \underbrace{min_{x_f}q_x)}_{zp_x} = round(q_x x_f - zp_x)
$$
In practice, we actually use $zp_x = round(min_{x_f}q_x)$. This means that zero is exactly representable by an integer in the quantized range. This is important, for example, for layers that have zero-padding. By rounding the zero-point, we effectively "nudge" the min/max values in the float range a little bit, in order to gain this exact quantization of zero.

Note that in the derivation above we use unsigned integer to represent the quantized range. That is, $x_q \in [0, 2^n-1]$. One could use signed integer if necessary (perhaps due to HW considerations). This can be achieved by subtracting $2^{n-1}$.

Let's see how a **convolution** or **fully-connected (FC)** layer is quantized in asymmetric mode: (we denote input, output, weights and bias with $x, y, w$ and $b$ respectively)
$$
y_f = \sum{x_f w_f} + b_f = \sum{\frac{x_q + zp_x}{q_x} \frac{w_q + zp_w}{q_w}} + \frac{b_q + zp_b}{q_b} =\\
= \frac{1}{q_x q_w} \left( \sum { (x_q + zp_x) (w_q + zp_w) + \frac{q_x q_w}{q_b}(b_q + zp_b) } \right)\\
Therefore:
y_q = round(q_y y_f) = round\left(\frac{q_y}{q_x q_w} \left( \sum { (x_q+zp_x) (w_q+zp_w) + \frac{q_x q_w}{q_b}(b_q+zp_b) } \right) \right)
$$
Notes:

- We can see that the bias has to be re-scaled to match the scale of the summation.
- In a proper integer-only HW pipeline, we would like our main accumulation term to simply be $sum{x_q w_q}$. In order to achieve this, one needs to further develop the expression we derived above. For further details please refer to the [gemmlowp documentation](https://github.com/google/gemmlowp/blob/master/doc/quantization.md#implementation-of-quantized-matrix-multiplication)

#### Symmetric Mode 对称模式 如下图所示

![](https://s2.ax1x.com/2019/03/17/AeJjOK.png)

In **symmetric** mode, instead of mapping the exact min/max of the float range to the quantized range, we choose the maximum absolute value between min/max. In addition, we don't use a zero-point. So, the floating-point range we're effectively quantizing is symmetric with respect to zero, and so is the quantized range.

Using the same notations as above, we get:
$$
x_q = round\left (x_f \underbrace{\frac{2^{n-1} - 1}{\max|x_f|}}_{q_x} \right) = round(q_x x_f)
$$

#### Comparing the Two Modes 比较两个模式

The main trade-off between these two modes is simplicity vs. utilization of the quantized range.

- When using asymmetric quantization, the quantized range is fully utilized. That is because we exactly map the min/max values from the float range to the min/max of the quantized range. Using symmetric mode, if the float range is biased towards one side, could result in a quantized range where significant dynamic range is dedicated to values that we'll never see. The most extreme example of this is after ReLU, where the entire tensor is positive. Quantizing it in symmetric mode means we're effectively losing 1 bit.
- On the other hand, if we look at the derviations for convolution / FC layers above, we can see that the actual implementation of symmetric mode is much simpler. In asymmetric mode, the zero-points require additional logic in HW. The cost of this extra logic in terms of latency and/or power and/or area will of course depend on the exact implementation.

### Other Features [Neural Network Distiller](https://nervanasystems.github.io/distiller/index.html) 其他的特点

- **Removing Outliers:** As discussed [here](https://nervanasystems.github.io/distiller/quantization/index.html#outliers-removal), in some cases the float range of activations contains outliers. Spending dynamic range on these outliers hurts our ability to represent the values we actually care about accurately.

![](https://s2.ax1x.com/2019/03/17/AeJOQx.png)

Currently, Distiller supports clipping of activations with averaging during post-training quantization. That is - for each batch, instead of calculating global min/max values, an average of the min/max values of each sample in the batch.

- **Scale factor scope:** For weight tensors, Distiller supports per-channel quantization (per output channel).

QQ: 329804334

Website:  www.weaf.top

Mail: air@weaf.top

Github: https://github.com/Milittle

