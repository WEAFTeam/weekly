---
title: chapter-06-AIR
date: 2018-04-22 14:31:24
tags: TensorFlow
category: TensorFlow
mathjax: true
thumbnail: 'https://s1.ax1x.com/2018/03/18/9oakkQ.png'
author: milittle
---

# TensorFlow 基础（3）

hello,大家好，几天我们继续学习基础知识，为我们以后建立模型打下基础。主要是最近有点忙，所以每一周的内容会少一些，请大家谅解，随后慢慢加快进度。

>这一次我们讲一下batch的概念，以及一些基本的操作，在之前的文章中，我们也讲过batch这个概念的。大家应该不会陌生

1. Working with Batch and Stochastic Training

```python
import tensorflow as tf
import matplotlib.pyplot as plt
import numpy as np
from tensorflow.python.framework import ops
ops.reset_default_graph()

sess = tf.Session()

# 随机梯度训练

# 生成数据

x_vals = np.random.normal(1., 0.1, 100)
y_vals = np.repeat(10., 100)

x_data = tf.placeholder(shape = [1], dtype = tf.float32)
y_target = tf.placeholder(shape = [1], dtype = tf.float32)

# A就相当于权重咯
A = tf.Variable(tf.random_normal(shape = [1]))

my_output = tf.multiply(x_data, A)


# 注意我们上一次的loss函数哦

loss = tf.square(my_output - y_target)

my_opt = tf.train.GradientDescentOptimizer(0.02) # 0.02就是学习率

train_step = my_opt.minimize(loss)

init = tf.global_variables_initializer()
sess.run(init)

# 开始训练模型
loss_stochastic = []

for i in range(100):
    rand_index = np.random.choice(100) # 随机选取一个样本进行训练
    rand_x = [x_vals[rand_index]]
    rand_y = [y_vals[rand_index]]
    sess.run(train_step, feed_dict = {x_data: rand_x, y_target: rand_y})
    if (i + 1) % 5 == 0:
        print('Step #' + str(i + 1) + ' A = ' + str(sess.run(A)))
        temp_loss = sess.run(loss, feed_dict = {x_data: rand_x, y_target: rand_y})
        print('Loss = ' + str(temp_loss))
        loss_stochastic.append(temp_loss)

        
# batch train      
ops.reset_default_graph()

sess = tf.Session()

batch_size = 25
x_vals = np.random.normal(1, 0.1, 100)
y_vals = np.repeat(10., 100)
x_data = tf.placeholder(shape=[None, 1], dtype=tf.float32) # 看出来变化了吧？
y_target = tf.placeholder(shape=[None, 1], dtype=tf.float32) # 这里也是

A = tf.Variable(tf.random_normal(shape=[1,1]))

my_output = tf.matmul(x_data, A)
# 是不是l2loss
loss = tf.reduce_mean(tf.square(my_output - y_target))
# 这里已经强调过很多遍了
init = tf.global_variables_initializer()
sess.run(init)

# 这里是优化器
my_opt = tf.train.GradientDescentOptimizer(0.02)
train_step = my_opt.minimize(loss)

loss_batch = []
# Run Loop
for i in range(100):
    rand_index = np.random.choice(100, size=batch_size) # 看出来区别了么？
    rand_x = np.transpose([x_vals[rand_index]])
    rand_y = np.transpose([y_vals[rand_index]])
    sess.run(train_step, feed_dict={x_data: rand_x, y_target: rand_y})
    if (i+1)%5==0:
        print('Step #' + str(i+1) + ' A = ' + str(sess.run(A)))
        temp_loss = sess.run(loss, feed_dict={x_data: rand_x, y_target: rand_y})
        print('Loss = ' + str(temp_loss))
        loss_batch.append(temp_loss)

plt.plot(range(0, 100, 5), loss_stochastic, 'b-', label='Stochastic Loss')
plt.plot(range(0, 100, 5), loss_batch, 'r--', label='Batch Loss, size=20')
plt.legend(loc='upper right', prop={'size': 11})
plt.show()
```

让你感受一些，batch训练的loss收敛：

![-](https://s1.ax1x.com/2018/04/22/CMTbv9.png)

你就说上面的你理解没理解，没理解要好好理解理解了~！！！

2. 这个就是结合了，把所有上面价格的基础结合在一起，你说说是不是很棒

```python
# 联合所有的基础操作，弄一个分类的例子，期不期待
import matplotlib.pyplot as plt
import numpy as np
from sklearn import datasets
import tensorflow as tf
from tensorflow.python.framework import ops
ops.reset_default_graph()

iris = datasets.load_iris() # 将鸢尾花数据集下下来 总共是四个属性的数据集，根据四个种类，预测种类，总共三类
binary_target = np.array([1. if x==0 else 0. for x in iris.target]) # 将label作为二分类问题
iris_2d = np.array([[x[2], x[3]] for x in iris.data]) # 将后两个属性取出来用作种类预测

batch_size = 20

sess = tf.Session()

x1_data = tf.placeholder(shape=[None, 1], dtype=tf.float32)
x2_data = tf.placeholder(shape=[None, 1], dtype=tf.float32)
y_target = tf.placeholder(shape=[None, 1], dtype=tf.float32)

A = tf.Variable(tf.random_normal(shape=[1, 1]))
b = tf.Variable(tf.random_normal(shape=[1, 1]))

my_mult = tf.matmul(x2_data, A)
my_add = tf.add(my_mult, b)
my_output = tf.subtract(x1_data, my_add) # 你能不能自己使用公式写出我们预测种类的公式呢？

xentropy = tf.nn.sigmoid_cross_entropy_with_logits(logits = my_output, labels = y_target)

my_opt = tf.train.GradientDescentOptimizer(0.05)
train_step = my_opt.minimize(xentropy)

init = tf.global_variables_initializer()
sess.run(init)

for i in range(1000):
    rand_index = np.random.choice(len(iris_2d), size=batch_size)
    #rand_x = np.transpose([iris_2d[rand_index]])
    rand_x = iris_2d[rand_index]
    rand_x1 = np.array([[x[0]] for x in rand_x])
    rand_x2 = np.array([[x[1]] for x in rand_x])
    #rand_y = np.transpose([binary_target[rand_index]])
    rand_y = np.array([[y] for y in binary_target[rand_index]])
    sess.run(train_step, feed_dict={x1_data: rand_x1, x2_data: rand_x2, y_target: rand_y})
    if (i+1)%200==0:
        print('Step #' + str(i+1) + ' A = ' + str(sess.run(A)) + ', b = ' + str(sess.run(b)))

# Pull out slope/intercept
[[slope]] = sess.run(A)
[[intercept]] = sess.run(b)

# Create fitted line
x = np.linspace(0, 3, num=50)
ablineValues = []
for i in x:
  ablineValues.append(slope*i+intercept)

# Plot the fitted line over the data
setosa_x = [a[1] for i,a in enumerate(iris_2d) if binary_target[i]==1]
setosa_y = [a[0] for i,a in enumerate(iris_2d) if binary_target[i]==1]
non_setosa_x = [a[1] for i,a in enumerate(iris_2d) if binary_target[i]==0]
non_setosa_y = [a[0] for i,a in enumerate(iris_2d) if binary_target[i]==0]
plt.plot(setosa_x, setosa_y, 'rx', ms=10, mew=2, label='setosa')
plt.plot(non_setosa_x, non_setosa_y, 'ro', label='Non-setosa')
plt.plot(x, ablineValues, 'b-')
plt.xlim([0.0, 2.7])
plt.ylim([0.0, 7.1])
plt.suptitle('Linear Separator For I.setosa', fontsize=20)
plt.xlabel('Petal Length')
plt.ylabel('Petal Width')
plt.legend(loc='lower right')
plt.show()
```

![](https://s1.ax1x.com/2018/04/22/CM7GV0.png)

上面是分类的结果图

3. 验证模型：

接下来，我们会实现一个简单的回归模型和分类模型，分别做出他们的测试样例

```python
# 回归模型
import matplotlib.pyplot as plt
import numpy as np
import tensorflow as tf
from tensorflow.python.framework import ops
ops.reset_default_graph()

sess = tf.Session()
batch_size = 25

x_vals = np.random.normal(1, 0.1, 100)
y_vals = np.repeat(10., 100)
x_data = tf.placeholder(shape=[None, 1], dtype=tf.float32)
y_target = tf.placeholder(shape=[None, 1], dtype=tf.float32)

# 将数据分为训练集80%和测试集20%
train_indices = np.random.choice(len(x_vals), round(len(x_vals) * 0.8), replace = False)
test_indices = np.array(list(set(range(len(x_vals))) - set(train_indices)))
x_vals_train = x_vals[train_indices]
x_vals_test = x_vals[test_indices]
y_vals_train = y_vals[train_indices]
y_vals_test = y_vals[test_indices]

# 下面这些我们是不是写了好多遍了！！！
A = tf.Variable(tf.random_normal(shape=[1,1]))

my_output = tf.matmul(x_data, A)

# 还记得我么？你也见过我好多次了，为什么加reduce_mean? 你也知道batch可不是一个数据样本呀~
loss = tf.reduce_mean(tf.square(my_output - y_target))

# 创建优化器
my_opt = tf.train.GradientDescentOptimizer(0.02)
train_step = my_opt.minimize(loss)
init = tf.global_variables_initializer()
sess.run(init)

# 算了，我都不想写了，你说说我们写了多少遍了，该会了
for i in range(100):
    rand_index = np.random.choice(len(x_vals_train), size=batch_size)
    rand_x = np.transpose([x_vals_train[rand_index]])
    rand_y = np.transpose([y_vals_train[rand_index]])
    sess.run(train_step, feed_dict={x_data: rand_x, y_target: rand_y})
    if (i+1)%25==0:
        print('Step #' + str(i+1) + ' A = ' + str(sess.run(A)))
        print('Loss = ' + str(sess.run(loss, feed_dict={x_data: rand_x, y_target: rand_y})))

# 验证呀，一般训练出来的模型，我们要在测试集上面跑的很好，在测试集上面的准确率好了，没用，因为有过拟合的嫌疑，就是太认真了。泛化能力太差。
# 验证回归模型，那就要使用loss去衡量这个模型的好坏了
mse_test = sess.run(loss, feed_dict={x_data: np.transpose([x_vals_test]), y_target: np.transpose([y_vals_test])})
mse_train = sess.run(loss, feed_dict={x_data: np.transpose([x_vals_train]), y_target: np.transpose([y_vals_train])})
print('MSE on test:' + str(np.round(mse_test, 2)))
print('MSE on train:' + str(np.round(mse_train, 2)))


# 分类工作，小伙伴是不是一直觉得分类工作怎么能做，那是因为使用概率做的，也就是说，比如概率大于某一个阈值0.5 我们就分为某一类，如果小于我们就分为另一种，这是二分类，那么多分类怎么办，那么就要引入one-hot编码，这个我们后来慢慢深入。先来看例子
ops.reset_default_graph()
sess = tf.Session()
batch_size = 25

# 一毛一样
x_vals = np.concatenate((np.random.normal(-1, 1, 50), np.random.normal(2, 1, 50)))
y_vals = np.concatenate((np.repeat(0., 50), np.repeat(1., 50)))
x_data = tf.placeholder(shape=[1, None], dtype=tf.float32)
y_target = tf.placeholder(shape=[1, None], dtype=tf.float32)

train_indices = np.random.choice(len(x_vals), round(len(x_vals)*0.8), replace=False)
test_indices = np.array(list(set(range(len(x_vals))) - set(train_indices)))
x_vals_train = x_vals[train_indices]
x_vals_test = x_vals[test_indices]
y_vals_train = y_vals[train_indices]
y_vals_test = y_vals[test_indices]

A = tf.Variable(tf.random_normal(mean=10, shape=[1]))

my_output = tf.add(x_data, A) # 我们直接使用了一个加法操作

# 关键还是loss函数
xentropy = tf.reduce_mean(tf.nn.sigmoid_cross_entropy_with_logits(logits = my_output, labels = y_target))

my_opt = tf.train.GradientDescentOptimizer(0.05)
train_step = my_opt.minimize(xentropy)

init = tf.global_variables_initializer()
sess.run(init)

# Run loop
for i in range(1800):
    rand_index = np.random.choice(len(x_vals_train), size=batch_size)
    rand_x = [x_vals_train[rand_index]]
    rand_y = [y_vals_train[rand_index]]
    sess.run(train_step, feed_dict={x_data: rand_x, y_target: rand_y})
    if (i + 1) % 200 == 0:
        print('Step #' + str(i+1) + ' A = ' + str(sess.run(A)))
        print('Loss = ' + str(sess.run(xentropy, feed_dict={x_data: rand_x, y_target: rand_y})))
# 在测试集上面做测试
y_prediction = tf.squeeze(tf.round(tf.nn.sigmoid(tf.add(x_data, A))))
correct_prediction = tf.equal(y_prediction, y_target)
accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))
acc_value_test = sess.run(accuracy, feed_dict={x_data: [x_vals_test], y_target: [y_vals_test]})
acc_value_train = sess.run(accuracy, feed_dict={x_data: [x_vals_train], y_target: [y_vals_train]})
print('Accuracy on train set: ' + str(acc_value_train))
print('Accuracy on test set: ' + str(acc_value_test))

# 画出分类结果
A_result = -sess.run(A)
bins = np.linspace(-5, 5, 50)
plt.hist(x_vals[0:50], bins, alpha=0.5, label='N(-1,1)', color='white')
plt.hist(x_vals[50:100], bins[0:50], alpha=0.5, label='N(2,1)', color='red')
plt.plot((A_result, A_result), (0, 8), 'k--', linewidth=3, label='A = '+ str(np.round(A_result, 2)))
plt.legend(loc='upper right')
plt.title('Binary Classifier, Accuracy=' + str(np.round(acc_value_test, 2)))
plt.show()
```

总结：这次下来我们就把TensorFlow的所有基础内容讲完了，有什么问题，可以给我发邮件：air@weaf.top

希望大家把这些基础知识好好稳固一下，务必牢记于心，随后我们就会很顺利。

下一次我们就会开始一些基本算法~，基础学完不得好好联系一下么？

