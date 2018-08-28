---
title: TensorFlow逻辑回归详解
tags: TensorFlow
category: TensorFlow
mathjax: true
thumbnail: 'https://s1.ax1x.com/2018/03/18/9oakkQ.png'
author: milittle
abbrlink: befe0ef0
date: 2018-05-20 19:31:09
---

# TensorFlow Logistic Regression

还是得先和大家开个头，我最近有点忙，所以有时候需要抽开时间补上拉下的博客，这是我2018年5月20号补5月七号那个月的博客，先来介绍一下逻辑回归的概念，也就是相对于线性回归，其实逻辑回归就是一个分类问题，大家可以这么理解，也就是最后将值得分布确定在几类当中，就像我们最开始在博客一开始学习得fashion mnist一样，只不过，我们这节课来一个，简单得二分类问题。也就是：
$$
y=sigmoid(A*x+b)
$$
最后我们得到得y的预测值都会是0，或者是1，我们使用的数据是github的一个数据：

1. 正式开始代码的编写，其实很简单。

```python
import matplotlib.pyplot as plt
import numpy as np
import tensorflow as tf
import requests
from tensorflow.python.framework import ops
import os.path
import csv

# 上面一如既往的模块导入
ops.reset_default_graph()

# 创建我们的会话
sess = tf.Session()



# 数据的文件名
birth_weight_file = 'birth_weight.csv'

# 下载数据
if not os.path.exists(birth_weight_file):
    
    birthdata_url = 'https://github.com/nfmcclure/tensorflow_cookbook/' + \
    'raw/master/01_Introduction/07_Working_with_Data_Sources/birthweight_data/birthweight.dat'
    birth_file = requests.get(birthdata_url)
    birth_data = birth_file.text.split('\r\n')
    birth_header = birth_data[0].split('\t')
    birth_data = [[float(x) for x in y.split('\t') if len(x)>=1] for y in birth_data[1:] if len(y)>=1]
    with open(birth_weight_file, "w") as f:
        writer = csv.writer(f)
        writer.writerow(birth_header)
        writer.writerows(birth_data)
        f.close()

# 读取数据，就像我们之前说的，要放在placeholder得数据是要加载在数组里面得
birth_data = []
with open(birth_weight_file, newline='') as csvfile:
     csv_reader = csv.reader(csvfile)
     birth_header = next(csv_reader)
     for row in csv_reader:
         birth_data.append(row)

birth_data = [[float(x) for x in row] for row in birth_data]

# 分数据
y_vals = np.array([x[0] for x in birth_data])
x_vals = np.array([x[1:8] for x in birth_data])

seed = 99
np.random.seed(seed)
tf.set_random_seed(seed)

# 分离训练集和测试集
train_indices = np.random.choice(len(x_vals), round(len(x_vals)*0.8), replace=False)
test_indices = np.array(list(set(range(len(x_vals))) - set(train_indices)))
x_vals_train = x_vals[train_indices]
x_vals_test = x_vals[test_indices]
y_vals_train = y_vals[train_indices]
y_vals_test = y_vals[test_indices]

# 归一化数据函数
def normalize_cols(m):
    col_max = m.max(axis=0)
    col_min = m.min(axis=0)
    return (m-col_min) / (col_max - col_min)
    
x_vals_train = np.nan_to_num(normalize_cols(x_vals_train))
x_vals_test = np.nan_to_num(normalize_cols(x_vals_test))



# 定义批处理大小
batch_size = 25

# 初始化 placeholders
x_data = tf.placeholder(shape=[None, 7], dtype=tf.float32)
y_target = tf.placeholder(shape=[None, 1], dtype=tf.float32)

# 创建回归变量
A = tf.Variable(tf.random_normal(shape=[7,1]))
b = tf.Variable(tf.random_normal(shape=[1,1]))

# 定义模型图的操作
model_output = tf.add(tf.matmul(x_data, A), b)

# 定义交叉熵损失函数
loss = tf.reduce_mean(tf.nn.sigmoid_cross_entropy_with_logits(logits=model_output, labels=y_target))

# 定义优化器
my_opt = tf.train.GradientDescentOptimizer(0.01)
train_step = my_opt.minimize(loss)


# 初始化全局变量
init = tf.global_variables_initializer()
sess.run(init)

# 预测值
prediction = tf.round(tf.sigmoid(model_output))
predictions_correct = tf.cast(tf.equal(prediction, y_target), tf.float32)
accuracy = tf.reduce_mean(predictions_correct)

# 开始训练
loss_vec = []
train_acc = []
test_acc = []
for i in range(1500):
    rand_index = np.random.choice(len(x_vals_train), size=batch_size)
    rand_x = x_vals_train[rand_index]
    rand_y = np.transpose([y_vals_train[rand_index]])
    sess.run(train_step, feed_dict={x_data: rand_x, y_target: rand_y})

    temp_loss = sess.run(loss, feed_dict={x_data: rand_x, y_target: rand_y})
    loss_vec.append(temp_loss)
    temp_acc_train = sess.run(accuracy, feed_dict={x_data: x_vals_train, y_target: np.transpose([y_vals_train])})
    train_acc.append(temp_acc_train)
    temp_acc_test = sess.run(accuracy, feed_dict={x_data: x_vals_test, y_target: np.transpose([y_vals_test])})
    test_acc.append(temp_acc_test)
    if (i+1)%300==0: # 每三百次打一次损失函数
        print('Loss = ' + str(temp_loss))
        
        
#
# 画出loss的值
plt.plot(loss_vec, 'k-')
plt.title('Cross Entropy Loss per Generation')
plt.xlabel('Generation')
plt.ylabel('Cross Entropy Loss')
plt.show()

# 画出训练和测试的准确率
plt.plot(train_acc, 'k-', label='Train Set Accuracy')
plt.plot(test_acc, 'r--', label='Test Set Accuracy')
plt.title('Train and Test Accuracy')
plt.xlabel('Generation')
plt.ylabel('Accuracy')
plt.legend(loc='lower right')
plt.show()
```

这是补上周的博客，让大家认识一下逻辑回归模型的基本建立。有什么疑问，发邮件air@weaf.top。期待你哦！！！