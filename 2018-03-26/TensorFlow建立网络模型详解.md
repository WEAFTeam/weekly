---
title: TensorFlow建立网络模型详解
tags: TensorFlow
category: TensorFlow
author: milittle
mathjax: true
thumbnail: 'https://s1.ax1x.com/2018/03/18/9oakkQ.png'
abbrlink: 5b7df854
date: 2018-03-31 18:23:35
---

# TensorFlow 建立网络模型

上次一我们在fashion-mnist上面体验了一把，但是里面有一些建立模型和一些TensorFlow的基础概念都没有给大家讲，所以这节决定将这方面的知识介绍一些，上节是为了引起大家的注意，TensorFlow具有很强大的功能，我们只能后续慢慢的学习。

1. 其实在上一次的实例中，有很多地方确实是很困惑的，如果没有接触过机器学习的小伙伴可能理解起来会有一些问题，那么我开头就稍微讲一下，机器学习有一些什么？就我现在了解的一些内容给大家介绍，有可能有一些不到位的地方，还请多多包涵：

>* 其实机器学习，总的宗旨就是利用数据的特征来做识别和分类等任务
>* 第一大类是分类工作，假设有一百类，经典的做法，就是使用神经网络提取一些数据的特征，然后利用softmax输出层进行不同种类概率的预测：

$$
softmax(i) = \frac{X_i}{\sum_{i=0,99}X_i}
$$

>* 上面是softmax层计算的公式，从一百类里面找出每一类的概率值，然后按照概率值来预测输入数据是哪一种类型，就像上一次文章里面的fashion-mnist的数据一样，会预测出输出的类别。softmax(i)代表的就是这个种类的概率值，取最大值作为预测类别。
>* 你可以把一个矩阵看成一个数据集合，一行是一个数据信息，就和我们的关系型数据一样，一行代表一个表的一条信息，那么每列就是每一行数据的一个属性，那么在机器学习里面就是数据的特征了，因为在网络模型中，每个特征都有对应的权重，那么，对于每个特征来说，对于最后的分类，识别等工作起的重要程度是不一样的。这也和我们的数据库信息差不多，有一些信息也是无关紧要的。有些信息可以主要决定这一行数据。
>* 第二大类就是回归，回归可以看作是一个连续的分类，对于二维数据来说，其实就是根据你给出的数据来拟合一条线。对于三维来讲就是拟合一个平面。再高维就是超平面。

2. 最近，也就是2018年3月31在加利福尼亚州山景城的计算机历史博物馆举办了第二届TensorFlow开发者峰会，会上有超过500名使用TensorFlow的用户，还有一些观众，大家有兴趣的话可以关注youtube的TensorFlow官方频道。可以查看开会的视频。

* TensorFlow应用广泛，其中有使用TensorFlow来做开普勒任务分析的
* 也有使用TensorFlow预测心脏发作和中风概率
* 还有一些应用在现实当中的项目。
* 这让我们认识到TensorFlow对于实际领域中应用的越来越广泛，所以我们不学习是不是有点亏。这么好的开源项目。

3. 上一次我们既然做过了一次服装类别识别，那么这次我主要从TensorFlow建立模型的步骤讲起：让大家再深入理解一下TensorFlow。

* 第一步也是很重要的一步，那就是导入数据。
* 第二步一般就是对数据进行的预处理，一般包括归一化数据，转换数据等操作。
* 第三步设置算法的超参数，一般也就是学习率，batch_size(批处理个数)，epoch(轮次)。这里举一个例子，假如你有10000条训练数据。那么，batch_size设置为100，那么你的一个epoch就迭代100次才能将所有数据训练一遍，每次输入数据是100条，因为一个epoch的意思就是训练完一次训练数据，所以一个epoch是迭代100次就可以结束一轮了。learning_rate一般设置为0.1-0.0001之间，但是也不排除一些特殊情况，主要是learning_rate设置的过小，反向传播更新参数的时候速度会很慢，设置的过大，会出现无法收敛的情况。
* 第四步设置变量和placeholders，变量是记录权重和偏置项信息的，一般在最小化loss函数的时候，反向传播算法会更新权重和偏置项，TensorFlow导入数据是通过placeholders来实现的，大家还记得我们上次的fashion-mnist识别，我的数据就是通过先定义placeholders，最后在Session运行的时候，在feed_dict这个字典参数里面将训练数据喂进去的。

```python
a_var = tf.constant(42)
x_input = tf.placeholder(tf.float32, [n_x, None], name="X")
x_output = tf.placeholder(tf.float32, [n_y, None], name="y")
# 定义输入数据的一些方式
```

* 第五步定义图模型，我们有了数据，初始化了变量和placeholders，那我们就需要定义一个图模型，来生成TensorFlow的图模型（计算图）我们必须告诉TensorFlow对我们的数据进行哪些操作，来让我们的模型具有预测能力（更加深入的运算我们在后续的博客里面会陆续讲到）

```python
h_pre_output = tf.add(tf.matmul(W, x_input) + B)
```

* 第六步声明loss函数，在上面计算图中我们定义了一些对我们数据的操作。那么我们需要验证我们预测的输出，和我们真实之间的差距，一般对于回归任务来讲的话，就是平方误差：这样就求得了平方误差。但是对于分类任务，那就是交叉熵误差。就像上一节我们用到的loss生成函数就是softmax这种方式。，交叉熵的公式后续用到再给大家介绍。

$$
loss(i)=\frac{1}{N}\sum{_i}(y\_pre_i-y\_true_i)^2
$$

```python
TensorFlow求法：
loss = tf.reduce_mean(tf.square(y_pre - y_true))
```

* 第七步声明了loss函数以后，我们需要使用BP算法也就是反向传播算法来更新权重和偏置项。在TenorFlow框架里面有好多这样的优化器，都在 tf.train这个模块里面。

```python
optimizer = tf.train.AdamOptimizer(learning_rate = 0.001).minimize(loss)
这个就是我们上次使用的优化器，来优化我们的loss
```

* 最后一步那就是初始化会话Session()，开始训练模型

```python
with tf.Session() as session:
	session.run(init)
	.....
```

4. 由上面的步骤，大家再结合上一次的网络代码，是不是可以理解了TensorFlow在建立一个网络模型的时候的具体步骤。
5. 其实在TensorFlow中还有一个很重要的概念，那就是Tensor，上次说过了它的概念，那么接下来我讲一下TensorFlow里面的Tensor。

```python
import tensorflow as tf
from tensorflow.python.framework import ops
ops.reset_default_graph()
# 定义一个会话，记得，TensorFlow里面都是通过session来执行的
sess = tf.Session()
# 创建一个1 * 20的向量
tensor_zeros = tf.zeros([1, 20])
sess.run(tensor_zeros) # 你可以运行一下看看
my_var = tf.Variable(tf.zeros([1, 20])) # 使用tenso来初始化变量
sess.run(my_var.initializer) # 又一种运行变量初始化器的方式
sess.run(my_var) #打印出来看看

# tf.ones() 生成全是1
# tf.zeros() 生成全是0
# tf.constant() 生成一个常量Tensor
# 如果我们想要通过一个已知的Tensor来创建另一个，则可以使用ones_like()和zeros_like()这两个函数
zero_similar = tf.Variable(tf.zeros_like(tensor_zeros))

sess.run(zero_similar.initializer)
print(sess.run(zero_similar))
# 注意上面的两个函数的参数是为了确定生成Tensor的大小，而产生的值是通过函数决定的
tf.fill([row, col], -1)  # 用具体的数字填充
tf.linspace(start=0.0, stop=1.0, num=3) # 线性分布 包括end
tf.range(start=6, limit=15, delta=3)    # 也是线性均匀 不包括end
tf.random_normal([row_dim, col_dim], mean=0.0, stddev=1.0) # 随机 均值0 方差1.0
tf.random_uniform([row_dim, col_dim], minval=0, maxval=4) # 或者最小最大值随机初始化
```

```python
import tensorflow as tf
from tensorflow.python.framework import ops
ops.reset_default_graph()

sess = tf.Session()

my_var = tf.Variable(tf.zeros([1,20]))

merged = tf.summary.merge_all()

writer = tf.summary.FileWriter("./tmp/variable_logs", graph=sess.graph)

initialize_op = tf.global_variables_initializer()

sess.run(initialize_op)
# 上面的就是一个Tensor放在一个变量里面，我们使用了一条语句 merged = tf.summary.merge_all() 还有writer = tf.summary.FileWriter("/tmp/variable_logs", graph=sess.graph)，这两句这是为了将变量在TensorBoard里面显示出来，让我们更加了解TensorFLow的一些操作。
# 上面的操作过程会在当前文件夹里面创建一个/tmp/variable_logs文件夹然后会将变量信息存储在一个文件里面
```

6. 那怎么使用tensorboard

```
#进去我们的环境变量，然后执行
tensorboard --logdir=tmp的绝对路径
```

![-](https://s1.ax1x.com/2018/04/01/9zF2P1.png)

可以看到我上面执行的命令。然后在浏览器里面输入127.0.0.1:6006然后你就可以看到刚才那个变量的操作过程，这就是tensorboard的魅力

![-](https://s1.ax1x.com/2018/04/01/9zF7ad.png)

上面就是一个变量在进行初始化时候可视化显示

7. Placeholders使用(一样可以使用tensorboard来查看)

```
import numpy as np
import tensorflow as tf
from tensorflow.python.framework import ops
ops.reset_default_graph()
sess = tf.Session()
# 定义一个placeholder
x = tf.placeholder(tf.float32, shape = (4, 4))

# 随机生成4 * 4的矩阵
reand_array = np.random.rand(4, 4)
y = tf.identity(x) # 返回与输入对象相同的内容和大小
print(sess.run(y, feed_dict={x: rand_array}))

merged = tf.summary.merge_all()
writer = tf.summary.FileWriter("./tmp/variable_logs", sess.graph)
```

![-](https://s1.ax1x.com/2018/04/01/9zAshR.png)

##### 总结

这次我们就TensorFlow的一些基础概念的介绍，也是为了让大家在以后的TensorFlow使用过程中少一些疑问，后面的章节，我们会慢慢深入。小伙伴们不要着急，我的邮箱是air@weaf.top，依旧是那个可以交流学习的milittle。谢谢大家的驻足。

[第一篇 TensorFlow安装](https://weaf.top/posts/8e8e4531/)

[第二篇 TensorFlow初体验（fasion-mnist识别）](https://weaf.top/posts/b0821049/)

[修改pip全局镜像方法](https://weaf.top/posts/233074e6/)