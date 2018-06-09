---
title: chapter-11-AIR
tags: TensorFlow
category: TensorFlow
mathjax: true
thumbnail: 'https://s1.ax1x.com/2018/03/18/9oakkQ.png'
author: milittle
abbrlink: 45d29694
date: 2018-05-29 11:27:34
---

# TensorFlow 实现Siamese Network

1. 这次我们来实现一个基于lenet的Siamese Network，大家如果想了解Siamese Network，给大家推荐Andrew NG的课[deeplearning.ai](https://www.coursera.org/learn/convolutional-neural-networks/lecture/bjhmj/siamese-network)作为了解
2. 接下来我们直接上code：大家也可以直接移步到我的github 仓库里面找代码[代码](https://github.com/Milittle/MLModel/blob/milittle/siameasenetwork/mnist/train.py)

```
代码目录组织结构：
siameasenetwork
	__init__.py
	mnist：
		__init__.py
		train.py
		lenet.py
		util.py
		DataShuffler.py
```

train.py

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Time    : 2018/5/25 22:16
# @Author  : milittle
# @Site    : www.weaf.top
# @File    : train.py
# @Software: PyCharm

import tensorflow as tf
import numpy as np
 
from siameasenetwork.mnist import util # 这是外部依赖文件
from siameasenetwork.mnist.DataShuffler import * #这是打乱数据文件
from siameasenetwork.mnist.lenet import Lenet # 主网络结构

SEED = 10


def compute_euclidean_distance(x, y):
    """
    Computes the euclidean distance between two tensorflow variables
    """

    with tf.name_scope('euclidean_distance') as scope:
        #d = tf.square(tf.sub(x, y))
        #d = tf.sqrt(tf.reduce_sum(d)) # What about the axis ???
        d = tf.sqrt(tf.reduce_sum(tf.square(tf.subtract(x, y)), 1))
        return d


def compute_contrastive_loss(left_feature, right_feature, label, margin, is_target_set_train=True):

    """
    Compute the contrastive loss as in

    http://yann.lecun.com/exdb/publis/pdf/hadsell-chopra-lecun-06.pdf

    L = 0.5 * (Y) * D^2 + 0.5 * (1-Y) * {max(0, margin - D)}^2

    OR MAYBE THAT

    L = 0.5 * (1-Y) * D^2 + 0.5 * (Y) * {max(0, margin - D)}^2

    **Parameters**
     left_feature: First element of the pair
     right_feature: Second element of the pair
     label: Label of the pair (0 or 1)
     margin: Contrastive margin

    **Returns**
     Return the loss operation

    """

    with tf.name_scope("contrastive_loss"):
        label = tf.to_float(label)
        one = tf.constant(1.0)

        d = compute_euclidean_distance(left_feature, right_feature)

        d_sqrt = tf.sqrt(d)

        loss = label * tf.square(tf.maximum(0., margin - d_sqrt)) + (1 - label) * d

        loss = 0.5 * tf.reduce_mean(loss)
        # #first_part = tf.mul(one - label, tf.square(d))  # (Y-1)*(d^2)
        # #first_part = tf.mul(label, tf.square(d))  # (Y-1)*(d^2)
        # between_class = tf.exp(tf.multiply(one-label, tf.square(d)))  # (1-Y)*(d^2)
        # max_part = tf.square(tf.maximum(margin-d, 0))
        #
        # within_class = tf.multiply(label, max_part)  # (Y) * max((margin - d)^2, 0)
        # #second_part = tf.mul(one-label, max_part)  # (Y) * max((margin - d)^2, 0)
        #
        # loss = 0.5 * tf.reduce_mean(within_class + between_class)

        # return loss, tf.reduce_mean(within_class), tf.reduce_mean(between_class)
        return loss

def compute_contrastive_loss_andre(left_feature, right_feature, label, margin, is_target_set_train=True):

    """
    Compute the contrastive loss as in

    https://gitlab.idiap.ch/biometric/xfacereclib.cnn/blob/master/xfacereclib/cnn/scripts/experiment.py#L156


    With Y = [-1 +1] --> [POSITIVE_PAIR NEGATIVE_PAIR]

    L = log( m + exp( Y * d^2)) / N



    **Parameters**
     left_feature: First element of the pair
     right_feature: Second element of the pair
     label: Label of the pair (0 or 1)
     margin: Contrastive margin

    **Returns**
     Return the loss operation

    """

    with tf.name_scope("contrastive_loss_andre"):
        label = tf.to_float(label)
        d = compute_euclidean_distance(left_feature, right_feature)

        loss = tf.log(tf.exp(tf.multiply(label, d)))
        loss = tf.reduce_mean(loss)

        # Within class part
        genuine_factor = tf.multiply(label-1, 0.5)
        within_class = tf.reduce_mean(tf.log(tf.exp(tf.multiply(genuine_factor, d))))

        # Between class part
        impostor_factor = tf.multiply(label + 1, 0.5)
        between_class = tf.reduce_mean(tf.log(tf.exp(tf.multiply(impostor_factor, d))))

        # first_part = tf.mul(one - label, tf.square(d))  # (Y-1)*(d^2)
        return loss, between_class, within_class


def main():
    BATCH_SIZE = 128
    BATCH_SIZE_TEST = 128
    #BATCH_SIZE_TEST = 300
    ITERATIONS = 10000
    VALIDATION_TEST = 100
    perc_train = 0.9
    CONTRASTIVE_MARGIN = 1
    USE_GPU = True
    ANDRE_LOSS = False
    LOG_NAME = 'logs'

    train_data, train_labels, test_data, test_labels = util.load_mnist()

    data = np.concatenate((train_data, test_data), axis=0)
    label = np.concatenate((train_labels, test_labels), axis=0)

    data_shuffler = DataShuffler(train_data, train_labels, scale=True)

    # Creating the variables
    lenet_architecture = Lenet(seed=SEED, use_gpu=USE_GPU)
    # Siamease place holders - Training
    train_left_data = tf.placeholder(tf.float32, shape=(BATCH_SIZE*2, 28, 28, 1), name="left")
    train_right_data = tf.placeholder(tf.float32, shape=(BATCH_SIZE*2, 28, 28, 1), name="right")
    labels_data = tf.placeholder(tf.int32, shape=BATCH_SIZE*2)
    # Creating the graphs for training
    lenet_train_left = lenet_architecture.create_lenet(train_left_data)
    lenet_train_right = lenet_architecture.create_lenet(train_right_data)

    # Siamease place holders - Validation
    validation_data = tf.placeholder(tf.float32, shape=(data_shuffler.validation_data.shape[0], 28, 28, 1), name="validation")
    labels_data_validation = tf.placeholder(tf.int32, shape=BATCH_SIZE_TEST)
    # Creating the graphs for validation
    lenet_validation = lenet_architecture.create_lenet(validation_data, train=False)

    if ANDRE_LOSS:
        loss, between_class, within_class = compute_contrastive_loss_andre(lenet_train_left, lenet_train_right, labels_data, CONTRASTIVE_MARGIN)
    else:
        # Regular contrastive loss
        loss = compute_contrastive_loss(lenet_train_left, lenet_train_right, labels_data, CONTRASTIVE_MARGIN)


    distances = compute_euclidean_distance(lenet_train_left, lenet_train_right)


    #regularizer = (tf.nn.l2_loss(lenet_architecture.W_fc1) + tf.nn.l2_loss(lenet_architecture.b_fc1) +
    #                tf.nn.l2_loss(lenet_architecture.W_fc2) + tf.nn.l2_loss(lenet_architecture.b_fc2))
    #loss += 5e-4 * regularizer

    # Defining training parameters
    batch = tf.Variable(0)
    learning_rate = tf.train.exponential_decay(
        0.01, # Learning rate
        batch * BATCH_SIZE,
        data_shuffler.train_data.shape[0],
        0.95 # Decay step
    )

    #optimizer = tf.train.GradientDescentOptimizer(learning_rate).minimize(loss, global_step=batch)
    optimizer = tf.train.AdamOptimizer(0.01, name='Adam').minimize(loss, global_step=batch)

    # pp = PdfPages("groups.pdf")
    # Training
    with tf.Session() as session:

        #Trying to write things on tensor board
        train_writer = tf.summary.FileWriter(LOG_NAME + '/train',
                                              session.graph)

        test_writer = tf.summary.FileWriter(LOG_NAME + '/test',
                                              session.graph)

        tf.summary.scalar('loss', loss)
        # tf.summary.scalar('between_class', between_class)
        # tf.summary.scalar('within_class', within_class)
        tf.summary.scalar('lr', learning_rate)
        merged = tf.summary.merge_all()

        init = tf.global_variables_initializer()
        session.run(init)

        for step in range(ITERATIONS+1):

            batch_left, batch_right, labels = data_shuffler.get_pair(BATCH_SIZE, zero_one_labels=not ANDRE_LOSS)

            feed_dict = {train_left_data: batch_left,
                         train_right_data: batch_right,
                         labels_data: labels}

            _, l, lr, summary = session.run([optimizer, loss, learning_rate, merged], feed_dict=feed_dict)
            train_writer.add_summary(summary, step)

            print("Step {0}. Loss Validation = {1}".
                  format(step, l))

            if step % VALIDATION_TEST == 0:

                # Siamese validation
                batch_left, batch_right, labels = data_shuffler.get_pair(n_pair=BATCH_SIZE_TEST,
                                                                         is_target_set_train=False,
                                                                         zero_one_labels=not ANDRE_LOSS)
                feed_dict = {train_left_data: batch_left,
                             train_right_data: batch_right,
                             labels_data: labels}

                d, lv, summary = session.run([distances, loss, merged], feed_dict=feed_dict)
                test_writer.add_summary(summary, step)


                ###################################
                # Siamese as a feature extractor
                ###################################
                batch_train_data, batch_train_labels = data_shuffler.get_batch(
                    data_shuffler.validation_data.shape[0],
                    train_dataset=True)

                features_train = session.run(lenet_validation,
                                             feed_dict={validation_data: batch_train_data[:]})

                batch_validation_data, batch_validation_labels = data_shuffler.get_batch(
                    data_shuffler.validation_data.shape[0],
                    train_dataset=False)

                features_validation = session.run(lenet_validation,
                                                  feed_dict={validation_data: batch_validation_data[:]})

                acc = util.compute_accuracy(features_train, batch_train_labels, features_validation, batch_validation_labels, 10)

                print("Step {0}. Loss Validation = {1}, acc = {2}".
                      format(step, lv, acc))

                # fig = util.plot_embedding_lda(features_validation, batch_validation_labels)
                # pp.savefig(fig)

                #if ANDRE_LOSS:
                #    positive_scores = d[numpy.where(labels == -1)[0]].astype("float")
                #    negative_scores = d[numpy.where(labels == 1)[0]].astype("float")
                #else:
                #    positive_scores = d[numpy.where(labels == 1)[0]].astype("float")
                #    negative_scores = d[numpy.where(labels == 0)[0]].astype("float")

                #threshold = bob.measure.eer_threshold(negative_scores, positive_scores)
                #far, frr = bob.measure.farfrr(negative_scores, positive_scores, threshold)
                #eer = ((far + frr) / 2.) * 100.

        print("End !!")
        train_writer.close()
        test_writer.close()
        # pp.close()


if __name__ == '__main__':
    main()
```

lenet.py

```
#!/usr/bin/env python
# vim: set fileencoding=utf-8 :
# @author: Tiago de Freitas Pereira <tiago.pereira@idiap.ch>
# @date: Wed 11 May 2016 09:39:36 CEST 

"""
Class that creates the lenet architecture
"""

from siameasenetwork.mnist.util import *


class Lenet(object):

    def __init__(self,
                 conv1_kernel_size=5,
                 conv1_output=16,

                 conv2_kernel_size=5,
                 conv2_output=32,

                 fc1_output=400,
                 n_classes=10,

                 seed=10, use_gpu = False):
        """
        Create all the necessary variables for this CNN
        **Parameters**
            conv1_kernel_size=5,
            conv1_output=32,
            conv2_kernel_size=5,
            conv2_output=64,
            fc1_output=400,
            n_classes=10
            seed = 10
        """
        # First convolutional
        self.W_conv1 = create_weight_variables([conv1_kernel_size, conv1_kernel_size, 1, conv1_output], seed=seed, name="W_conv1", use_gpu=use_gpu)
        self.b_conv1 = create_bias_variables([conv1_output], name="bias_conv1", use_gpu=use_gpu)

        # Second convolutional
        self.W_conv2 = create_weight_variables([conv2_kernel_size, conv2_kernel_size, conv1_output, conv2_output], seed=seed, name="W_conv2", use_gpu=use_gpu)
        self.b_conv2 = create_bias_variables([conv2_output], name="bias_conv2", use_gpu=use_gpu)

        # First fc
        self.W_fc1 = create_weight_variables([(28 // 4) * (28 // 4) * conv2_output, fc1_output], seed=seed, name="W_fc1", use_gpu=use_gpu)
        self.b_fc1 = create_bias_variables([fc1_output], name="bias_fc1", use_gpu=use_gpu)

        # Second FC fc
        self.W_fc2 = create_weight_variables([fc1_output, n_classes], seed=seed, name="W_fc2", use_gpu=use_gpu)
        self.b_fc2 = create_bias_variables([n_classes], name="bias_fc2", use_gpu=use_gpu)

        self.seed = seed

    def create_lenet(self, data, train=True):
        """
        Create the Lenet Architecture
        **Parameters**
          data: Input data
          train:
        **Returns
          features_back: Features for backpropagation
          features_val: Features for validation
        """

        # Creating the architecture
        # First convolutional
        with tf.name_scope('conv_1') as scope:
            conv1 = create_conv2d(data, self.W_conv1)
        # relu1 = create_relu(conv1, self.b_conv1)
        # relu1 = create_sigmoid(conv1, self.b_conv1)

        with tf.name_scope('tanh_1') as scope:
            tanh_1 = create_tanh(conv1, self.b_conv1)


        # Pooling
        # pool1 = create_max_pool(relu1)
        # pool1 = create_max_pool(relu1)
        with tf.name_scope('pool_1') as scope:
            pool1 = create_max_pool(tanh_1)


        # Second convolutional
        with tf.name_scope('conv_2') as scope:
            conv2 = create_conv2d(pool1, self.W_conv2)
        # relu2 = create_relu(conv2, self.b_conv2)
        # relu2 = create_sigmoid(conv2, self.b_conv2)


        with tf.name_scope('tanh_2') as scope:
            # pool2 = create_max_pool(relu2)
            #tanh_2 = create_relu(conv2, self.b_conv2)
            # pool2 = create_max_pool(conv2)
            tanh_2 = create_tanh(conv2, self.b_conv2)

        # Pooling
        with tf.name_scope('pool_2') as scope:
            pool2 = create_max_pool(tanh_2)


        #if train:
            #pool2 = tf.nn.dropout(pool2, 0.5, seed=self.seed)

        # Reshaping all the convolved images to 2D to feed the FC layers
        # FC1
        with tf.name_scope('fc_1') as scope:
            pool_shape = pool2.get_shape().as_list()
            reshape = tf.reshape(pool2, [pool_shape[0], pool_shape[1] * pool_shape[2] * pool_shape[3]])
            #fc1 = tf.nn.relu(tf.matmul(reshape, self.W_fc1) + self.b_fc1)
            fc1 = tf.nn.tanh(tf.matmul(reshape, self.W_fc1) + self.b_fc1)

        #if train:
            #fc1 = tf.nn.dropout(fc1, 0.5, seed=self.seed)

        # FC2
        with tf.name_scope('fc_2') as scope:
            fc2 = tf.matmul(fc1, self.W_fc2) + self.b_fc2
            #fc2 = tf.nn.softmax(tf.matmul(fc1, self.W_fc2) + self.b_fc2)

        return fc2
```

util.py

```
#!/usr/bin/env python
# vim: set fileencoding=utf-8 :
# @author: Tiago de Freitas Pereira <tiago.pereira@idiap.ch>
# @date: Wed 11 May 2016 09:39:36 CEST 

import numpy
import tensorflow as tf
numpy.random.seed(10)
from tensorflow.examples.tutorials.mnist import input_data



def create_weight_variables(shape, seed, name, use_gpu=False):
    """
    Create gaussian random neurons with mean 0 and std 0.1
    **Paramters**
      shape: Shape of the layer
    """

    #import ipdb; ipdb.set_trace()

    if len(shape) == 4:
        in_out = shape[0] * shape[1] * shape[2] + shape[3]
    else:
        in_out = shape[0] + shape[1]

    import math
    stddev = math.sqrt(3.0 / in_out) # XAVIER INITIALIZER (GAUSSIAN)

    initializer = tf.truncated_normal(shape, stddev=stddev, seed=seed)
    
    if use_gpu:
        with tf.device("/gpu"):
            return tf.get_variable(name, initializer=initializer, dtype=tf.float32)
    else:
        with tf.device("/cpu"):
            return tf.get_variable(name, initializer=initializer, dtype=tf.float32)


def create_bias_variables(shape, name, use_gpu=False):
    """
    Create the bias term
    """
    initializer = tf.constant(0.1, shape=shape)
    
    if use_gpu:
        with tf.device("/gpu"):
            return tf.get_variable(name, initializer=initializer, dtype=tf.float32)
    else:
        with tf.device("/cpu"):
            return tf.get_variable(name, initializer=initializer, dtype=tf.float32)
    
    
   


def create_conv2d(x, W):
    """
    Create a convolutional kernel with 1 pixel of stride
    **Parameters**
        x: input layer
        W: Neurons
    """
    return tf.nn.conv2d(x, W, strides=[1, 1, 1, 1], padding='SAME')


def create_max_pool(x):
    """
    Create max pooling using a patch of 2x2
    **Parameters**
        x: input layer
    """
    return tf.nn.max_pool(x, ksize=[1, 2, 2, 1], strides=[1, 2, 2, 1], padding='SAME')


def create_tanh(x, bias):
    """
    Create the Tanh activations
    **Parameters**
        x: input layer
        bias: bias term
    """

    return tf.nn.tanh(tf.nn.bias_add(x, bias))


def create_relu(x, bias):
    """
    Create the ReLU activations
    **Parameters**
        x: input layer
        bias: bias term
    """

    return tf.nn.relu(tf.nn.bias_add(x, bias))


def create_sigmoid(x, bias):
    """
    Create the Sigmoid activations
    **Parameters**
        x: input layer
        bias: bias term
    """

    return tf.nn.sigmoid(tf.nn.bias_add(x, bias))


def scale_mean_norm(data, scale=0.00390625):
    mean = numpy.mean(data)
    data = (data - mean) * scale

    return data


def evaluate_softmax(data, labels, session, network, data_node):

    """
    Evaluate the network assuming that the output layer is a softmax
    """

    predictions = numpy.argmax(session.run(
        network,
        feed_dict={data_node: data[:]}), 1)

    return 100. * numpy.sum(predictions == labels) / predictions.shape[0]


def load_mnist():
    mnist_data = input_data.read_data_sets("MNIST_data/", one_hot=True)
    train_data = mnist_data.train.images
    test_data = mnist_data.test.images
    train_labels = mnist_data.train.labels
    test_labels = mnist_data.test.labels

    return train_data, train_labels, test_data, test_labels


def plot_embedding_pca(features, labels):
    """
    Trains a PCA using bob, reducing the features to dimension 2 and plot it the possible clusters
    :param features:
    :param labels:
    :return:
    """

    import bob.learn.linear
    import matplotlib.pyplot as mpl

    colors = ['#FF0000', '#FFFF00', '#FF00FF', '#00FFFF', '#000000',
             '#AA0000', '#AAAA00', '#AA00AA', '#00AAAA', '#330000']

    # Training PCA
    trainer = bob.learn.linear.PCATrainer()
    machine, lamb = trainer.train(features.astype("float64"))

    # Getting the first two most relevant features
    projected_features = machine(features.astype("float64"))[:, 0:2]

    # Plotting the classes
    n_classes = max(labels)+1
    fig = mpl.figure()

    for i in range(n_classes):
        indexes = numpy.where(labels == i)[0]

        selected_features = projected_features[indexes,:]
        mpl.scatter(selected_features[:, 0], selected_features[:, 1],
                 marker='.', c=colors[i], linewidths=0, label=str(i))
    mpl.legend()
    return fig

def plot_embedding_lda(features, labels):
    """
    Trains a LDA using bob, reducing the features to dimension 2 and plot it the possible clusters
    :param features:
    :param labels:
    :return:
    """

    import bob.learn.linear
    import matplotlib.pyplot as mpl

    colors = ['#FF0000', '#FFFF00', '#FF00FF', '#00FFFF', '#000000',
             '#AA0000', '#AAAA00', '#AA00AA', '#00AAAA', '#330000']
    n_classes = max(labels)+1


    # Training PCA
    trainer = bob.learn.linear.FisherLDATrainer(use_pinv=True)
    lda_features = []
    for i in range(n_classes):
        indexes = numpy.where(labels == i)[0]
        lda_features.append(features[indexes, :].astype("float64"))

    machine, lamb = trainer.train(lda_features)

    #import ipdb; ipdb.set_trace();


    # Getting the first two most relevant features
    projected_features = machine(features.astype("float64"))[:, 0:2]

    # Plotting the classes
    fig = mpl.figure()

    for i in range(n_classes):
        indexes = numpy.where(labels == i)[0]

        selected_features = projected_features[indexes,:]
        mpl.scatter(selected_features[:, 0], selected_features[:, 1],
                 marker='.', c=colors[i], linewidths=0, label=str(i))
    mpl.legend()
    return fig


def compute_eer(data_train, labels_train, data_validation, labels_validation, n_classes):
    from scipy.spatial.distance import cosine

    # Creating client models
    models = []
    for i in range(n_classes):
        labels_num = numpy.argmax(labels_train, axis=1)
        indexes = numpy.where(labels_num == i)
        models.append(numpy.mean(data_train[indexes, :], axis=0))

    # Probing
    positive_scores = numpy.zeros(shape=0)
    negative_scores = numpy.zeros(shape=0)

    for i in range(n_classes):
        # Positive scoring
        val_num = numpy.argmax(labels_validation)
        indexes = numpy.where(val_num == i)
        positive_data = data_validation[indexes, :]
        p = [cosine(models[i], positive_data[j]) for j in range(positive_data.shape[0])]
        positive_scores = numpy.hstack((positive_scores, p))

        # negative scoring
        indexes = numpy.where(val_num != i)
        negative_data = data_validation[indexes, :]
        n = [cosine(models[i], negative_data[j]) for j in range(negative_data.shape[0])]
        negative_scores = numpy.hstack((negative_scores, n))

    # Computing performance based on EER
    negative_scores = (-1) * negative_scores
    positive_scores = (-1) * positive_scores

    # threshold = bob.measure.eer_threshold(negative_scores, positive_scores)
    # far, frr = bob.measure.farfrr(negative_scores, positive_scores, threshold)
    # eer = (far + frr) / 2.

    return negative_scores, positive_scores


def compute_accuracy(data_train, labels_train, data_validation, labels_validation, n_classes):
    from scipy.spatial.distance import cosine

    # Creating client models
    models = []
    for i in range(n_classes):
        index = numpy.argmax(labels_train, axis=1)
        indexes = numpy.where(index == i)[0]
        data_t = [data_train[i] for i in indexes]
        models.append(numpy.mean(data_t, axis=0))

    # Probing
    tp = 0
    for i in range(data_validation.shape[0]):

        d = data_validation[i,:]
        l = labels_validation[i]

        scores = [cosine(m, d) for m in models]
        predict = numpy.argmin(scores)

        ll = numpy.argmax(l, axis=0)
        if predict == ll:
            tp += 1

    return (float(tp) / data_validation.shape[0]) * 100


def compute_acc(data_val, labels_val):
    tp = 0
    data_size = data_val.shape[0]
    for i in range(data_size):
        index = numpy.argmax(data_val[i])
        if index == numpy.argmax(labels_val[i]):
            tp += 1


    return (float(tp) / data_size)

```

DataShuffler.py

```
#!/usr/bin/env python
# vim: set fileencoding=utf-8 :
# @author: Tiago de Freitas Pereira <tiago.pereira@idiap.ch>
# @date: Wed 11 May 2016 09:39:36 CEST 

import numpy


def scale_mean_norm(data, scale=0.00390625):
    mean = numpy.mean(data)
    data = (data - mean) * scale

    return data, mean


"""
Data
"""


class DataShuffler(object):
    def __init__(self, data, labels, perc_train=0.9, scale=True):
        """
         Some base functions for neural networks
         **Parameters**
           data:
        """
        scale_value = 0.00390625

        total_samples = data.shape[0] #总共的样本数量

        indexes = numpy.array(range(total_samples))
        numpy.random.shuffle(indexes)

        # Spliting train and validation
        train_samples = int(round(total_samples * perc_train)) #训练样本数目
        validation_samples = total_samples - train_samples #验证集样本数目
        data = numpy.reshape(data, (data.shape[0], 28, 28, 1)) #所有输入样本

        self.train_data = data[indexes[0:train_samples], :, :, :] # 训练样本
        self.train_labels = labels[indexes[0:train_samples]] # 训练labels

        self.validation_data = data[indexes[train_samples:train_samples + validation_samples], :, :, :]
        self.validation_labels = labels[indexes[train_samples:train_samples + validation_samples]]
        self.total_labels = 10

        if scale:
            # data = scale_minmax_norm(data,lower_bound = -1, upper_bound = 1)
            self.train_data, self.mean = scale_mean_norm(self.train_data)
            self.validation_data = (self.validation_data - self.mean) * scale_value

    def get_batch(self, n_samples, train_dataset=True):

        if train_dataset:
            data = self.train_data
            label = self.train_labels
        else:
            data = self.validation_data
            label = self.validation_labels

        # Shuffling samples
        indexes = numpy.array(range(data.shape[0]))
        numpy.random.shuffle(indexes)

        selected_data = data[indexes[0:n_samples], :, :, :]
        selected_labels = label[indexes[0:n_samples]]

        return selected_data.astype("float32"), selected_labels

    def get_pair(self, n_pair=1, is_target_set_train=True, zero_one_labels=True):
        """
        Get a random pair of samples
        **Parameters**
            is_target_set_train: Defining the target set to get the batch
        **Return**
        """

        def get_genuine_or_not(input_data, input_labels, genuine=True):

            if genuine:
                # TODO: THIS KEY SELECTION NEEDS TO BE MORE EFFICIENT

                # Getting a client
                index = numpy.random.randint(self.total_labels)

                # Getting the indexes of the data from a particular client
                arg_max = numpy.argmax(input_labels, axis=1)
                indexes = numpy.where(arg_max == index)[0]
                numpy.random.shuffle(indexes)

                # Picking a pair
                data = input_data[indexes[0], :, :, :]
                data_p = input_data[indexes[1], :, :, :]

            else:
                # Picking a pair from different clients
                index = numpy.random.choice(self.total_labels, 2, replace=False)

                # Getting the indexes of the two clients
                arg_max = numpy.argmax(input_labels, axis=1)
                indexes = numpy.where(arg_max == index[0])[0]
                indexes_p = numpy.where(arg_max == index[1])[0]
                numpy.random.shuffle(indexes)
                numpy.random.shuffle(indexes_p)

                # Picking a pair
                data = input_data[indexes[0], :, :, :]
                data_p = input_data[indexes_p[0], :, :, :]

            return data, data_p

        if is_target_set_train:
            target_data = self.train_data
            target_labels = self.train_labels
        else:
            target_data = self.validation_data
            target_labels = self.validation_labels

        total_data = n_pair * 2
        c = target_data.shape[3]
        w = target_data.shape[1]
        h = target_data.shape[2]

        data = numpy.zeros(shape=(total_data, w, h, c), dtype='float32')
        data_p = numpy.zeros(shape=(total_data, w, h, c), dtype='float32')
        labels_siamese = numpy.zeros(shape=total_data, dtype='float32')

        genuine = True
        for i in range(total_data):
            data[i, :, :, :], data_p[i, :, :, :] = get_genuine_or_not(target_data, target_labels, genuine=genuine)
            if zero_one_labels:
                labels_siamese[i] = not genuine
            else:
                labels_siamese[i] = -1 if genuine else +1
            genuine = not genuine

        return data, data_p, labels_siamese

    def get_triplet(self, n_labels, n_triplets=1, is_target_set_train=True):
        """
        Get a triplet
        **Parameters**
            is_target_set_train: Defining the target set to get the batch
        **Return**
        """

        def get_one_triplet(input_data, input_labels):

            # Getting a pair of clients
            index = numpy.random.choice(n_labels, 2, replace=False)
            label_positive = index[0]
            label_negative = index[1]

            # Getting the indexes of the data from a particular client
            indexes = numpy.where(input_labels == index[0])[0]
            numpy.random.shuffle(indexes)

            # Picking a positive pair
            data_anchor = input_data[indexes[0], :, :, :]
            data_positive = input_data[indexes[1], :, :, :]

            # Picking a negative sample
            indexes = numpy.where(input_labels == index[1])[0]
            numpy.random.shuffle(indexes)
            data_negative = input_data[indexes[0], :, :, :]

            return data_anchor, data_positive, data_negative, label_positive, label_positive, label_negative

        if is_target_set_train:
            target_data = self.train_data
            target_labels = self.train_labels
        else:
            target_data = self.validation_data
            target_labels = self.validation_labels

        c = target_data.shape[3]
        w = target_data.shape[1]
        h = target_data.shape[2]

        data_a = numpy.zeros(shape=(n_triplets, w, h, c), dtype='float32')
        data_p = numpy.zeros(shape=(n_triplets, w, h, c), dtype='float32')
        data_n = numpy.zeros(shape=(n_triplets, w, h, c), dtype='float32')
        labels_a = numpy.zeros(shape=n_triplets, dtype='float32')
        labels_p = numpy.zeros(shape=n_triplets, dtype='float32')
        labels_n = numpy.zeros(shape=n_triplets, dtype='float32')

        for i in range(n_triplets):
            data_a[i, :, :, :], data_p[i, :, :, :], data_n[i, :, :, :], \
            labels_a[i], labels_p[i], labels_n[i] = \
                get_one_triplet(target_data, target_labels)

        return data_a, data_p, data_n, labels_a, labels_p, labels_n
```

