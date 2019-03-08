---
title: tf-2.0-effective-info
date: 2019-03-08 16:03:04
category: TensorFlow
tags: TensorFlow
mathjax: true
thumbnail: 'https://s1.ax1x.com/2018/03/18/9oakkQ.png'
author: Milittle
---

## 如何高效使用TensorFlow2.0

#### Install: `pip install tensorflow==2.0.0-alpha0`

#### or Build From source https://www.weaf.top/posts/c05ce623/

-------------------------------------------------------分割线-----------------------------------------------------------------------------------------

#### 首先在2.0中，为了使得用户效率提高有许多改变。2.0中移除了许多冗余的APIs，使得更多的API一致，而且很好的和Python集成了Eager execution。（着实让人很舒服）[Eager execution](https://www.tensorflow.org/guide/eager)

#### 许多的RFCs对于2.0做的改变都有详细的解释.[RFCs](https://github.com/tensorflow/community/pulls?utf8=%E2%9C%93&q=is%3Apr) 假定你已经对TensorFlow1.x已经很熟悉的情况下，我们来介绍一下TensorFlow在开发中应该是什么样子的。

### 主要修改的简单总结

#### API 更加干净

许多的APIs在TF2.0中已经被移出掉。一些主要的改变包括`tf.app`,`tf.flags`和`tf.logging`，赞成现在开源的项目[absl-py](https://github.com/abseil/abseil-py), 重新安置了`tf.contrib`项目。清理了一些在`tf.*`模块到比如`tf.math`模块下。还有一些模块在TF2.0中具有相同的功能，比如`tf.summary`, `tf.keras.metrics`, 和 `tf.keras.optimizers`.最简单将这些名字命名为2.0的格式是通过[v2 upgrade script](https://www.tensorflow.org/alpha/guide/upgrade)

#### Eager execution

TensorFlow1.x要求通过tf.* API用户手动去缝合一棵抽象的语法树（就是那个graph）。要求用户手动编译抽象的语法树，通过`session.run()`输入张量来得到输出张量。在TensorFlow2.0中，用户可以像Python一样，立马得到结果，graph和sessions这些要求是去实现一些细节时候所使用到的功能。

#### 没有更多的Global

TensorFlow1.x中都使用了隐式的全局空间。当你调用`tf.Variable()`.这样就会放入到默认的图中。他就会留在图中，即使你忘了将它赋值给一个Python变量保存。当然你可以通过`tf.Variable()`来恢复它，哈哈哈，但是你只有在知道它构建时候的名字才能找到它，所以之前我们在TensorFlow1.x中建议大家，尽可能给创建的变量一个名字，而且每一层次使用`tf.variable_scope()`去管理是多么的重要。这样导致的结果呢，就会有很多机制去帮助用户去找到他们的创建过的变量，并在框架中找到他们的变量。比如Variable scope， global collections，帮助的方法比如`tf.get_global_step()`, `tf.global_variables_initializer()` 而且优化器隐式的计算了所有变量的梯度等等。TensorFlow2.0将这些机制都去除了，所以，哥们们，实在是太难学了。[Variables2.0RFC](https://github.com/tensorflow/community/pull/11)

还是要遵循默认的机制，保留对变量的跟踪，如果你失去了你的变量`tf.Variable()`，那么你将永远试去它，请珍惜它。它会被当作垃圾回收。

这样的话，我们就需要提出新的让用户跟踪自己变量的机制。但是只要是和Keras的对象打交道（随后会讲到），那么你会觉得很轻松。

#### Functions， not sessions

一个`session.run()`的调用，类似于一个函数的调用。你给一个函数输入，这个函数将给你一个输出。在TensorFlow2.0中。你可以将你的Python函数使用`tf.function`装饰器装饰。将其标记为JIT编译。因此TensorFlow2.0会将它作为一个单个图运行[Functions 2.0 RFC](https://github.com/tensorflow/community/pull/20),这种机制会让TensorFlow2.0获得所有图机制的好处（这个改进相当于将每一步运算可以加入到一个子图中，这样可以将所有的图联合起来就是一个模型，这个确实很牛叉）。

- 性能：这个函数可以被优化（节点剪枝，核融合等等）
- 可移植性：这个函数可以被导出核重导入（SavedModel 2.0 RFC）运行用户可以重用核共享模块函数。（我个人感觉这种方式重用性更高）

运行方式：

```python
# TensorFlow 1.X
outputs = session.run(f(placeholder), feed_dict={placeholder: input})
# TensorFlow 2.0
outputs = f(input)
```

这种方式可以让你在TF和Python中随意切换。我们希望用户可以充分的使用Python的表现力，但是方便的同时，在一些移动、c++、和JS中没有Python编译器时候，添加`@tf.function`这样的代码就需要用户重构许多代码。所以[AutoGraph](https://www.tensorflow.org/alpha/guide/autograph)将转换一个部分Python的子集到与TensorFlow等价的操作。

- `for`/`while` -> `tf.while_loop` (`break` and `continue` are supported)
- `if` -> `tf.cond`
- `for _ in dataset` -> `dataset.reduce`

AutoGraph支持任意的嵌套控制流，它可以尽可能的简明扼要实现许多ML程序比如序列模型，强化学习模型，定制的训练循环等等。

### TensorFlow2.0中的惯用方式

#### 将你的code重构到一些更加小的函数（这是tf.function装饰器带来的趋势）

在TensorFlow1.x中你习惯了“厨房水槽”方法。将所有的计算联合起来。然后选中Tensor，最后通过`session.run()`得到结果。在TensorFlow2.0中。用户应该将你们的代码重构到一些很小的函数中。只是在需要的时候调用。一般，没有必要将所有的函数都加上`tf.function`装饰器。你只需要给最高级的计算加上，直接调用即可。举例：一部的训练，或者你模型的一次前馈操作。

#### 使用Keras层和模型管理你的变量

Keras模型和层都提供了方便的`variables`和`trainable_variables`属性，以递归的方式收集所有依赖的变量。这使得变量管理变得十分简单。

对比：

```python
def dense(x, W, b):
  return tf.nn.sigmoid(tf.matmul(x, W) + b)

@tf.function
def multilayer_perceptron(x, w0, b0, w1, b1, w2, b2 ...):
  x = dense(x, w0, b0)
  x = dense(x, w1, b1)
  x = dense(x, w2, b2)
  ...

# You still have to manage w_i and b_i, and their shapes are defined far away from the code.
```

with the Keras version:

```python
# Each layer can be called, with a signature equivalent to linear(x)
layers = [tf.keras.layers.Dense(hidden_size, activation=tf.nn.sigmoid) for _ in range(n)]
perceptron = tf.keras.Sequential(layers)

# layers[3].trainable_variables => returns [w3, b3]
# perceptron.trainable_variables => returns [w0, b0, ...]
```

Keras layers/models 都是继承自`tf.train.Checkpointable`也被集成了`@tf.function`, 这样可以使得直接从Keras 对象获取到checkpoint或者导出SavedModels。你都没有必要使用Keras的`fit()`API去获取这些集成特性。

下面是一个迁移学习的例子，描述了怎么使用Keras方便的收集相关变量的子集。假设你在共享分支上训练一个多输入模型。

```python
trunk = tf.keras.Sequential([...]) # shared trunk
head1 = tf.keras.Sequential([...]) # input one
head2 = tf.keras.Sequential([...]) # input two

path1 = tf.keras.Sequential([trunk, head1])
path2 = tf.keras.Sequential([trunk, head2])

# Train on primary dataset
for x, y in main_dataset:
  with tf.GradientTape() as tape:
    prediction = path1(x)
    loss = loss_fn_head1(prediction, y)
  # Simultaneously optimize trunk and head1 weights.
  gradients = tape.gradients(loss, path1.trainable_variables)
  optimizer.apply_gradients(gradients, path1.trainable_variables)

# Fine-tune second head, reusing the trunk
for x, y in small_dataset:
  with tf.GradientTape() as tape:
    prediction = path2(x)
    loss = loss_fn_head2(prediction, y)
  # Only optimize head2 weights, not trunk weights
  gradients = tape.gradients(loss, head2.trainable_variables)
  optimizer.apply_gradients(gradients, head2.trainable_variables)

# You can publish just the trunk computation for other people to reuse.
tf.saved_model.save(trunk, output_path)
```

(上面的例子是在一个共享分支上训练两个模型，先使用一个主要的数据集，来训练trunk分支，然后再通过一个小的数据集来微调trunk分支，典型的迁移思想)。

#### 将`tf.data.Datasets`和`@tf.function`结合

当你迭代将训练数据填充到内存中，你可以随意使用Python规则的迭代。不然，你也可以使用`tf.data.Datasets`,这是一种最好的方式将训练数据从硬盘填充到内存中。Datasets是可迭代的不是迭代器[iterables (not iterators)](https://docs.python.org/3/glossary.html#term-iterable) 在Eager模型下，和Python迭代一样工作。你可以完全利用Dataset异步预装载/流特征通过`tf.funtion`来装饰你的code。它使用AutoGraph替换了Python迭代的等效图操作。

```python
@tf.function
def train(model, dataset, optimizer):
  for x, y in dataset:
    with tf.GradientTape() as tape:
      prediction = model(x)
      loss = loss_fn(prediction, y)
    gradients = tape.gradients(loss, model.trainable_variables)
    optimizer.apply_gradients(gradients, model.trainable_variables)
```

如果你使用Keras 的fit API，则你不用担心dataset的迭代：

```python
model.compile(optimizer=optimizer, loss=loss_fn)
model.fit(dataset)
```

#### 利用AutoGraph来代替Python的控制流

AutoGraph提供了一些方法，去转换数据依赖的依赖流进入图中，就像`tf.cond`和`tf.while_loop`

一个常用的数据依赖流出现的地方就是序列模型。`tf.keras.layers.RNN`封装了一个RNN单元，允许你静态或者动态的循环展开。为了示范，你可以重新实现如下动态展开：

```python
class DynamicRNN(tf.keras.Model):

  def __init__(self, rnn_cell):
    super(DynamicRNN, self).__init__(self)
    self.cell = rnn_cell

  def call(self, input_data):
    # [batch, time, features] -> [time, batch, features]
    input_data = tf.transpose(input_data, [1, 0, 2])
    outputs = tf.TensorArray(tf.float32, input_data.shape[0])
    state = self.cell.zero_state(input_data.shape[1], dtype=tf.float32)
    for i in tf.range(input_data.shape[0]):
      output, state = self.cell(input_data[i], state)
      outputs = outputs.write(i, output)
    return tf.transpose(outputs.stack(), [1, 0, 2]), state
```

有关更多详细的AutoGraph的功能，请看[the guide](https://www.tensorflow.org/alpha/guide/autograph)

#### 使用`tf.metrics`去集成data，随后使用`tf.summary`去log它

为了log summary，使用`tf.summary.(scalar | histogram | ...)`需要重定位它到上下文管理器（如果你忘记了上下文管理器，将什么都不会被记录下来）不想TF1.x, 所有的summaries都会被直接写入到writer中，没有单独的merge操作，和add_summary()调用。这意味着step值必须在调用点提供。

```python
summary_writer = tf.summary.create_file_writer('/tmp/summaries')
with summary_writer.as_default():
  tf.summary.scalar('loss', 0.1, step=42)
```

在log他们为summary前为了聚合数据，使用`tf.metrics`。Metrics是有状态的，它可以累计数据然后当你调用`.result()`时返回一个累计的结果。使用`.reset_states()`去清除累计值。

```python
def train(model, optimizer, dataset, log_freq=10):
  avg_loss = tf.keras.metrics.Mean(name='loss', dtype=tf.float32)
  for images, labels in dataset:
    loss = train_step(model, optimizer, images, labels)
    avg_loss.update_state(loss)
    if tf.equal(optimizer.iterations % log_freq, 0):
      tf.summary.scalar('loss', avg_loss.result(), step=optimizer.iterations)
      avg_loss.reset_states()

def test(model, test_x, test_y, step_num):
  loss = loss_fn(model(test_x), test_y)
  tf.summary.scalar('loss', loss, step=step_num)

train_summary_writer = tf.summary.create_file_writer('/tmp/summaries/train')
test_summary_writer = tf.summary.create_file_writer('/tmp/summaries/test')

with train_summary_writer.as_default():
  train(model, optimizer, dataset)

with test_summary_writer.as_default():
  test(model, test_x, test_y, optimizer.iterations)
```

在log目录通过Tensorboard可视化生成的结果： `tensorboard --logdir /tmp/summaries`.

QQ：329804334

Mail：mizeshuang@gmail.com