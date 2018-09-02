---
title: TensorFlow data模块详解
tags: TensorFlow
category: TensorFlow
mathjax: true
author: Milittle
thumbnail: 'https://s1.ax1x.com/2018/03/18/9oakkQ.png'
abbrlink: cd5ba0c4
date: 2018-09-02 22:31:21
---

[TOC]

大家好，本周给大家讲解tensorflow的data模块的详细信息，主要涉及到tf.data.Dataset对象的一些操作，和tf.data.Iterator以及后续使用tf.data API实现Numpy数据的读取、TFRecord数据格式的读取、文本文件的读取以及CSV文件的读取，最后应用批数据来进行训练和验证工作。

# **tf.data API的基本特性：**

1. 这个**API**让你可以从简单的，重复使用的片段构造出复杂的输入流水线。
2. 对于图片模型的管道可以从一个分布式文件系统来聚集数据，而且可以实现每张图片的随机抖动。
3. 可以实现随机选取图片的合并到一个**batch**，然后训练。
4. 对于文本模型的管道可能会涉及到从原文本中抽取符号。
5. 转换他们到一个嵌入标识的查询表。
6. 批处理不同长度的序列。
7. 这个**API**可以很轻易的实现大数量数据的处理。
8. 还可以处理不同的数据格式，还可以实现负责的转换。

对于**TensorFlow tf.data API**介绍了两种抽象：

1. `tf.data.Dataset`代表了元素的序列，每个元素包括一个或者多个**Tensor**对象。举例，在输入图片的流水线中，一个元素可能是一个单独的训练样本，是一对**tensors**，分别代表的是图片数据以及label。下面有两种不同的方式来创建一个**dataset**。

- 创建一个**source**(比如，`Dataset.from_tensor_slices()`) 从一个或者多个`tf.Tensor`对象构造。
- 应用一个**transformation**(比如，`Dataset.batch()`) 从一个或者多个`tf.data.Dataset`对象构造。

1. 一个`tf.data.Iterator`提供了从一个**dataset**中提取元素的主要方法。当`Iterator.get_next()`方法执行以后，返回的就是**Dataset**的下一个元素。典型的作为输入流水线和你的模型之间的接口。最简单的迭代器是一次性的迭代器，它和一个**Dataset**联系，然后可以迭代一次。对于更加复杂的使用，`Iterator.initializer`操作开启你重新初始化，和参数化一个**iterator**和不同的**datasets**关联。这样，你就可以在同一个程序中，实现训练数据和验证集数据的多次迭代。

# **基本的机制**

这一部分指引介绍的是创建不同的**Dataset**和**Iterator**，还有怎么从这些对象中抽取数据。

开始一个输入流水线，你必须定义一个*source*，举一个例子：

1. 为了从内存的tensors构建一个`Dataset`，你可以使用下面的方法：

```python
tf.data.Dataset.from_tensors() or tf.data.Dataset.from_tensor_slices()
```

1. 如果你的输入数据在硬盘上，那么推荐你是用TFRecord格式，你可以构建一个`tf.data.TFRecordDataset`

一旦你有了一个`Dataset`对象，你可以通过链式方法调用来实现新的Dataset的转换。举一个例子，比如你可以应用一个预元素处理方法`Dataset.map()`实现每个元素的应用操作。多个元素的转换的方法`Dataset.batch()`。详细情况你可以查看`tf.data.Dataset`完整的转换方法。

从`Dataset`中消耗值最传统的方式是通过`iterator`对象。这个对象提供了一次访问dataset中的一个元素。（举例，通过调用 `Dataset.make_one_shot_iterator()`）。一个`tf.data.Iterator`对象提供了两个操作：1. `Iterator.initializer`,这个方法可以实现（重）初始化**iterator**的状态。2. `Iterator.get_next()`,这个方法返回的是`tf.Tensor`对象，这个对象和下一个元素相关。依赖你使用的情况，你可能选择不同类型的迭代器，选项概述如下。

### **数据集结构**

1. 用`Dataset.output_types`表示数据集数据类型
2. 用`Dataset.output_shapes`表示数据集数据大小

**dataset**包含的每个元素都有相同的结构。一个元素包括一个或者多个`tf.Tensor`对象，被称为组件（这些tensor对象就是组件）。每个组件有一个`tf.DType`代表的是元素中tensor的类型。一个`tf.TensorShape`代表的是每个元素静态**shape**（有可能是部分指定）。`Dataset.output_types`和`Dataset.output_shapes`属性允许你可以检查推断**dataset**元素每个组件的**types**和**shapes**。这有可能是单个**tensor**，也有可能是**tuple tensors**，或者是**tensors**的嵌套。举个例子：

```python
dataset1 = tf.data.Dataset.from_tensor_slices(tf.random_uniform([4, 10]))
print(dataset1.output_types)  # ==> "tf.float32"
print(dataset1.output_shapes)  # ==> "(10,)"

dataset2 = tf.data.Dataset.from_tensor_slices(
   (tf.random_uniform([4]),
    tf.random_uniform([4, 100], maxval=100, dtype=tf.int32)))
print(dataset2.output_types)  # ==> "(tf.float32, tf.int32)"
print(dataset2.output_shapes)  # ==> "((), (100,))"

dataset3 = tf.data.Dataset.zip((dataset1, dataset2))
print(dataset3.output_types)  # ==> (tf.float32, (tf.float32, tf.int32))
print(dataset3.output_shapes)  # ==> "(10, ((), (100,)))"
```

对一个元素的组件起一个名字，通常是很便捷的事情。而且起名字以后打印相关信息会变得更加直观。举例，比如如果你代表不同的训练数据。另外，**tuple**你可以使用`collections.namedtuple`或者一个**dictionary**映射从字符串到tensors来表示一个单个元素的**Dataset**。

```python
dataset = tf.data.Dataset.from_tensor_slices(
   {"a": tf.random_uniform([4]),
    "b": tf.random_uniform([4, 100], maxval=100, dtype=tf.int32)})
print(dataset.output_types)  # ==> "{'a': tf.float32, 'b': tf.int32}"
print(dataset.output_shapes)  # ==> "{'a': (), 'b': (100,)}"
```

`Dataset`转换支持任意结构的**datasets**，当你使用`Dataset.map()`, `Dataset.flat_map()`, 和`Dataset.filter()`转换的时候，为了给每个元素应用一个函数，元素结构决定了函数的参数：

```python
dataset1 = dataset1.map(lambda x: ...)

dataset2 = dataset2.flat_map(lambda x, y: ...)

# Note: Argument destructuring is not available in Python 3.
dataset3 = dataset3.filter(lambda x, (y, z): ...)
```

### **创建一个iterator**

一旦你构建了一个**Dataset**去表示你的输入数据，下一步就是创建一个**Iterator**从**Dataset**访问你的数据。**tf.data API**当前支持下面的迭代器，来提升复杂的等级。

- **one-shot**
- **initializable**
- **reinitializable**
- **feedable**

**one-shot**迭代器是最简单的迭代器，它只支持，从数据集中迭代读取一次数据，不需要显示的初始化。一次性迭代器基本上可以处理所有基于队列输入管道的支持。但是这不支持参数化。使用**Dataset.range()**的例子：

```python
dataset = tf.data.Dataset.range(100)
iterator = dataset.make_one_shot_iterator()
next_element = iterator.get_next()

for i in range(100):
    value = sess.run(next_element)
    assert i == value
```

**Note:**  Currently, one-shot iterators are the only type that is easily usable with an `Estimator`.（当前一次性迭代器是唯一一个和Estimator共同使用的类型）

一个**initializable**迭代器，在使用之前要求你显示运行`iterator.initializer`操作。对于这个不方便引入的同时，它可以让你参数化一个已经定义好的**Dataset**，当你初始化一个**iterator**可以使用一个或者多个`tf.placeholder()` tensors来输入数据。就`Dataset.range()`的例子。

```python
max_value = tf.placeholder(tf.int64, shape=[])
dataset = tf.data.Dataset.range(max_value)
iterator = dataset.make_initializable_iterator()
next_element = iterator.get_next()

# Initialize an iterator over a dataset with 10 elements.
# 必须显示run iterator.initializer
sess.run(iterator.initializer, feed_dict={max_value: 10})
for i in range(10):
    value = sess.run(next_element)
    assert i == value

# Initialize the same iterator over a dataset with 100 elements.
sess.run(iterator.initializer, feed_dict={max_value: 100})
for i in range(100):
    value = sess.run(next_element)
    assert i == value
```

一个**reinitializable**迭代器可以从多个不同的**Dataset**对象中实现初始化。举例，你可能有一个训练输入管道使用随机抖动来增强图片为了实现网络的泛化性能。一个验证集输入管道验证预测是未修改的数据。这些管道就是使用不同的**Dataset**对象的典型示例，但是有相同的结构。（比如，相同的类型，每个组件的兼容shapes）

```python
# Define training and validation datasets with the same structure.
#　定义训练和验证集　具有相同的结构
training_dataset = tf.data.Dataset.range(100).map(
    lambda x: x + tf.random_uniform([], -10, 10, tf.int64))
validation_dataset = tf.data.Dataset.range(50)

# A reinitializable iterator is defined by its structure. We could use the
# `output_types` and `output_shapes` properties of either `training_dataset`
# or `validation_dataset` here, because they are compatible.
'''
一个reinitializable iterator是由其结构来定义的，我们可以使用（training或者validation）的output_types和output_shapes属性，因为彼此都是兼容的。
'''
iterator = tf.data.Iterator.from_structure(training_dataset.output_types,
                                           training_dataset.output_shapes)
next_element = iterator.get_next()

training_init_op = iterator.make_initializer(training_dataset)
validation_init_op = iterator.make_initializer(validation_dataset)

# Run 20 epochs in which the training dataset is traversed, followed by the
# validation dataset.
for _ in range(20):
  　　# Initialize an iterator over the training dataset.
  　　sess.run(training_init_op)
  　　for _ in range(100):
    　　　　sess.run(next_element)

  　　# Initialize an iterator over the validation dataset.
  　　sess.run(validation_init_op)
  　　for _ in range(50):
    　　　　sess.run(next_element)
```

一个**feedable**迭代器，可以和`tf.placeholder`一起去选择Iterator为了使用每一次的`tf.Session.run()`,通过类似于feed_dict的机制来实现这一功能。它提供了和**reinitializable**迭代器相同的功能，但是这个不需要你在切换迭代器的时候初始化迭代器。举例，使用相同的训练和验证的示例，你可以使用`tf.data.Iterator.from_string_handle`去定义一个**feedable iterator**，然后允许你切换两个数据集：

```python
# Define training and validation datasets with the same structure.
training_dataset = tf.data.Dataset.range(100).map(
    lambda x: x + tf.random_uniform([], -10, 10, tf.int64)).repeat()
validation_dataset = tf.data.Dataset.range(50)

# A feedable iterator is defined by a handle placeholder and its structure. We
# could use the `output_types` and `output_shapes` properties of either
# `training_dataset` or `validation_dataset` here, because they have
# identical structure.
handle = tf.placeholder(tf.string, shape=[])
iterator = tf.data.Iterator.from_string_handle(
    handle, training_dataset.output_types, training_dataset.output_shapes)
next_element = iterator.get_next()

# You can use feedable iterators with a variety of different kinds of iterator
# (such as one-shot and initializable iterators).
'''
你可以在feedable iterator中使用不同种类的iterator，比如one-shot和initializable iterator
'''
training_iterator = training_dataset.make_one_shot_iterator()
validation_iterator = validation_dataset.make_initializable_iterator()

# The `Iterator.string_handle()` method returns a tensor that can be evaluated
# and used to feed the `handle` placeholder.
'''
Iterator.string_handle()方法可以实现handle placeholder的字符串抽取
'''
training_handle = sess.run(training_iterator.string_handle())
validation_handle = sess.run(validation_iterator.string_handle())

# Loop forever, alternating between training and validation.
while True:
    # Run 200 steps using the training dataset. Note that the training dataset is
    # infinite, and we resume from where we left off in the previous `while` loop
    # iteration.
    for _ in range(200):
        sess.run(next_element, feed_dict={handle: training_handle})

    # Run one pass over the validation dataset.
    sess.run(validation_iterator.initializer)
    for _ in range(50):
        sess.run(next_element, feed_dict={handle: validation_handle})
```

### **从迭代器中消耗值**

```python
dataset = tf.data.Dataset.range(5)
iterator = dataset.make_initializable_iterator()
next_element = iterator.get_next()

# Typically `result` will be the output of a model, or an optimizer's
# training operation.
result = tf.add(next_element, next_element)

sess.run(iterator.initializer)
print(sess.run(result))  # ==> "0"
print(sess.run(result))  # ==> "2"
print(sess.run(result))  # ==> "4"
print(sess.run(result))  # ==> "6"
print(sess.run(result))  # ==> "8"
try:
  sess.run(result)
except tf.errors.OutOfRangeError:
  print("End of dataset")  # ==> "End of dataset"
```

一个正常模式就是捕获异常：

```python
sess.run(iterator.initializer)
while True:
  try:
    sess.run(result)
  except tf.errors.OutOfRangeError:
    break
```

如果**dataset**的每个元素有一个嵌套结构的话，**Iterator.get_next()**方法返回的是一个或者多个对象，也具有相同的嵌套结构：

```python
dataset1 = tf.data.Dataset.from_tensor_slices(tf.random_uniform([4, 10]))
dataset2 = tf.data.Dataset.from_tensor_slices((tf.random_uniform([4]), tf.random_uniform([4, 100])))
dataset3 = tf.data.Dataset.zip((dataset1, dataset2))

iterator = dataset3.make_initializable_iterator()

sess.run(iterator.initializer)
next1, (next2, next3) = iterator.get_next()
```

注意 next1， next2，next3。都是被相同的操作生成的。（通过Iterator.get_next()创建）。因此，评估这些tensors的话，就可以加强迭代所有的组件。一个典型的iterator消费器，将包括所有组件的单一表达式。

### **保存迭代器的状态**

`tf.contrib.data.make_saveable_from_iterator`函数从一个迭代器中创建一个`SaveableObject`，这个对象可以实现当前迭代器的保存和恢复（有效的，就是整个输入管道）。一个saveable对象创建以后可以添加到`tf.train.Saver`变量list或者是`tf.GraphKeys.SAVEABLE_OBJECTS`集合里面，为了保存和恢复，这种方式和`tf.Variable`机制差不多。这涉及到了Saveing and Restoring的细节。怎么保存和恢复变量。

```python
# Create saveable object from iterator.
saveable = tf.contrib.data.make_saveable_from_iterator(iterator)

# Save the iterator state by adding it to the saveable objects collection.
tf.add_to_collection(tf.GraphKeys.SAVEABLE_OBJECTS, saveable)
saver = tf.train.Saver()

with tf.Session() as sess:

    if should_checkpoint:
        saver.save(path_to_checkpoint)

# Restore the iterator state.
with tf.Session() as sess:
    saver.restore(sess, path_to_checkpoint)
```

# **读取输入数据**

### **消耗NumPy数组数据**

如果你的数据适合在内存中管理，最简单的方式就是将原始数据转换为`tf.Tensor`对象，然后使用`Dataset.from_tenosr_slices`来创建一个**Dataset**

```python
# Load the training data into two NumPy arrays, for example using `np.load()`.
# 加载训练数据到两个Numpy数组，使用np.load()示例
with np.load("/var/data/training_data.npy") as data:
    features = data["features"]
    labels = data["labels"]

# Assume that each row of `features` corresponds to the same row as `labels`.
# 假定每一行的features和labels所对应。
assert features.shape[0] == labels.shape[0]

dataset = tf.data.Dataset.from_tensor_slices((features, labels))
```

值得注意的是，上面的代码片段将嵌入**features**和**labels**数组到TensorFlow图中的tf.constant()操作。这对于效地数据集是可取的，但是浪费内存---因为数组的内容将会被拷贝多次，---对于`tf.GraphDef`协议缓存会达到2GB的缓存限制。

可选的，你可以通过`tf.placeholder()`tensors来定义一个**Dataset**，当你从一个**Dataset**初始化一个**Iterator**以后，*feed* Numpy数组。

```python
# Load the training data into two NumPy arrays, for example using `np.load()`.
# 加载训练数据到两个Numpy数组，使用np.load()示例
with np.load("/var/data/training_data.npy") as data:
    features = data["features"]
    labels = data["labels"]

# Assume that each row of `features` corresponds to the same row as `labels`.
# 假定每一行的features和labels所对应。
assert features.shape[0] == labels.shape[0]

features_placeholder = tf.placeholder(features.dtype, features.shape)
labels_placeholder = tf.placeholder(labels.dtype, labels.shape)

dataset = tf.data.Dataset.from_tensor_slices((features_placeholder, labels_placeholder))
# [Other transformations on `dataset`...]
dataset = ...
iterator = dataset.make_initializable_iterator()

sess.run(iterator.initializer, feed_dict={features_placeholder: features,
                                          labels_placeholder: labels})
```

### **消耗TFRecord数据**

**tf.data API**支持很多种文件格式，因此，你可以处理很大的数据集而不是直接加载在内存中。举例，TFRecord文件格式是一个简单的面向记录的二进制文件格式，许多TensorFlow的应用都使用它来作为训练数据。`tf.data.TFRecordDataset`类可以使你流化一个或者多个TFRecord文件作为你输入管道的一部分。

```python
# Creates a dataset that reads all of the examples from two files.
# 从两个文件中读取所有样例，然后创建一个dataset
filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
dataset = tf.data.TFRecordDataset(filenames)
```

给**TFRecordDataset**初始化的**filenames**参数也可以是一个**strings**，或者是一个`tf.Tensor`的**strings**对象。因此，如果你有两个集合，一个是训练集，一个是测试集，那么你可以使用一个`tf.placeholder(tf.string)`去代表一个**filenames**，然后从一个合适的**filenames**来初始化一个迭代器。

```python
filenames = tf.placeholder(tf.string, shape=[None])
dataset = tf.data.TFRecordDataset(filenames)
dataset = dataset.map(...)  # Parse the record into tensors. 解析record到tensors
dataset = dataset.repeat()  # Repeat the input indefinitely. 重复无限输入数据
dataset = dataset.batch(32)
iterator = dataset.make_initializable_iterator()

# You can feed the initializer with the appropriate filenames for the current
# phase of execution, e.g. training vs. validation.
'''
你可以给不同阶段的执行初始化不同的filenames，比如训练和验证
'''

# Initialize `iterator` with training data.
# 用训练数据来初始化iterator
training_filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
sess.run(iterator.initializer, feed_dict={filenames: training_filenames})

# Initialize `iterator` with validation data.
# 用验证数据来初始化iterator
validation_filenames = ["/var/data/validation1.tfrecord", ...]
sess.run(iterator.initializer, feed_dict={filenames: validation_filenames})
```

### **消耗text数据**

许多数据集是分布在一个或者多个text文件中的。`tf.data.TextLineDataset`提供了从一个或者多个text文件中抽取行数据的简单方法。给一个或者多个filenames，一个**TextLineDataset**将会在这些文件的每一行都生成一个**string-value**元素，像**TFRecordDataset**，**TextLineDataset**接受filenames作为`tf.Tensor`,所以你也可以通过`tf.placeholder(tf.string)`参数化**filenames**

```python
filenames = ["/var/data/file1.txt", "/var/data/file2.txt"]
dataset = tf.data.TextLineDataset(filenames)
```

默认的，一个**TextLineDataset**产出每一个文件的每一行，这有可能不是我们想要的，举例，如果文件的开始头，或者注释。这些行我们可以通过`Dataset.skip()`和`Dataset.filter()`进行转换。为了给每一个文件分别应用转换，我们可以使用`Dataset.flat_map()`为每一个文件创建一个嵌套的**Dataset**。

```python
filenames = ["/var/data/file1.txt", "/var/data/file2.txt"]

dataset = tf.data.Dataset.from_tensor_slices(filenames)

# Use `Dataset.flat_map()` to transform each file as a separate nested dataset,
# and then concatenate their contents sequentially into a single "flat" dataset.
# * Skip the first line (header row).
# * Filter out lines beginning with "#" (comments).

'''
使用 Dataset.flat_map()去转换每一个文件，分离出一个嵌套的dataset
然后使用级联他们的内容序列化为一个单个的flat dataset
* skip第一行（header row）
* filter掉行开始是#注释符号的行
'''
dataset = dataset.flat_map(
    lambda filename: (
        tf.data.TextLineDataset(filename)
        .skip(1)
        .filter(lambda line: tf.not_equal(tf.substr(line, 0, 1), "#"))))
```

### **消耗CSV数据**

CSV文件格式是很流行的一种格式，它存储的格式是通过tab来间隔数据，在一个无格式的文本文件中。`tf.contrib.data.CsvDataset`类提供了从一个或者多个CSV文件中提取数据的方式。给一个或者多个**filenames**或者一个默认的list，一个**CsvDataset**将会生成一个tuple的元素，元素类型和每一个CSV记录默认提供的类型是相关的。像**TFRecordDataset**和**TextLineDataset**一样，**CsvDataset**接受**filenames**作为一个`tf.Tenosr`,因此你一样可以使用`tf.placeholder(tf.string)`实现参数化。

```python
# Creates a dataset that reads all of the records from two CSV files, each with
# eight float columns
# 从两个CSV文件读取所有的记录来创建一个dataset对象
# 每一个csv文件中有八列float数据
filenames = ["/var/data/file1.csv", "/var/data/file2.csv"]
record_defaults = [tf.float32] * 8   # Eight required float columns
dataset = tf.contrib.data.CsvDataset(filenames, record_defaults)
```

如果有一些列是空的，你可以提供默认的值，代替其类型

```python
# Creates a dataset that reads all of the records from two CSV files, each with
# four float columns which may have missing values
# 从两个CSV文件读取所有的记录来创建一个dataset对象
# 
record_defaults = [[0.0]] * 8
dataset = tf.contrib.data.CsvDataset(filenames, record_defaults)
```

默认的，一个**CsvDataset**产出每一个文件每一行的每列，如果我们不想这么多数据怎么办，举例，如果我们不想要开头，还有一些列我们也不想要。这些行和字段都可以使用**header**和**select_cols**参数来进行忽略。

```python
# Creates a dataset that reads all of the records from two CSV files with
# headers, extracting float data from columns 2 and 4.
# 从两个CSV文件读取所有的记录来创建一个dataset对象
# 头忽略，然后只要第二列和第四列
record_defaults = [[0.0]] * 2  # Only provide defaults for the selected columns
dataset = tf.contrib.data.CsvDataset(filenames, record_defaults, header=True, select_cols=[2,4])
```

# **使用Dataset.map()来预处理数据**

`Dataset.map(f)`对输入dataset的每一个元素应用函数**f**来生成新的dataset。这是基于**map**函数的，这是在函数式编程语言中通常会应用一个lists或者其他结构的映射方法。函数**f**输入一个`tf.Tenosr`对象代表一个输入的单个元素。然后返回一个`tf.Tensor`对象代表新dataset的单个元素。它的实现都是基于TensorFlow的标准转换操作。

这部分覆盖了一些怎么使用`Dataset.map()`的例子

### **解析`tf.Example`协议缓存信息**

许多输入管道从一个**TFRecord-格式文件**（写，举例，使用`tf.python_io.TFRecordWriter`）提取tf.train.Example协议缓存信息。每一个`tf.train.Example`记录包含一个或者多个features，输入管道可以转换这些features到tensors。

```python
# Transforms a scalar string `example_proto` into a pair of a scalar string and
# a scalar integer, representing an image and its label, respectively.
def _parse_function(example_proto):
    features = {"image": tf.FixedLenFeature((), tf.string, default_value=""),
                "label": tf.FixedLenFeature((), tf.int32, default_value=0)}
    parsed_features = tf.parse_single_example(example_proto, features)
    return parsed_features["image"], parsed_features["label"]

# Creates a dataset that reads all of the examples from two files, and extracts
# the image and label features.
filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
dataset = tf.data.TFRecordDataset(filenames)
dataset = dataset.map(_parse_function)
```

### **编码图片数据然后resize**

当训练一个用真实世界图片数据的神经网络的时候，将图片的大写resize到一个通用的size是很有必要的。因此必须批处理为固定的大小。

```python
# Reads an image from a file, decodes it into a dense tensor, and resizes it
# to a fixed shape.
def _parse_function(filename, label):
  image_string = tf.read_file(filename)
  image_decoded = tf.image.decode_jpeg(image_string)
  image_resized = tf.image.resize_images(image_decoded, [28, 28])
  return image_resized, label

# A vector of filenames.
filenames = tf.constant(["/var/data/image1.jpg", "/var/data/image2.jpg", ...])

# `labels[i]` is the label for the image in `filenames[i].
labels = tf.constant([0, 37, ...])

dataset = tf.data.Dataset.from_tensor_slices((filenames, labels))
dataset = dataset.map(_parse_function)
```

### **使用tf.py_func()应用任意的Python逻辑**

为性能的原因，我们鼓励你使用TensorFlow操作来预处理你的数据。然后，有时候当解析你的数据时候调用外部的Python库对你来说是有用的。为了实现这样，在Dataset.map()中调用`tf.py_func()`来实现转换。

```python
import cv2

# Use a custom OpenCV function to read the image, instead of the standard
# TensorFlow `tf.read_file()` operation.
def _read_py_function(filename, label):
    image_decoded = cv2.imread(filename.decode(), cv2.IMREAD_GRAYSCALE)
    return image_decoded, label

# Use standard TensorFlow operations to resize the image to a fixed shape.
def _resize_function(image_decoded, label):
    image_decoded.set_shape([None, None, None])
    image_resized = tf.image.resize_images(image_decoded, [28, 28])
    return image_resized, label

filenames = ["/var/data/image1.jpg", "/var/data/image2.jpg", ...]
labels = [0, 37, 29, 1, ...]

dataset = tf.data.Dataset.from_tensor_slices((filenames, labels))
dataset = dataset.map(
    lambda filename, label: tuple(tf.py_func(
        _read_py_function, [filename, label], [tf.uint8, label.dtype])))
dataset = dataset.map(_resize_function)
```

# **批处理dataset元素**

### **简单的batching**

批处理堆积n个连续的dataset元素到一个单个元素。`Dataset.batch()`就这这么做的。与之相同的约束是`tf.stack()`操作，应用到每一个元素的每一个组件，比如，对于每一个组件**i**，所有的元素有相同shape的tensor。

```python
inc_dataset = tf.data.Dataset.range(100)
dec_dataset = tf.data.Dataset.range(0, -100, -1)
dataset = tf.data.Dataset.zip((inc_dataset, dec_dataset))
batched_dataset = dataset.batch(4)

iterator = batched_dataset.make_one_shot_iterator()
next_element = iterator.get_next()

print(sess.run(next_element))  # ==> ([0, 1, 2,   3],   [ 0, -1,  -2,  -3])
print(sess.run(next_element))  # ==> ([4, 5, 6,   7],   [-4, -5,  -6,  -7])
print(sess.run(next_element))  # ==> ([8, 9, 10, 11],   [-8, -9, -10, -11])
```

### **给tensors批量添加padding**

关于上面的tensor来讲，有相同的size，然而，许多的模型（比如，序列模型）可以有变化的尺寸。（比如，不同的序列长度）为了处理这种情况，`Dataset.padded_batch()`转换使得你可以处理不同shapes的tensors，通过指定一个或者多个你想要padding的维度。

```python
dataset = tf.data.Dataset.range(100)
dataset = dataset.map(lambda x: tf.fill([tf.cast(x, tf.int32)], x))
dataset = dataset.padded_batch(4, padded_shapes=[None])

iterator = dataset.make_one_shot_iterator()
next_element = iterator.get_next()

print(sess.run(next_element))  # ==> [[0, 0, 0], [1, 0, 0], [2, 2, 0], [3, 3, 3]]
print(sess.run(next_element))  # ==> [[4, 4, 4, 4, 0, 0, 0],
                               #      [5, 5, 5, 5, 5, 0, 0],
                               #      [6, 6, 6, 6, 6, 6, 0],
                               #      [7, 7, 7, 7, 7, 7, 7]]
```

`Dataset.padded_batch()`转换允许你去给每一个组件的每一个维度设置不同的padding，而且它可以实现变长（通过指定None的方式）或者固定长度。去复写padding值，也是可以的，默认的值一般是0。

# **训练流程**

**tf.data API**提供两种处理相同书的多次epochs。

最简单的放还是就是使用`Dataset.repeat()`方法的转换来实现多次的epochs。举例，创建一个dataset然后重复10 epochs。

```python
filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
dataset = tf.data.TFRecordDataset(filenames)
dataset = dataset.map(...)
dataset = dataset.repeat(10)
dataset = dataset.batch(32)
```

应用`Dataset.repeat()`时候，如果没有参数，将会无限重复输入。`Dataset.repeat()`转换符号链接它的参数没有将一个epoch的结束和下一个epoch的开始做信号处理。

如果你想接受一个epoch结束的信号，你可以写一个训练循环，然后在数据集的结束捕获`tf.errors.OutOfRangeError`。在这个点上你可以为epoch收集一些统计信息。

```python
filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
dataset = tf.data.TFRecordDataset(filenames)
dataset = dataset.map(...)
dataset = dataset.batch(32)
iterator = dataset.make_initializable_iterator()
next_element = iterator.get_next()

# Compute for 100 epochs.
for _ in range(100):
    sess.run(iterator.initializer)
    while True:
      try:
          sess.run(next_element)
      except tf.errors.OutOfRangeError:
          break

  # [Perform end-of-epoch calculations here.]
```

### **随机打乱输入数据**

`Dataset.shuffle()`转换时随机打乱输入数据。类似于`tf.RandomShoffleQueue`的算法。它包含固定的缓存，和选择下一个元素的正太随机分布的缓存。

```python
filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
dataset = tf.data.TFRecordDataset(filenames)
dataset = dataset.map(...)
dataset = dataset.shuffle(buffer_size=10000)
dataset = dataset.batch(32)
dataset = dataset.repeat()
```

### **使用高级的APIs**

`tf.train.MonitoredTrainingSession` API 简化了许多TensorFlow的分布式设置。**MonitoredTrainingSession**使用`tf.errors.OutOfRangeError`去信号化训练是否完成，因此使用**tf.data API**,我们推荐使用`Dataset.make_one_shor_iterator()`,举例：

```python
filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
dataset = tf.data.TFRecordDataset(filenames)
dataset = dataset.map(...)
dataset = dataset.shuffle(buffer_size=10000)
dataset = dataset.batch(32)
dataset = dataset.repeat(num_epochs)
iterator = dataset.make_one_shot_iterator()

next_example, next_label = iterator.get_next()
loss = model_function(next_example, next_label)

training_op = tf.train.AdagradOptimizer(...).minimize(loss)

with tf.train.MonitoredTrainingSession(...) as sess:
    while not sess.should_stop():
        sess.run(training_op)
```

为了在一个`tf.estimator.Estimator`的**input_fn**中使用**Dataset**，我们也推荐使用`Dataset. make_one_shot_iterator()`，举例：

```python
def dataset_input_fn():
    filenames = ["/var/data/file1.tfrecord", "/var/data/file2.tfrecord"]
    dataset = tf.data.TFRecordDataset(filenames)

    # Use `tf.parse_single_example()` to extract data from a `tf.Example`
    # protocol buffer, and perform any additional per-record preprocessing.
    def parser(record):
        keys_to_features = {
          "image_data": tf.FixedLenFeature((), tf.string, default_value=""),
          "date_time": tf.FixedLenFeature((), tf.int64, default_value=""),
          "label": tf.FixedLenFeature((), tf.int64,
                                    default_value=tf.zeros([], dtype=tf.int64)),
      }
        parsed = tf.parse_single_example(record, keys_to_features)

        # Perform additional preprocessing on the parsed data.
        image = tf.image.decode_jpeg(parsed["image_data"])
        image = tf.reshape(image, [299, 299, 1])
        label = tf.cast(parsed["label"], tf.int32)

        return {"image_data": image, "date_time": parsed["date_time"]}, label

    # Use `Dataset.map()` to build a pair of a feature dictionary and a label
    # tensor for each example.
    dataset = dataset.map(parser)
    dataset = dataset.shuffle(buffer_size=10000)
    dataset = dataset.batch(32)
    dataset = dataset.repeat(num_epochs)
    iterator = dataset.make_one_shot_iterator()

    # `features` is a dictionary in which each value is a batch of values for
    # that feature; `labels` is a batch of labels.
    features, labels = iterator.get_next()
    return features, labels
```

这次的tf.data API就讲到这里，本文章翻译自tensorflow官方网站，有一些是用自己的话描述出来，但是难免会有翻译不恰当，或者有误的地方，如果发现错误，可以直接发我邮箱或者加我qq

qq:  329804334

email:  air@weaf.top

参考文献：

> https://www.tensorflow.org/guide/datasets