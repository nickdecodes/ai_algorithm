# 手写数字识别（基于tensorflow）

![卷积神经网络gif动图](https://geektutu.com/post/tensorflow2-mnist-cnn/cnn_image_sample.gif)

[大白话讲解卷积神经网络工作原理](https://www.bilibili.com/video/av35087157/)，推荐一个bilibili的讲卷积神经网络的视频，up主从youtube搬运过来，用中文讲了一遍。

这篇文章是 **TensorFlow 2.0 Tutorial** 入门教程的第五篇文章，介绍如何使用**卷积神经网络**（Convolutional Neural Network, **CNN**）来提高mnist手写数字识别的准确性。之前使用了最简单的784x10的神经网络，达到了 `0.91` 的正确性，而这篇文章在使用了卷积神经网络后，正确性达到了`0.99`

> **卷积神经网络**（Convolutional Neural Network, **CNN**）是一种[前馈神经网络](https://zh.wikipedia.org/wiki/前馈神经网络)，它的人工神经元可以响应一部分覆盖范围内的周围单元，对于大型图像处理有出色表现。
>
> 卷积神经网络由一个或多个卷积层和顶端的全连通层（对应经典的神经网络）组成，同时也包括关联权重和池化层（pooling layer）。这一结构使得卷积神经网络能够利用输入数据的二维结构。与其他深度学习结构相比，卷积神经网络在图像和[语音识别](https://zh.wikipedia.org/wiki/语音识别)方面能够给出更好的结果。这一模型也可以使用[反向传播算法](https://zh.wikipedia.org/wiki/反向传播算法)进行训练。相比较其他深度、前馈神经网络，卷积神经网络需要考量的参数更少，使之成为一种颇具吸引力的深度学习结构。
>
> ——[维基百科](https://zh.wikipedia.org/wiki/卷积神经网络)

## 1. 安装TensorFlow 2.0

Google与2019年3月发布了TensorFlow 2.0，TensorFlow 2.0 清理了废弃的API，通过减少重复来简化API，并且通过Keras能够轻松地构建模型，从这篇文章开始，教程示例采用`TensorFlow 2.0`版本。

```bash
conda install tensorflow
```

或者在这里下载whl包安装：https://pypi.tuna.tsinghua.edu.cn/simple/tensorflow/

## 2. 代码目录结构

```python
.
├── README.md
├── __pycache__
│   └── model.cpython-37.pyc
├── checkset
│   ├── 0005.data-00000-of-00001
│   ├── 0005.index
│   └── checkpoint
├── dataset
│   ├── t10k-images.idx3-ubyte
│   ├── t10k-labels.idx1-ubyte
│   ├── train-images.idx3-ubyte
│   └── train-labels.idx1-ubyte
├── minstset
├── model.py
├── outimages.py
├── predict.py
├── testimages
│   ├── 0.png
│   ├── 1.png
│   ├── 4.png
│   └── 5.png
└── train.py
```

## 3. CNN模型代码（model.py）

模型定义的前半部分主要使用Keras.layers提供的`Conv2D`（卷积）与`MaxPooling2D`（池化）函数。

CNN的输入是维度为 (image_height, image_width, color_channels)的张量，mnist数据集是黑白的，因此只有一个`color_channel`（颜色通道），一般的彩色图片有3个（R,G,B）,熟悉Web前端的同学可能知道，有些图片有4个通道(R,G,B,A)，A代表透明度。对于mnist数据集，输入的张量维度就是(28,28,1)，通过参数`input_shape`传给网络的第一层。

```python
import tensorflow as tf

class CNN(object):
    def __init__(self):
        model = tf.keras.models.Sequential()
        # 第1层卷积，卷积核大小为3*3，32个，28*28为待训练图片的大小
        model.add(tf.keras.layers.Conv2D(32, (3, 3), activation='relu', input_shape=(28, 28, 1)))
        model.add(tf.keras.layers.MaxPooling2D((2, 2)))
        # 第2层卷积，卷积核大小为3*3，64个
        model.add(tf.keras.layers.Conv2D(64, (3, 3), activation='relu'))
        model.add(tf.keras.layers.MaxPooling2D((2, 2)))
        # 第3层卷积，卷积核大小为3*3，64个
        model.add(tf.keras.layers.Conv2D(64, (3, 3), activation='relu'))

        model.add(tf.keras.layers.Flatten())
        model.add(tf.keras.layers.Dense(64, activation='relu'))
        model.add(tf.keras.layers.Dense(10, activation='softmax'))

        model.summary()

        self.model = model
```

`model.summary()`用来打印我们定义的模型的结构。

```python
Model: "sequential"
_________________________________________________________________
Layer (type)                 Output Shape              Param #   
=================================================================
conv2d (Conv2D)              (None, 26, 26, 32)        320       
_________________________________________________________________
max_pooling2d (MaxPooling2D) (None, 13, 13, 32)        0         
_________________________________________________________________
conv2d_1 (Conv2D)            (None, 11, 11, 64)        18496     
_________________________________________________________________
max_pooling2d_1 (MaxPooling2 (None, 5, 5, 64)          0         
_________________________________________________________________
conv2d_2 (Conv2D)            (None, 3, 3, 64)          36928     
_________________________________________________________________
flatten (Flatten)            (None, 576)               0         
_________________________________________________________________
dense (Dense)                (None, 64)                36928     
_________________________________________________________________
dense_1 (Dense)              (None, 10)                650       
=================================================================
Total params: 93,322
Trainable params: 93,322
Non-trainable params: 0
_________________________________________________________________
```

我们可以看到，每一个`Conv2D`和`MaxPooling2D`层的输出都是一个三维的张量(height, width, channels)。height和width会逐渐地变小。输出的channel的个数，是由第一个参数(例如，32或64)控制的，随着height和width的变小，channel可以变大（从算力的角度）。

模型的后半部分，是定义输出张量的。`layers.Flatten`会将三维的张量转为一维的向量。展开前张量的维度是(3, 3, 64) ，转为一维(576)的向量后，紧接着使用`layers.Dense`层，构造了2层全连接层，逐步地将一维向量的位数从576变为64，再变为10。

后半部分相当于是构建了一个隐藏层为64，输入层为576，输出层为10的普通的神经网络。最后一层的激活函数是`softmax`，10位恰好可以表达0-9十个数字。

最大值的下标即可代表对应的数字，使用`numpy`很容易计算出来： 

```python
import numpy as np

y1 = [0, 0.8, 0.1, 0.1, 0, 0, 0, 0, 0, 0]
y2 = [0, 0.1, 0.1, 0.1, 0.5, 0, 0.2, 0, 0, 0]
np.argmax(y1) # 1
np.argmax(y2) # 4
```

## 4. mnist数据集预处理（train.py）

```python
class DataSource(object):
    def __init__(self):

        # mnist数据集存储的位置，如何不存在将自动下载
        data_path = os.path.abspath(os.path.dirname(__file__)) + './dataset/'
        (train_images, train_labels), (test_images, test_labels) = tf.keras.datasets.mnist.load_data(data_path)
        # 6万张训练图片，1万张测试图片
        train_images = train_images.reshape((60000, 28, 28, 1))
        test_images = test_images.reshape((10000, 28, 28, 1))
        # 像素值映射到 0 - 1 之间
        train_images, test_images = train_images / 255.0, test_images / 255.0

        self.train_images, self.train_labels = train_images, train_labels
        self.test_images, self.test_labels = test_images, test_labels

```

因为mnist数据集国内下载不稳定，因此数据集也同步到了Github仓库。

对mnist数据集的介绍，大家可以参考这个系列的第一篇文章[TensorFlow入门(一) - mnist手写数字识别(网络搭建)](https://geektutu.com/post/tensorflow-mnist-simplest.html)。

## 5. 开始训练并保存训练结果（train.py）

```python
class Train:
    def __init__(self):
        self.cnn = CNN()
        self.data = DataSource()

    def train(self):
        check_path = '../checkset/{epoch:04d}'
        # period 每隔5epoch保存一次
        save_model_cb = tf.keras.callbacks.ModelCheckpoint(check_path, save_weights_only=True, verbose=1, period=5)

        self.cnn.model.compile(optimizer='adam',
                               loss='sparse_categorical_crossentropy',
                               metrics=['accuracy'])
        self.cnn.model.fit(self.data.train_images, self.data.train_labels, epochs=5, callbacks=[save_model_cb])

        test_loss, test_acc = self.cnn.model.evaluate(self.data.test_images, self.data.test_labels)
        print("准确率: %.4f，共测试了%d张图片 " % (test_acc, len(self.data.test_labels)))


if __name__ == "__main__":
    app = Train()
    app.train()
```

在执行`python train.py`后，会得到以下的结果：

```python
Train on 60000 samples
Epoch 1/5
60000/60000 [==============================] - 45s 749us/sample - loss: 0.1477 - accuracy: 0.9536
Epoch 2/5
60000/60000 [==============================] - 45s 746us/sample - loss: 0.0461 - accuracy: 0.9860
Epoch 3/5
60000/60000 [==============================] - 50s 828us/sample - loss: 0.0336 - accuracy: 0.9893
Epoch 4/5
60000/60000 [==============================] - 50s 828us/sample - loss: 0.0257 - accuracy: 0.9919
Epoch 5/5
59968/60000 [============================>.] - ETA: 0s - loss: 0.0210 - accuracy: 0.9930
Epoch 00005: saving model to ./ckpt/cp-0005.ckpt
60000/60000 [==============================] - 51s 848us/sample - loss: 0.0210 - accuracy: 0.9930
10000/10000 [==============================] - 3s 290us/sample - loss: 0.0331 - accuracy: 0.9901
准确率: 0.9901，共测试了10000张图片
```

可以看到，在第一轮训练后，识别准确率达到了0.9536，5轮之后，使用测试集验证，准确率达到了`0.9901`

在第五轮时，模型参数成功保存在了`./ckpt/cp-0005.ckpt`。接下来我们就可以加载保存的模型参数，恢复整个卷积神经网络，进行真实图片的预测了。

## 6. 图片预测（predict.py）

为了将模型的训练和加载分开，预测的代码写在了`predict.py`中。

```python
import numpy as np
from PIL import Image
from model import CNN
import tensorflow as tf


class Predict(object):
    def __init__(self):
        latest = tf.train.latest_checkpoint('checkset')
        self.cnn = CNN()
        # 恢复网络权重
        self.cnn.model.load_weights(latest)

    def predict(self, image_path):
        # 以黑白方式读取图片
        img = Image.open(image_path).convert('L')
        flatten_img = np.reshape(img, (28, 28, 1))
        x = np.array([1 - flatten_img])

        # API refer: https://keras.io/models/model/
        y = self.cnn.model.predict(x)

        # 因为x只传入了一张图片，取y[0]即可
        # np.argmax()取得最大值的下标，即代表的数字
        print(image_path)
        print(y[0])
        print('        -> Predict digit', np.argmax(y[0]))


if __name__ == "__main__":
    app = Predict()
    app.predict('./testimages/0.png')
    app.predict('./testimages/1.png')
    app.predict('./testimages/4.png')
    app.predict('./testimages/5.png')

```

最终，执行predict.py，可以看到：

```python
$ python predict.py
../test_images/0.png
[1. 0. 0. 0. 0. 0. 0. 0. 0. 0.]
        -> Predict digit 0
../test_images/1.png
[0. 1. 0. 0. 0. 0. 0. 0. 0. 0.]
        -> Predict digit 1
../test_images/4.png
[0. 0. 0. 0. 1. 0. 0. 0. 0. 0.]
        -> Predict digit 4
```

## 附：与TensorFlow1.0的区别总结

1. 数据集从**tensorflow.examples.tutorials.mnist**切换到了**tensorflow.keras.datasets**
2. Keras的接口成为了主力，**datasets, layers, models**都是从Keras引入的，而且在网络的搭建上，代码更少，更为简洁。