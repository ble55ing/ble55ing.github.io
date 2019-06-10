---
layout: post
title:  "基于 roBDD 的细颗粒度动态污点分析"
categories: paper
tags: paper DTA
author: ble55ing
---

* content
{:toc}
## Keras 部分功能 

这次算是与Keras和tensorflow打了一次交道了。。虽然最后并没有怎么用上。

## 自定义损失函数

损失函数的自定义用于实现一些目的的模型计算 。注意损失函数是模型训练的目的，而不属于模型之中。

```
def my_bin(y_true, y_pred):
    return K.binary_crossentropy(y_true,y_pred);
```

这样的方式就可以重新写一个损失函数了。当然也是可以有额外的损失函数的参数的，就是需要去定义预设值

## 自定义层

这里给出官方的说明文档，最后没用上所以可没有具体的写

```
class MyLayer(Layer):

    def __init__(self, output_dim, **kwargs):
        self.output_dim = 1
        super(MyLayer, self).__init__(**kwargs)

    def build(self, input_shape):
        assert isinstance(input_shape, list)
        # 为该层创建一个可训练的权重
        self.kernel = self.add_weight(name='kernel',
                                      shape=(input_shape[0][1], self.output_dim),
                                      initializer='uniform',
                                      trainable=True)
        super(MyLayer, self).build(input_shape)  # 一定要在最后调用它

    def call(self, x):
        assert isinstance(x, list)
        a, b = x
        return [K.dot(a, self.kernel) + b, K.mean(b, axis=-1)]

    def compute_output_shape(self, input_shape):
        assert isinstance(input_shape, list)
        shape_a, shape_b = input_shape
        return [(shape_a[0], self.output_dim), shape_b[:-1]]
```

官方文档的网址：https://keras.io/zh/layers/writing-your-own-keras-layers/，很多时候比个人写的blog有用的多

## 模型模拟

很有意思的想法，用一个新的模型根据之前的模型进行简化的模拟，然后可以对这个模型进行操作，也不会影响到之前的模型了。

还可以通过模型的predict来预测某个输入对应模型输出（当然是用model.predict([x])的方法也是可以的得到模型输出的，这两种得到的其实是一样的，就是新建立模型可以在上面做一些动作）。

```
dense1_layer_model = Model(inputs=model.input,outputs=model.get_layer('activation_2').output)
dense1_output = dense1_layer_model.predict([x])
print dense1_output[0]
```

## 关于计算模型梯度

```
loss = layer_list[-2][1].output[:,f]#分片，选择了第f个神经元的输出与输入之间的关系是个Tensor，shape(?,)。原先是shape(?,45)。
grads = K.gradients(loss,model.input)[0]
iterate = K.function([model.input], [loss, grads])
loss_value, grads_value = iterate([x])
```

最终得到的就是损失函数计算值和各输入字节的梯度



