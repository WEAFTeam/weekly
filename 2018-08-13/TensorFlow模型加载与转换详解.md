---
title: TensorFlow模型加载与转换详解
date: 2018-08-28 09:27:11
tags: TensorFlow
category: TensorFlow
mathjax: true
author: Milittle
thumbnail: 'https://s1.ax1x.com/2018/03/18/9oakkQ.png'
---

# TensorFlow模型加载与转换详解

##### 本次讲解主要涉及到TensorFlow框架训练时候模型文件的管理以及转换。

1. 首先我们需要明确TensorFlow模型文件的存储格式以及文件个数：

```
model_folder:
------checkpoint
------model.meta
------model.data-00000-of-00001
------model.index
以上是模型文件夹里面存在的所有文件：
checkpoint文件是存储所有模型文件的名字，在使用tf.train.latest_checkpoint()的时候，该函数会借助此文件内容获取最新模型文件。
model.meta文件是图的基本架构，pb格式文件，里面包含变量，操作，集合等数据。
model.data-00000-of-00001文件和model.index文件就是ckpt文件，里面的内容存储的就是权重、偏置等内容。在TensorFlow0.11之前，使用ckpt一个后缀文件存储，以后的TensorFlow版本都是使用这两个文件共同存储模型参数。
```

明确了这一点以后，我们就开始创建计算图，也就是网络结构已经内部的运算。

2. 网络的搭建，为了简单起见，我们搭建的网络就比较简单：input layer,conv_1,conv_2,fc1,dropout,fc2(输出层),具体搭建代码如下：

```python
def network():

    # define the placeholder by using feed the data
    with tf.name_scope('input_placeholder'):
        x = tf.placeholder(tf.float32, [None, 784], 'x')  # 28*28=784 dim
        x_input = tf.reshape(x, [-1, 28, 28, 1], 'x_reshape')  # reshape for conv, -1表示不固定数量，1为通道数
        y_label = tf.placeholder(tf.float32, [None, FLAGS.classes], 'y_label')  # label - 10 dim

    # define convolution layer1
    with tf.name_scope('conv_layer1'):
        W_conv1 = weight_variable([5, 5, 1, 32], name='w_conv_1')  # Weight in:1  out:32
        b_conv1 = bias_variable([32], name='b_conv_1')  # bias
        h_relu1 = tf.nn.relu(conv2d(x_input, W_conv1) + b_conv1, name='relu_1')  # relu
        h_pool1 = max_pool_2(h_relu1, name='pool_1')  # pool after relu1

    # define convolution layer2
    with tf.name_scope('conv_layer2'):
        W_conv2 = weight_variable([5, 5, 32, 64], name='w_conv_2')  # Weight in:32  out:64
        b_conv2 = bias_variable([64], name='b_conv_2')  # bias for 64 kernel
        h_relu2 = tf.nn.relu(conv2d(h_pool1, W_conv2) + b_conv2, name='relu_2')  # relu
        h_pool2 = max_pool_2(h_relu2, name='pool_2')  # pool after relu2

    # define the first FC layer
    with tf.name_scope('fc1'):
        W_fc1 = weight_variable([7 * 7 * 64, 1024], name='w_fc1')  # Weight in:7*7res*64  out:1024
        b_fc1 = bias_variable([1024], name='b_fc1')  # bias for 1024
        h_pool2_flat = tf.reshape(h_pool2, [-1, 7 * 7 * 64], name='pool1')
        h_fc1 = tf.nn.relu(tf.matmul(h_pool2_flat, W_fc1) + b_fc1, name='relu1')

    # adding the dropout, in order to restrain overfitting
    with tf.name_scope('drop_out'):
        keep_prob = tf.placeholder(tf.float32, name='drop_out_placeholder')
        drop_fc1 = tf.nn.dropout(h_fc1, keep_prob, name='drop_out_fc')

    # define the second FC layer, by using softmax
    with tf.name_scope('fc2'):
        W_fc2 = weight_variable([1024, FLAGS.classes], name='w_fc2')  # Weight in:1024  out:10
        b_fc2 = bias_variable([FLAGS.classes], name='b_fc2')  # bias for 10, 10类划分
        y = tf.nn.softmax(tf.matmul(drop_fc1, W_fc2) + b_fc2, name='y_out')  # 计算结果

    global_step = tf.Variable(0, trainable=False)

    # define the loss
    with tf.name_scope('loss'):
        cross_entropy = tf.reduce_mean(-tf.reduce_sum(y_label * tf.log(y), reduction_indices=[1]), name='cross_entropy')
    with tf.name_scope('train_op'):
        train_step = tf.train.AdamOptimizer(FLAGS.lr).minimize(cross_entropy,
                                                               global_step=global_step,
                                                               name='train_operation')  # Adam 替代SGD

    # define the accuracy
    with tf.name_scope('accuracy'):
        correct_pred = tf.equal(tf.argmax(y, 1), tf.argmax(y_label, 1), name='condition')
        accuracy = tf.reduce_mean(tf.cast(correct_pred, tf.float32), name='accuracy')

    return x, y, keep_prob, y_label, train_step, accuracy, global_step
```

以上需要注意的地方是，我在每一层中都加入了name_scope，这样的好处就是可以更清楚的分清层与层之间的关系，以及对于后续我们直接通过tensor name来获取变量，而无须创建计算图架构做准备。

3. 数据加载以及开始训练

```python
# 数据加载
mnist = input_data.read_data_sets("MNIST_data/", one_hot=True)
# 将数据全部加载在mnist中，供后需训练和测试使用
```



```python
# 模型训练，也可以看到模型保存的类
def train():

    # the sign which save the meta graph, just once.
    a = False
    x, y, keep_prob, y_label, train_step, accuracy, global_step = network()

    sess.run(tf.global_variables_initializer())
    saver = tf.train.Saver(max_to_keep=3)

    if FLAGS.use_model:
        model_t = tf.train.latest_checkpoint(FLAGS.model_path)
        saver.restore(sess, model_t)

    for i in range(FLAGS.max_iter_step):
        batch = mnist.train.next_batch(FLAGS.batch_size)  # 每50个一个batch
        if i % 100 == 0:
            # eval执行过程－训练精度
            train_accuracy = sess.run(accuracy, feed_dict={x: batch[0], y_label: batch[1], keep_prob: 1.0})
            print("step {step}, training accuracy {acc}".format(step=i, acc=train_accuracy))
            if (train_accuracy > 0.5):
                if a == 0:
                    saver.export_meta_graph(FLAGS.model_path + FLAGS.meta_graph_name)
                    a = True
                saver.save(sess, FLAGS.model_path + FLAGS.model_name, global_step=global_step, write_meta_graph=False)
        sess.run(train_step, feed_dict={x: batch[0], y_label: batch[1], keep_prob: FLAGS.keep_drop})
```

4. 那么怎么构建模型保存呢？

```python
# 首先，创建Saver对象
saver = tf.train.Saver(max_to_keep=3) # 这里设置的模型文件最大保存个数是三个，也就是说checkpoint文件中始终有三个版本的模型文件
# 第二步，那就是根据不同的迭代或者epoch，你可以随心所欲的保存模型。
# 我这里处理的逻辑就是每训练一百次，然后train_accuracy的大小大于0.5，那么我就开始存储。
# 这里需要注意一点，那就是在保存模型文件的时候，完全没有必要每次都保存meta，所以，可以单独在第一次保存meta，因为meta是graph，所以后续训练不会对meta起作用，所以减少开销。
saver.export_meta_graph(FLAGS.model_path + FLAGS.meta_graph_accuracy)
# 上面的代码只是在第一次保存模型的时候执行
saver.save(sess, FLAGS.model_path + FLAGS.model_name, global_step=global_step, write_meta_graph=False)
# 上面的代码每次都执行，但是不会保存meta数据，在一般的保存模型的时候，write_meta_graph标志位是True
```

5. 好，那么我们训练完成以后，模型文件已经有了，那么我们该如何导入刚才的模型文件执行测试呢？

```python
def test():

    if FLAGS.use_model:
        with tf.Session() as sess:
            saver = tf.train.import_meta_graph(FLAGS.model_path + FLAGS.meta_graph_name)
            saver.restore(sess, tf.train.latest_checkpoint(FLAGS.model_path))

            graph = tf.get_default_graph()


            # one operation possibly have many outputs, so you need specify the which output, such as "name:0"
            x = graph.get_tensor_by_name("input_placeholder/x:0")
            y_label = graph.get_tensor_by_name("input_placeholder/y_label:0")
            keep_prob = graph.get_tensor_by_name("drop_out/drop_out_placeholder:0")
            accuracy = graph.get_tensor_by_name("accuracy/accuracy:0")

            feed_dict = {x: mnist.test.images,
                         y_label: mnist.test.labels,
                         keep_prob: 1.0}

            acc = sess.run(accuracy, feed_dict=feed_dict)
            print("test accuracy {acc:.4f}".format(acc=acc))
```

上面的代码：

首先判断是否使用模型文件，

然后打开会话，在这里注意，我并没有创建网络结构，也就是说，在TensorFLow默认的图中是不存在我的计算图结构的。

然后我们使用tf.train.import_meta_graph()方法将模型图，也就是meta文件导入给saver

然后使用saver的restore方法将模型文件导入

然后使用tf.get_default_graph()方法获取TensorFlow默认的计算图（这回会获取到那个我们保存的计算图）

因为，我们没有定义网络结构中的变量，所以我们无法得到具体的网络执行变量，所以我们需要借助graph.get_tensor_by_name()方法来实现计算图变量的获取。

然后通过sess.run()方法来运行想要的tenosr值

（上面的代码中需要注意的是get_tensor_by_name的名字组成：（name_scope）/(tensor_name):第几个值）因为我们的运算是建立在tensor上的，但是每次运行的结果都是通过operation来实现的，也就是说，后面的那个index就是我们的第几个operation所要取的值。

6. 好，这里把模型文件的特殊保存和加载都讲完了，所以需要转换成pb文件，因为在后续我们使用TensorRT部署TensorFLow模型文件的时候，是需要pb文件，然后将pb文件转换为uff文件或者onnx文件来实现TenorRT网络的构建。

```python
def save_pb_file():

    if FLAGS.use_model:
        saver = tf.train.import_meta_graph(FLAGS.model_path + FLAGS.meta_graph_name)


        model_t = tf.train.latest_checkpoint(FLAGS.model_path)
        saver.restore(sess, model_t)

        graphdef = tf.get_default_graph().as_graph_def()

        frozen_graph = tf.graph_util.convert_variables_to_constants(sess, graphdef, ['fc2/y_out']) # 这个地方需要注意，是最后一个输出节点的tenosr名字

        return tf.graph_util.remove_training_nodes(frozen_graph)
    else:
        return False

graph_def = save_pb_file()

if graph_def is False:
    raise ValueError("The meta graph do not exist!!!")

    output_file = './graph.pb'
    with tf.gfile.GFile(name = output_file, mode = 'w') as f:
        s = graph_def.SerializeToString()
        f.write(s)
```

首先也是加载图，然后加载权重参数文件，然后将graph作为一个graphdef返回，然后通过tf.graph_util.convert_variables_to_constants将参数文件转换为常量，最后，使用tf.graph_util.remove_training_nodes(frozen_graph)将在训练阶段才使用的变量去除，也就是一些gradients。

返回去除训练阶段的节点，然后通过tf.gfile.GFile写入到指定文件。

7. 整体的代码：

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Time    : 2018/4/24 20:08
# @Author  : milittle
# @Site    : www.weaf.top
# @File    : model.py
# @Software: PyCharm
#coding=utf-8


import tensorflow as tf
from tensorflow.examples.tutorials.mnist import input_data
from tensorflow.python.framework import ops
import dataset

ops.reset_default_graph()
sess = tf.Session()

FLAGS = tf.app.flags.FLAGS
tf.app.flags.DEFINE_integer('max_iter_step', 1000, 'define iteration times')
tf.app.flags.DEFINE_integer('batch_size', 128, 'define batch size')
tf.app.flags.DEFINE_integer('classes', 10, 'define classes')
tf.app.flags.DEFINE_float('keep_drop', 0.5, 'define keep dropout')
tf.app.flags.DEFINE_float('lr', 0.001, 'define learning rate')
tf.app.flags.DEFINE_string('model_path', 'model\\','define model path')
tf.app.flags.DEFINE_string('model_name', 'model.ckpt', 'define model name')
tf.app.flags.DEFINE_string('meta_graph_name', 'model.meta', 'define model name')
tf.app.flags.DEFINE_bool('use_model', False, 'define use_model sign')
tf.app.flags.DEFINE_bool('is_train', True, 'define train sign')
tf.app.flags.DEFINE_bool('is_test', False, 'define train sign')


mnist = input_data.read_data_sets("MNIST_data/", one_hot=True)
# mnist_train = dataset.train("MNIST_data/")
# mnist_test = dataset.train("MNIST_data/")

# define W & b
def weight_variable(para, name):
    # 采用截断的正态分布，标准差stddev＝0.1
    initial = tf.truncated_normal(para,stddev=0.1)
    return tf.Variable(initial, name)

def bias_variable(para, name):
    initial = tf.constant(0.1, shape=para)
    return tf.Variable(initial, name)

# define conv & pooling
def conv2d(x,W):
    return tf.nn.conv2d( x,W,strides=[1,1,1,1],padding='SAME' )

def max_pool_2(x, name):
    return tf.nn.max_pool(x,ksize=[1,2,2,1],strides=[1,2,2,1],padding='SAME', name=name)

def network():

    # define the placeholder by using feed the data
    with tf.name_scope('input_placeholder'):
        x = tf.placeholder(tf.float32, [None, 784], 'x')  # 28*28=784 dim
        x_input = tf.reshape(x, [-1, 28, 28, 1], 'x_reshape')  # reshape for conv, -1表示不固定数量，1为通道数
        y_label = tf.placeholder(tf.float32, [None, FLAGS.classes], 'y_label')  # label - 10 dim

    # define convolution layer1
    with tf.name_scope('conv_layer1'):
        W_conv1 = weight_variable([5, 5, 1, 32], name='w_conv_1')  # Weight in:1  out:32
        b_conv1 = bias_variable([32], name='b_conv_1')  # bias
        h_relu1 = tf.nn.relu(conv2d(x_input, W_conv1) + b_conv1, name='relu_1')  # relu
        h_pool1 = max_pool_2(h_relu1, name='pool_1')  # pool after relu1

    # define convolution layer2
    with tf.name_scope('conv_layer2'):
        W_conv2 = weight_variable([5, 5, 32, 64], name='w_conv_2')  # Weight in:32  out:64
        b_conv2 = bias_variable([64], name='b_conv_2')  # bias for 64 kernel
        h_relu2 = tf.nn.relu(conv2d(h_pool1, W_conv2) + b_conv2, name='relu_2')  # relu
        h_pool2 = max_pool_2(h_relu2, name='pool_2')  # pool after relu2

    # define the first FC layer
    with tf.name_scope('fc1'):
        W_fc1 = weight_variable([7 * 7 * 64, 1024], name='w_fc1')  # Weight in:7*7res*64  out:1024
        b_fc1 = bias_variable([1024], name='b_fc1')  # bias for 1024
        h_pool2_flat = tf.reshape(h_pool2, [-1, 7 * 7 * 64], name='pool1')
        h_fc1 = tf.nn.relu(tf.matmul(h_pool2_flat, W_fc1) + b_fc1, name='relu1')

    # adding the dropout, in order to restrain overfitting
    with tf.name_scope('drop_out'):
        keep_prob = tf.placeholder(tf.float32, name='drop_out_placeholder')
        drop_fc1 = tf.nn.dropout(h_fc1, keep_prob, name='drop_out_fc')

    # define the second FC layer, by using softmax
    with tf.name_scope('fc2'):
        W_fc2 = weight_variable([1024, FLAGS.classes], name='w_fc2')  # Weight in:1024  out:10
        b_fc2 = bias_variable([FLAGS.classes], name='b_fc2')  # bias for 10, 10类划分
        y = tf.nn.softmax(tf.matmul(drop_fc1, W_fc2) + b_fc2, name='y_out')  # 计算结果

    global_step = tf.Variable(0, trainable=False)

    # define the loss
    with tf.name_scope('loss'):
        cross_entropy = tf.reduce_mean(-tf.reduce_sum(y_label * tf.log(y), reduction_indices=[1]), name='cross_entropy')
    with tf.name_scope('train_op'):
        train_step = tf.train.AdamOptimizer(FLAGS.lr).minimize(cross_entropy,
                                                               global_step=global_step,
                                                               name='train_operation')  # Adam 替代SGD

    # define the accuracy
    with tf.name_scope('accuracy'):
        correct_pred = tf.equal(tf.argmax(y, 1), tf.argmax(y_label, 1), name='condition')
        accuracy = tf.reduce_mean(tf.cast(correct_pred, tf.float32), name='accuracy')

    return x, y, keep_prob, y_label, train_step, accuracy, global_step

def train():

    # the sign which save the meta graph, just once.
    a = False
    x, y, keep_prob, y_label, train_step, accuracy, global_step = network()

    sess.run(tf.global_variables_initializer())
    saver = tf.train.Saver(max_to_keep=3)

    if FLAGS.use_model:
        model_t = tf.train.latest_checkpoint(FLAGS.model_path)
        saver.restore(sess, model_t)

    for i in range(FLAGS.max_iter_step):
        batch = mnist.train.next_batch(FLAGS.batch_size)  # 每50个一个batch
        if i % 100 == 0:
            # eval执行过程－训练精度
            train_accuracy = sess.run(accuracy, feed_dict={x: batch[0], y_label: batch[1], keep_prob: 1.0})
            print("step {step}, training accuracy {acc}".format(step=i, acc=train_accuracy))
            if (train_accuracy > 0.5):
                if a == 0:
                    saver.export_meta_graph(FLAGS.model_path + FLAGS.meta_graph_name)
                    a = True
                saver.save(sess, FLAGS.model_path + FLAGS.model_name, global_step=global_step, write_meta_graph=False)
        sess.run(train_step, feed_dict={x: batch[0], y_label: batch[1], keep_prob: FLAGS.keep_drop})

def test():

    if FLAGS.use_model:
        with tf.Session() as sess:
            saver = tf.train.import_meta_graph(FLAGS.model_path + FLAGS.meta_graph_name)
            saver.restore(sess, tf.train.latest_checkpoint(FLAGS.model_path))

            graph = tf.get_default_graph()


            # one operation possibly have many outputs, so you need specify the which output, such as "name:0"
            x = graph.get_tensor_by_name("input_placeholder/x:0")
            y_label = graph.get_tensor_by_name("input_placeholder/y_label:0")
            keep_prob = graph.get_tensor_by_name("drop_out/drop_out_placeholder:0")
            accuracy = graph.get_tensor_by_name("accuracy/accuracy:0")

            feed_dict = {x: mnist.test.images,
                         y_label: mnist.test.labels,
                         keep_prob: 1.0}

            acc = sess.run(accuracy, feed_dict=feed_dict)
            print("test accuracy {acc:.4f}".format(acc=acc))

def save_pb_file():

    if FLAGS.use_model:
        saver = tf.train.import_meta_graph(FLAGS.model_path + FLAGS.meta_graph_name)


        model_t = tf.train.latest_checkpoint(FLAGS.model_path)
        saver.restore(sess, model_t)

        graphdef = tf.get_default_graph().as_graph_def()

        frozen_graph = tf.graph_util.convert_variables_to_constants(sess, graphdef, ['fc2/y_out'])

        return tf.graph_util.remove_training_nodes(frozen_graph)
    else:
        return False

def main():
    if FLAGS.is_train:
        train()
    elif FLAGS.is_test:
        test()
    else:
        graph_def = save_pb_file()

        if graph_def is False:
            raise ValueError("The meta graph do not exist!!!")

        output_file = './graph.pb'
        with tf.gfile.GFile(name = output_file, mode = 'w') as f:
            s = graph_def.SerializeToString()
            f.write(s)

if __name__ == '__main__':
    try:
        main()
    except (ValueError, IndexError) as ve:
        print(ve)
```

今天的TensorFlow模型保存以及加载，以及将三个训练阶段使用的模型文件整合到一个pb文件中，这个pb文件不仅仅可以在构建TensorRT的网络中使用，也可以使用在部署TensorFlow serving中。整体的结构和存储过程就是这样。有什么问题，随时联系air@weaf.top。我一直都在。

如果你觉得我的内容对你有帮助，可以关注以下公众号，了解更多相关信息：

![](https://s1.ax1x.com/2018/08/28/PLcufe.md.jpg)

