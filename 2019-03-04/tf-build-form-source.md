---
title: 从源码构建TensorFlow2.0
category: TensorFlow
tags: TensorFlow
mathjax: true
thumbnail: 'https://s1.ax1x.com/2018/03/18/9oakkQ.png'
author: Milittle
abbrlink: c05ce623
date: 2019-03-05 19:07:57
---

### 从源码安装TensorFlow（未发布版本2.0）

#### 1. 今天想尝一尝tf2.0的鲜，就尝试去从源码构建TensorFlow2.0版本的GPU wheel包。

#### 2.首先你需要准备一些什么：

- Ubuntu16.04
- bazel(这个安装见https://docs.bazel.build/versions/master/install-ubuntu.html)（so easy）
- gcc4.8， g++4.8
- python3.6（包括six，numpy， wheel，mock，keras_applications, keras_preprocessing）具体的依赖版本可以在{tensorflow}/tensorflow/tools/pip_package/setup.py里面查看。

#### 3. 安装构建TensorFlow依赖软件

安装gcc和g++（TensorFlow使用gcc4.x构建，所以以下的gcc和g++版本都是4.8）

```shell
# install gcc and g++
$ sudo apt-get update
$ sudo apt-get install gcc-4.8 g++4.8
$ sudo rm /usr/bin/gcc 
$ sudo rm /usr/bin/g++
$ sudo ln -s /usr/bin/gcc-4.8 /usr/bin/gcc
$ sudo ln -s /usr/bin/g++-4.8 /usr/bin/g++
```

安装python,你可以选择一个已经有的python环境，或者新建一个python环境，将以上描述的包依此安装。TensorFlow2.0版本依赖的python包版本为：

```
REQUIRED_PACKAGES = [
    'absl-py >= 0.7.0',
    'astor >= 0.6.0',
    'gast >= 0.2.0',
    'google_pasta >= 0.1.2',
    'keras_applications >= 1.0.6',
    'keras_preprocessing >= 1.0.5',
    'numpy >= 1.14.5, < 2.0',
    'six >= 1.10.0',
    'protobuf >= 3.6.1',
    'tb-nightly >= 1.14.0a20190301, < 1.14.0a20190302',
    'tf-estimator-nightly >= 1.14.0.dev2019030115, < 1.14.0.dev2019030116',
    'termcolor >= 1.1.0',
]
```

#### 4.克隆TensorFlow库。

首先，你需要git，然后根据以下命令clone，然后构建TensorFlow项目

```shell
$ git clone https://github.com/tensorflow/tensorflow.git
$ cd tensorflow

$ git checkout branch_name  # 这里是r2.0
$ bazel test -c opt -- //tensorflow/... -//tensorflow/compiler/... -//tensorflow/lite/...
```

不出意外，测试结束以后，进行项目构建

####  5. 配置项目

```shell
$ ./configure
```

以下是项目配置的一个过程：

```shell
Please specify the location of python.[Default is /home/milittle/anaconda3/bin/python] # 注意这个就是你刚才安装的python环境
Found possible Python library paths:
/home/milittle/anaconda/lib/python3.6/site-packages
Please input the desired Python library path to use. Default is [/home/milittle/anaconda3/lib/python3.6/site-packages]
Do you wish to build TensorFlow with XLA JIT support? [Y/n]:Y
Do you wish to build TensorFlow with OpenCL SUCL support?[y/N]:N
Do you wish to build TensorFlow with ROCm support?[y/N]:N
Do you wish to build TensorFlow with CUDA support?[y/N]:y
Please specify the CUDA SDK version you want to use.[Leave empty to default to CUDA 10.0]:9.0
Please specify the location where CUDA 9.0 toolkit is installed. Refer to README.md for more details.[Default is /usr/local/cuda]:/home/milittle/cuda-9.0
Please specify the cuDNN version you want to use.[Leave empty to default to cuDNN 7]:7.1.4
Please specifu the location where cuDNN 7 library is installed.Refer to README.md for more details.[Default is /home/milittle/cuda-9.0]:
Do you wish to build TensorFlow with TensorRT support?[y/N]:N
Please specify the locally installed NCCL version you want to use. [Default is to use hppts://github.com/nvidia/nccl]:
Please specify a list of comma-separated Cuda compute capabilities you want to build with.
You can find the compute capability of your device at: https://developer.nvidia.com/cuda-gpus.
Please note that each additional compute capability significantly increases your
build time and binary size. [Default is: 6.1, 6.1, 6.1, 6.1]:6.1
Do you want to use clang as CUDA compiler? [y/N]:
nvcc will be used as CUDA compiler.
Please specify which gcc should be used by nvcc as the host compiler. [Default is /usr/bin/gcc]:
Do you wish to build TensorFlow with MPI support? [y/N]:
No MPI support will be enabled for TensorFlow.
Please specify optimization flags to use during compilation when bazel option "--config=opt" is specified [Default is -march=native]:

Would you like to interactively configure ./WORKSPACE for Android builds? [y/N]:
Not configuring the WORKSPACE for Android builds.

Preconfigured Bazel build configs. You can use any of the below by adding "--config=<>" to your build command. See .bazelrc for more details.
    --config=mkl            # Build with MKL support.
    --config=monolithic     # Config for mostly static monolithic build.
    --config=gdr			# Build with GDR support
    --config=verbs			# Build with libverbs support
    --config=ngraph			# BUild with Intel nGraph support
    --config=numa			# Build with NUMA suuport
    --config=dynamic_kernels # (Experimental) Build kernels into separate shared objects.
Preconfigured Bazel build configs to DISABLE default on feature :
	--config=noaws			# Disable AWS S3 filesystem support
	--config=nogcp			# Disable GCP support
	--config=nohdfs			# Disable HDFS support
	--config=noignite		# Disable Apache Ignite support
	--config=nokafka		# Disable Apache Kafka support
	--config=nonccl			# Disable NVIDIA NCCL support
Configuration finished
```

以上配置结束，需要注意的是，一个python的环境，还有python的包路径，CUDA的安装路径，cuDNN的路径，一般是要放在CUDA路径里面的，还有就是你的GPU的算力，还有就是gcc的位置，这个也很关键。

#### 6. 构建pip安装包

仅仅支持CPU

```
bazel build --config=opt //tensorflow/tools/pip_package:build_pip_package
```

支持GPU

```
bazel build --config=opt --config=cuda //tensorflow/tools/pip_package:build_pip_package
```

以上命令构建好了pip构建命令，通过以下就可以生成wheel包

```
./bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
```

以上的命令就在/tmp/tensorflow_pkg中生成了要的.whl包。

尽管可以在同一个源代码树下构建 CUDA 和非 CUDA 配置，但建议您在同一个源代码树中的这两种配置之间切换时运行 `bazel clean`。

#### 7. 安装.whl包

当然是在你的python环境（这个环境和上面依赖的环境可以不是同一个）中，使用pip直接安装：

```
pip install /tmp/tensorflow_pkg/tensorflow-version-tags.whl
```

#### 8. 结语

今天为了尝试一下从源码安装TensorFlow，这样的好处是以后可以根据自己的需求定制编译，然后使得自己的机器性能达到最大化，在这个过程中也学到了很多。大家有什么问题可以联系我：

QQ：329804334

Mail： mizeshuang@gmail.com