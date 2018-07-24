---
title: StackOverflow上简单数字识别案例学习
tags: OpenCV-Python
mathjax: true
category: OCR
author: Leno
thumbnail: 'https://i.loli.net/2018/07/24/5b56e06b958cf.png'
abbrlink: 8b09dcdd
date: 2018-07-24 12:59:38
---

>正如题目那样，这次的学习是StackOverflow上的一个案例，它的地址为[https://stackoverflow.com/questions/9413216/simple-digit-recognition-ocr-in-opencv-python](https://stackoverflow.com/questions/9413216/simple-digit-recognition-ocr-in-opencv-python)，但是源地址上的代码略微有点旧，很多函数在OpenCV3中有了些许的改动，所以我还是会贴上我自己改正过后的代码，然后附上我对作者的思路的理解整理以及对所有代码的理解，如有问题，欢迎和我进行沟通交流，邮箱地址：cliugeek@us-forever.com

# 思路： #

整体的思路：我们需要有一个训练（其实严格意义上说不上是训练，其实我们最终要做的是每个数字的特征提取并且将他们与实际的数字字符一一对应起来，姑且就称这个过程为“训练”吧）的图像，然后我们提取其中所有的数字的特征，然后根据一一对应的的关系，将特征和对应的字符存储起来，以便我们后续做其他图像中的数字识别使用。

所以先贴一下训练用到的数据图：

![digit.png](https://i.loli.net/2018/07/24/5b56be2a72945.png)

其实这样的训练数据存在着一个问题，那就是所有的字母都是相同的字体和大小，所以整体的鲁棒性很受影响。所以这是本程序需要做继续优化的地方。

接下来我们讲具体的训练过程：

	1 加载图像

	2 检测数字（通过轮廓查找找，并且通过对检测到的字母的面积和高度做相应的约束，来避免错误的检测）

	3 对检测到的字母绘制边框

	4 每绘制出一个边框，我们就需要在键盘上键入相应的数字键，这一步很关键，否则会导致我们训练过程无效，导致后面一系列的工作无法进行

	5 当我们按下相应的数字键之后，边框里面的东西便会进行重新的resize，变成10x10的大小，然后将其存入到一个100维的数组中（其实就是将这些像素值都存了进去作为我们的特征提取），相应的也会将我们键入的数字存到另一个data文件当中去

	6 训练过程完成之后，系统生成两个data文件。

在手动数字分类结束时，训练数据（其实就是上面给大家的那张图像）中所有的数字，都已经由我们手工标记，结果如下：

![norm.png](https://i.loli.net/2018/07/24/5b56bf1d8d153.png)

下面就是针对上述过程的代码：

```Python
import sys

import numpy as np
import cv2

im = cv2.imread('digit.png')
im3 = im.copy()

gray = cv2.cvtColor(im, cv2.COLOR_BGR2GRAY)
blur = cv2.GaussianBlur(gray, (5, 5), 0)
thresh = cv2.adaptiveThreshold(blur, 255, 1, 1, 11, 2)

#################      Now finding Contours         ###################

image, contours, hierarchy = cv2.findContours(thresh, cv2.RETR_LIST, cv2.CHAIN_APPROX_SIMPLE)

samples = np.empty((0, 100))
responses = []
keys = [i for i in range(48, 58)]

for cnt in contours:
    if cv2.contourArea(cnt) > 50:
        [x, y, w, h] = cv2.boundingRect(cnt)

        if h > 28:
            cv2.rectangle(im, (x, y), (x + w, y + h), (0, 0, 255), 2)
            roi = thresh[y:y + h, x:x + w]
            roismall = cv2.resize(roi, (10, 10))
            cv2.imshow('norm', im)
            key = cv2.waitKey(0)

            if key == 27:  # (escape to quit)
                sys.exit()
            elif key in keys:
                responses.append(int(chr(key)))
                sample = roismall.reshape((1, 100))
                samples = np.append(samples, sample, 0)

responses = np.array(responses, np.float32)
responses = responses.reshape((responses.size, 1))
print("training complete")

np.savetxt('generalsamples.data', samples)
np.savetxt('generalresponses.data', responses)

```

代码其实也很简单，所以简单的说一下，首先对图像进行灰度化，然后进行高斯去噪，再然后做二值化。接下来进行轮廓检测，然后对每个检测到的轮廓在面积和高度上进行控制，如果符合设定的标准就进行画框、resize和display，同时将输入的key进行相应的存储，到最后进行save，将他们分别存到generalsamples.data和generalresponses.data中。

接下来进行测试部分，同样先给出测试所用的图像：

![dig.png](https://i.loli.net/2018/07/24/5b56ca87860f9.png)

再然后就是测试部分的步骤：首先将刚才我们生成的data文件load进来，也就是将刚才的模型加载到内存中，然后我们用KNN（K邻近值法，选取与当前的测试图像中最相近的作为我们预测出来的值）做预测，然后针对于检测出来的每个轮廓都进行这样的操作，预测出来的图像，我们通过puttext方法放到将要生成的图像out上，最后将原图im和生成图out显示出来。

代码如下：

```Python
import cv2
import numpy as np

#######   training part    ###############
samples = np.loadtxt('generalsamples.data', np.float32)
responses = np.loadtxt('generalresponses.data', np.float32)
responses = responses.reshape((responses.size, 1))

model = cv2.ml.KNearest_create()
model.train(samples, cv2.ml.ROW_SAMPLE, responses)
# model.train(samples,responses)

############################# testing part  #########################

im = cv2.imread('dig.png')
out = np.zeros(im.shape, np.uint8)
gray = cv2.cvtColor(im, cv2.COLOR_BGR2GRAY)
thresh = cv2.adaptiveThreshold(gray, 255, 1, 1, 11, 2)

image, contours, hierarchy = cv2.findContours(thresh, cv2.RETR_LIST, cv2.CHAIN_APPROX_SIMPLE)

for cnt in contours:
    if cv2.contourArea(cnt) > 50:
        [x, y, w, h] = cv2.boundingRect(cnt)
        if h > 28:
            cv2.rectangle(im, (x, y), (x + w, y + h), (0, 255, 0), 2)
            roi = thresh[y:y + h, x:x + w]
            roismall = cv2.resize(roi, (10, 10))
            roismall = roismall.reshape((1, 100))
            roismall = np.float32(roismall)
            retval, results, neigh_resp, dists = model.findNearest(roismall, k=1)
            string = str(int((results[0][0])))
            cv2.putText(out, string, (x, y + h), 0, 1, (0, 255, 0))

cv2.imshow('im', im)
cv2.imshow('out', out)
cv2.waitKey(0)

```

这部分的代码其实按照刚才的流程理解就可以，所以不再赘述。

最后给大家看下效果：

![im.png](https://i.loli.net/2018/07/24/5b56dd88da6ad.png)

![out.png](https://i.loli.net/2018/07/24/5b56dd88c14c9.png)

我仔细对比了一遍，准确率可以达到100%。对于这个简单的例子，这样的结果可以称得上完美吧。

以上就是本次学习的所有内容。感谢驻足~