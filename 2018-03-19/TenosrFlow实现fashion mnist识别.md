---
title: TensorFlow实现fashion mnist识别
abbrlink: b0821049
date: 2018-03-25 21:18:37
tags: TensorFlow
mathjax: true
category: TensorFlow
author: milittle
thumbnail: https://s1.ax1x.com/2018/03/18/9oakkQ.png
---

# TensorFlow 初体验（Fashion-mnist）

1. 接着上一讲的内容，想必大家已经通过我的教程安装好了TensorFlow了吧，那我们这节课通过安装简单的跨平台的集成开发环境Spyder，在这个集成开发环境上面实现一些python程序。具体安装过程见如下阐述：

- 首先在应用程序里面找到Anaconda应用程序，打开里面的Anaconda Navigator，然后打开以后，选中我们上次建立好的环境tensorflow。

![-](https://s1.ax1x.com/2018/03/25/9qTXNV.png)

- 选中tensorflow这个环境变量以后，看到里面有一个集成开发环境叫spyder，这个工具就是今天我们要安装的，我的已经安装好了，所以是Launch，你们的没有安装好，所以是install状态，点解安装就好。（这个地方也可能需要翻墙）。
- 这个安装好以后，你就会在应用文件夹里面出现一个Spyder(tensorflow)这个应用程序，以后你就从应用文件夹启动就好。
- 那么启动以后：我也是启动了，出现了以下的情况：不慌，慢慢来。

![-](https://s1.ax1x.com/2018/03/25/9q7njH.png)

- 看到上面的错误，这个错误提示是因为没有安装jedi这个依赖库，而且要求版本要大于0.9.0。那我们接下来解决一下这个问题。

> 小插曲，一下就可以解决，具体操作步骤:
>
> 1. 还是打开上次那个AnacondaPrompt的命令行
> 2. 进去以后，执行`activate tensorflow` 相当于你要在这个环境下面给这个spyder安装这个依赖
> 3. 进去以后，执行`pip install jedi==0.9.0` 就可以了，然后重启spyder（可以直接在这个环境里面输入`spyder`命令就可以实现spyder的启动，你也可以在应用文件夹里面启动，性质是一样的）
> 4. 不出什么意外的话，spyder使用就没有问题了，有什么问题可以发邮件给我！！！

- 解决了上面的小插曲以后，我们在spyder中输入以下代码进行测试。

```python
import tensorflow as tf
sess = tf.Session()
init = tf.global_variables_initializer() 
# 此处的init是全局变量初始化器，
# TensorFlow的session必须执行这个初始化器才能执行前面建立好的图，
# 所以，这个是很重要的一点，后续也会强调
#（也就是后续再网络中建立变量就是通过那个初始化器来进行初始化工作的）
# 其实在没有变量的时候，这个初始化器是不需要的
# 但是为了让大家形成习惯，还是写上
sess.run(init)
hello = tf.constant('hello world')
print(sess.run(hello))
```

![-](https://s1.ax1x.com/2018/03/25/9q7r5V.png)

- 上图中左面是代码书写区域，右面上半部分是变量查看区域，还有文件夹区域可以切换，右面下半部分是执行console区域，我输入上面的代码，执行以后console区域打出hello world字符串。

2. 从上面的一些简单的测试以后，我们进入今天的主题，fashion-minist的识别，fashion-minist是一个服装识别的一个数据集，在这个数据集之前有一个mnist手写体识别数据集，这个手写数据集对应我们手写的十个数字，然后通过设计网络来识别手写体。但是今天我们不做手写体识别，直接来做fashion-minist识别。

- 闲话少说，上代码，边写边说。

首先目标是实现衣服种类的识别。

数据可以在 [Zalando_Fashion_MNIST_repository](https://github.com/zalandoresearch/fashion-mnist)这个Github仓库获取。

数据分为60000训练数据和10000测试数据，图片都是灰度图片，大小为28 X 28，总共也是由10类组成。

```python
# -*- coding: utf-8 -*-
"""
Created on Sun Mar 25 15:16:23 2018

@author: milittle
"""

# 导入一些必要的库
import numpy as np # 数学计算库
import matplotlib.pyplot as plt # 画图的一个库
import tensorflow as tf # TensorFlow的库

from tensorflow.examples.tutorials.mnist import input_data
fashion_mnist = input_data.read_data_sets('input/data', one_hot = True)



# 定义一个服装对应表
label_dict = {
    0: 'T-shirt/top',
    1: 'Trouser',
    2: 'Pullover',
    3: 'Dress',
    4: 'Coat',
    5: 'Sandal',
    6: 'Shirt',
    7: 'Sneaker',
    8: 'Bag',
    9: 'Ankle boot'
}

# 获取随机的数据和它的label
sample_1 = fashion_mnist.train.images[47].reshape(28,28)
sample_label_1 = np.where(fashion_mnist.train.labels[47] == 1)[0][0]

sample_2 = fashion_mnist.train.images[23].reshape(28,28)
sample_label_2 = np.where(fashion_mnist.train.labels[23] == 1)[0][0]

# 用matplot画出这个image和label
print("y = {label_index} ({label})".format(label_index=sample_label_1, label=label_dict[sample_label_1]))
plt.imshow(sample_1, cmap='Greys')
plt.show()

print("y = {label_index} ({label})".format(label_index=sample_label_2, label=label_dict[sample_label_2]))
plt.imshow(sample_2, cmap='Greys')
plt.show()

# 接下来就是设计网络参数
n_hidden_1 = 128 # 第一个隐藏层的单元个数
n_hidden_2 = 128 # 第二个隐藏层的单元个数
n_input = 784 # fashion mnist输入图片的维度（单元个数） (图片大小: 28*28)
n_classes = 10 # fashion mnist的种类数目 (0-9 数字)


# 创建 placeholders
def create_placeholders(n_x, n_y):
    """
    为sess创建一个占位对象。
    
    参数:
    n_x -- 向量, 图片大小 (28*28 = 784)
    n_y -- 向量, 种类数目 (从 0 到 9, 所以是 -> 10种)
    
    返回参数:
    X -- 为输入图片大小的placeholder shape是[784, None] 
    Y -- 为输出种类大小的placeholder shape是[10, None] None在这里表示以后输入的数据可以任意多少
    """
    
    X = tf.placeholder(tf.float32, [n_x, None], name="X")
    Y = tf.placeholder(tf.float32, [n_y, None], name="Y")
    
    return X, Y


# 测试上面的create_placeholders()
X, Y = create_placeholders(n_input, n_classes)
print("Shape of X: {shape}".format(shape=X.shape))
print("Shape of Y: {shape}".format(shape=Y.shape))

# 定义初始化参数参数
def initialize_parameters():
    """
    参数初始化，下面是每个参数的shape，总共有三层
                        W1 : [n_hidden_1, n_input]
                        b1 : [n_hidden_1, 1]
                        W2 : [n_hidden_2, n_hidden_1]
                        b2 : [n_hidden_2, 1]
                        W3 : [n_classes, n_hidden_2]
                        b3 : [n_classes, 1]
    
    返回:
    包含所有权重和偏置项的dic
    """
    
    # 设置随机数种子
    tf.set_random_seed(42)
    
    # 为每一层的权重和偏置项进行初始化工作
    W1 = tf.get_variable("W1", [n_hidden_1, n_input], initializer = tf.contrib.layers.xavier_initializer(seed = 42))
    b1 = tf.get_variable("b1", [n_hidden_1, 1], initializer = tf.zeros_initializer())
    
    W2 = tf.get_variable("W2", [n_hidden_2, n_hidden_1], initializer = tf.contrib.layers.xavier_initializer(seed = 42))
    b2 = tf.get_variable("b2", [n_hidden_2, 1], initializer = tf.zeros_initializer())
    
    W3 = tf.get_variable("W3", [n_classes, n_hidden_2], initializer=tf.contrib.layers.xavier_initializer(seed = 42))
    b3 = tf.get_variable("b3", [n_classes, 1], initializer = tf.zeros_initializer())
    
    # 将参数存储在一个dict对象里面返回去
    parameters = {
        "W1": W1,
        "b1": b1,
        "W2": W2,
        "b2": b2,
        "W3": W3,
        "b3": b3
    }
    
    return parameters

# 测试初始化参数
tf.reset_default_graph()
with tf.Session() as sess:
    parameters = initialize_parameters()
    print("W1 = {w1}".format(w1=parameters["W1"]))
    print("b1 = {b1}".format(b1=parameters["b1"]))
    print("W2 = {w2}".format(w2=parameters["W2"]))
    print("b2 = {b2}".format(b2=parameters["b2"]))
    

# 前向传播算法（就是神经网络的前向步骤）
def forward_propagation(X, parameters):
    """
    实现前向传播的模型 LINEAR -> RELU -> LINEAR -> RELU -> LINEAR -> SOFTMAX
    上面的显示就是三个线性层，每一层结束以后，实现relu的作用，实现非线性功能，最后三层以后用softmax实现分类
    
    参数:
    X -- 输入训练数据的个数[784, n] 这里的n代表可以一次训练多个数据
    parameters -- 包括上面所有的定义参数三个网络中的权重W和偏置项B

    返回:
    Z3 -- 最后的一个线性单元输出
    """
    
    # 从参数dict里面取到所有的参数
    W1 = parameters['W1']
    b1 = parameters['b1']
    W2 = parameters['W2']
    b2 = parameters['b2']
    W3 = parameters['W3']
    b3 = parameters['b3']
    
    # 前向传播过程
    Z1 = tf.add(tf.matmul(W1,X), b1)     # Z1 = np.dot(W1, X) + b1
    A1 = tf.nn.relu(Z1)                  # A1 = relu(Z1)
    Z2 = tf.add(tf.matmul(W2,A1), b2)    # Z2 = np.dot(W2, a1) + b2
    A2 = tf.nn.relu(Z2)                  # A2 = relu(Z2)
    Z3 = tf.add(tf.matmul(W3,A2), b3)    # Z3 = np.dot(W3,Z2) + b3
    
    return Z3


# 测试前向传播喊出
tf.reset_default_graph()
with tf.Session() as sess:
    X, Y = create_placeholders(n_input, n_classes)
    parameters = initialize_parameters()
    Z3 = forward_propagation(X, parameters)
    print("Z3 = {final_Z}".format(final_Z=Z3))

# 定义计算损失函数
# 是计算loss的时候了
def compute_cost(Z3, Y):
    """
    计算cost
    
    参数:
    Z3 -- 前向传播的最终输出（[10, n]）n也是你输入的训练数据个数
    Y -- 
    
    返回:
    cost - 损失函数 张量（Tensor）
    """
    
    # 获得预测和准确的label
    logits = tf.transpose(Z3)
    labels = tf.transpose(Y)
    
    # 计算损失
    cost = tf.reduce_mean(tf.nn.softmax_cross_entropy_with_logits(logits = logits, labels = labels))
    
    return cost

# 测试计算损失函数
tf.reset_default_graph()
with tf.Session() as sess:
    X, Y = create_placeholders(n_input, n_classes)
    parameters = initialize_parameters()
    Z3 = forward_propagation(X, parameters)
    cost = compute_cost(Z3, Y)
    print("cost = {cost}".format(cost=cost))

# 这个就是关键了，因为每一层的参数都是通过反向传播来实现权重和偏置项参数更新的
# 总体的原理就是经过前向传播，计算到最后的层，利用softmax加交叉熵，算出网络的损失函数
# 然后对损失函数进行求偏导，利用反向传播算法实现每一层的权重和偏置项的更新
def model(train, test, learning_rate=0.0001, num_epochs=16, minibatch_size=32, print_cost=True, graph_filename='costs'):
    """
    实现了一个三层的网络结构: LINEAR->RELU->LINEAR->RELU->LINEAR->SOFTMAX.
    
    参数:
    train -- 训练集
    test -- 测试集
    learning_rate -- 优化权重时候所用到的学习率
    num_epochs -- 训练网络的轮次
    minibatch_size -- 每一次送进网络训练的数据个数（也就是其他函数里面那个n参数）
    print_cost -- 每一轮结束以后的损失函数
    
    返回:
    parameters -- 被用来学习的参数
    """
    
    # 确保参数不被覆盖重写
    tf.reset_default_graph()
    tf.set_random_seed(42)
    seed = 42
    # 获取输入和输出大小
    (n_x, m) = train.images.T.shape
    n_y = train.labels.T.shape[0]
    
    costs = []
    
    # 创建输入输出数据的占位符
    X, Y = create_placeholders(n_x, n_y)
    # 初始化参数
    parameters = initialize_parameters()
    
    # 进行前向传播
    Z3 = forward_propagation(X, parameters)
    # 计算损失函数
    cost = compute_cost(Z3, Y)
    # 使用AdamOptimizer优化器实现反向传播算法（最小化cost）
    # 其实我们这个地方的反向更新参数的过程都是tensorflow给做了
    optimizer = tf.train.AdamOptimizer(learning_rate).minimize(cost)
    
    # 变量初始化器
    init = tf.global_variables_initializer()
    
    # 开始tensorflow的sess 来计算tensorflow构建好的图
    with tf.Session() as sess:
        
        # 这个就是之前说过的要进行初始化的
        sess.run(init)
        
        # 训练轮次
        for epoch in range(num_epochs):
            
            epoch_cost = 0.
            num_minibatches = int(m / minibatch_size)
            seed = seed + 1
            
            for i in range(num_minibatches):
                
                # 获取下一个batch的训练数据和label数据
                minibatch_X, minibatch_Y = train.next_batch(minibatch_size)
                
                # 执行优化器
                _, minibatch_cost = sess.run([optimizer, cost], feed_dict={X: minibatch_X.T, Y: minibatch_Y.T})
                
                # 更新每一轮的损失
                epoch_cost += minibatch_cost / num_minibatches
                
            # 打印每一轮的损失
            if print_cost == True:
                print("Cost after epoch {epoch_num}: {cost}".format(epoch_num=epoch, cost=epoch_cost))
                costs.append(epoch_cost)
        
        # 使用matplot画出损失的变化曲线图
        plt.figure(figsize=(16,5))
        plt.plot(np.squeeze(costs), color='#2A688B')
        plt.xlim(0, num_epochs-1)
        plt.ylabel("cost")
        plt.xlabel("iterations")
        plt.title("learning rate = {rate}".format(rate=learning_rate))
        plt.savefig(graph_filename, dpi = 300)
        plt.show()
        
        # 保存参数
        parameters = sess.run(parameters)
        print("Parameters have been trained!")
        
        # 计算预测准率
        correct_prediction = tf.equal(tf.argmax(Z3), tf.argmax(Y))
        
        # 计算测试准率
        accuracy = tf.reduce_mean(tf.cast(correct_prediction, "float"))
        
        print ("Train Accuracy:", accuracy.eval({X: train.images.T, Y: train.labels.T}))
        print ("Test Accuracy:", accuracy.eval({X: test.images.T, Y: test.labels.T}))
        
        return parameters

# 要开始训练我们的fashion mnist网络了
train = fashion_mnist.train # 训练的数据
test = fashion_mnist.test # 测试的数据

parameters = model(train, test, learning_rate = 0.001, num_epochs = 16, graph_filename = 'fashion_mnist_costs')
```

- 上面的代码是写好了，这里有一个python的依赖库（matplotlib）需要安装以下，同样的办法，就是进去tensorflow这个环境里面，然后执行`pip install matplotlib`就可以了。
- 在这个过程中，可能从tensorflow下载数据的时候会很慢。（我们选择直接从上面给出下载数据集的github网址，直接下载以后，将数据拷贝在代码所在文件夹的input/data/文件夹里面，总共由四个文件组成）分别是训练数据图片、训练数据label和测试数据图片、测试数据label。这样就可以省去下载数据时候漫长的等待。

3. 上面就是我们使用TensorFlow实现的fashion-mnist的识别，总体根据实验结果来说，从测试集的数据来看，我达到的准确率结果是88.5%，还算可以。后续我们可能使用其他一些现有的网络结构来实现fashion-mnist的识别，看看准确率会不会提高。
4. 如下是我对上面TensorFLow出现的方法介绍：

```python
tf.placeholders(
	dtype,
	shape=None,
	name=None
)
从参数上面看到，总共有三个参数：
	dtype：在tensor中被喂数据的元素类型
	shape: tensor的shape
	name：命名
说明一下，这个函数返回的是一个tensor，在TensorFlow里面，tensor是一个很重要的概念，大家务必掌握，也叫张量，比如我们的一个数:就是0-阶张量，也叫标量。一个向量，就是1-阶张量。一个矩阵，就是2-阶张量，后面的就是一直往高维了走，对应的就是多少阶张量。
这个方法，很重要的原因也在于它是定义在Session执行run的时候，在后面填充数据的占位符，也就是feed_dict这个变量里面的数据，所以大家，务必记住这一关键的概念。后续用起来就会很顺手。
tf.get_variable()
这个方法后续在展开来说，你先理解就是使用它可以定义变量（保存权重和偏置项的），还可以加一些优化器，比如说正则优化器等等
tf.matmul(
	a,
	b,
)
展示给你们列出这两个参数：
	a：就是待操作的矩阵1
	b: 就是待操作的矩阵2
函数功能就是实现矩阵的相乘运算（当然要符合基本的矩阵运算格式）
tf.transpose(
	a,
)
先列出来一个参数，就是矩阵的转置
Session().run(
	fetches,
	feed_list=None,
	)
这个方法就是运行图。很关键，先掌握两个参数:
	fetches: 你要从图里面取出的数据（）
	feed_list: 你要给图喂的数据（输入和label数据就是用这样的方式来做的）
    比如我们训练的网络中输入的图片信息和对应的label信息
tf.reduce_mean(
	input_tensor,
	axis=None,
	keepdims=None,
	name=None,
	redcution_indices=None,
	keep_dims=None
	)
计算输入tensor的总和：
	input_tensor: 要叠加的tensor
	axis: 选择那个维度叠加
	keepdims: 叠加元素以后，保留原来的维度信息
	name：就是名字
	redcution_indices：被axis取代
	keep_dims：被keepdims取代
```

我们今天的任务量可能有一些大，大家坚持。总的来说就是使用神经网络对实际的一个fashion-mnist数据集进行服装种类的识别，大家主要看看我的代码。有什么不明白的我在代码里面都做出了注释。

邮箱------air@weaf.top欢迎来探讨