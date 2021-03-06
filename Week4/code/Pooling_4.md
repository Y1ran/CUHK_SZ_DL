# 池化层

回忆在[“二维卷积层”](conv-layer.md)小节里介绍的图像物体边缘检测应用中，我们构造卷积核从而精确地找到了像素变化的位置。设任意二维数组`X`的`i`行`j`列的元素为`X[i, j]`。如果我们构造的卷积核输出`Y[i, j]=1`，那么说明输入中`X[i, j]`和`X[i, j+1]`数值不一样。这可能意味着物体边缘通过这两个元素之间。但实际中图像里我们感兴趣的物体不会总出现在固定位置：即使我们连续拍摄同一个物体也极有可能出现像素上的偏移。这样导致同一个边缘对应的输出可能出现在卷积输出`Y`中的不同位置，进而对后面的模式识别造成不便。

这一节我们介绍池化（pooling）层，它的提出是为了缓解卷积层对位置的过度敏感性。

## 二维最大池化层和平均池化层

同卷积层一样，池化层每次对输入数据的一个固定形状窗口（又称池化窗口）中的元素计算输出。不同于卷积层里计算输入和核的互相关性，池化层直接计算池化窗口内元素的最大值或者平均值。该运算也称最大池化或平均池化。在最大池化中，池化窗口从输入数组的最左和最上方开始，按从左往右、从上往下的顺序，依次在输入数组上滑动。当池化窗口滑动到某一位置时，窗口中的输入子数组的最大值即输出数组中相应位置的元素。

![池化窗口形状为$2\times 2$的最大池化层。阴影部分为第一个输出元素及其计算所使用的输入元素：$\max(0,1,3,4)=4$](./pooling.svg)

图5.6中的输出数组高和宽分别为2，其中的四个元素由取最大值运算得出：

$$
\max(0,1,3,4)=4,\\
\max(1,2,4,5)=5,\\
\max(3,4,6,7)=7,\\
\max(4,5,7,8)=8.\\
$$


平均池化的工作原理与最大池化类似，但将最大运算符替换成平均运算符。池化窗口形状为$p \times q$的池化层称为$p \times q$池化层。其中的池化计算叫做$p \times q$池化。

让我们再次回到本节开始提到的图像物体边缘检测例子。现在我们将卷积层的输出作为$2\times 2$最大池化的输入。设该卷积层输入是`X`、池化层输出为`Y`。无论是`X[i, j]`和`X[i, j+1]`值不同，还是`X[i, j+1]`和`X[i, j+2]`不同，池化层输出均有`Y[i, j]=1`。也就是说，使用$2\times 2$最大池化层，只要卷积层识别的模式在高和宽上移动不超过一个元素，我们依然可以将它检测出来。

下面我们把池化层的前向计算实现在`pool2d`函数里。它跟[“二维卷积层”](conv-layer.md)一节里`corr2d`函数非常类似，唯一的区别是在计算输出`Y`上。

```{.python .input  n=1}
from mxnet import nd
from mxnet.gluon import nn

def pool2d(X, pool_size, mode='max'):
    p_h, p_w = pool_size
    Y = nd.zeros((X.shape[0] - p_h + 1, X.shape[1] - p_w + 1))
    for i in range(Y.shape[0]):
        for j in range(Y.shape[1]):
            if mode == 'max':
                Y[i, j] = X[i: i + p_h, j: j + p_w].max()
            elif mode == 'avg':
                Y[i, j] = X[i: i + p_h, j: j + p_w].mean()            
    return Y
```

```{.json .output n=1}
[
 {
  "name": "stderr",
  "output_type": "stream",
  "text": "/home/zli/anaconda3/lib/python3.6/site-packages/h5py/__init__.py:36: FutureWarning: Conversion of the second argument of issubdtype from `float` to `np.floating` is deprecated. In future, it will be treated as `np.float64 == np.dtype(float).type`.\n  from ._conv import register_converters as _register_converters\n"
 }
]
```

我们可以构造图5.6中的输入数组`X`来验证二维最大池化层的输出。

```{.python .input  n=2}
X = nd.array([[0, 1, 2], [3, 4, 5], [6, 7, 8]])
pool2d(X, (2, 2))
```

```{.json .output n=2}
[
 {
  "data": {
   "text/plain": "\n[[4. 5.]\n [7. 8.]]\n<NDArray 2x2 @cpu(0)>"
  },
  "execution_count": 2,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

同时我们实验一下平均池化层。

```{.python .input  n=3}
pool2d(X, (2, 2), 'avg')
```

```{.json .output n=3}
[
 {
  "data": {
   "text/plain": "\n[[2. 3.]\n [5. 6.]]\n<NDArray 2x2 @cpu(0)>"
  },
  "execution_count": 3,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

## 填充和步幅

同卷积层一样，池化层也可以在输入的高和宽两侧的填充并调整窗口的移动步幅来改变输出形状。池化层填充和步幅与卷积层填充和步幅的工作机制一样。我们将通过`nn`模块里的二维最大池化层MaxPool2D来演示池化层填充和步幅的工作机制。我们先构造一个形状为`(1, 1, 4, 4)`的输入数据，前两个维度分别是批量和通道。

```{.python .input  n=4}
X = nd.arange(16).reshape((1, 1, 4, 4))
X
```

```{.json .output n=4}
[
 {
  "data": {
   "text/plain": "\n[[[[ 0.  1.  2.  3.]\n   [ 4.  5.  6.  7.]\n   [ 8.  9. 10. 11.]\n   [12. 13. 14. 15.]]]]\n<NDArray 1x1x4x4 @cpu(0)>"
  },
  "execution_count": 4,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

`MaxPool2D`类里默认步幅设置成跟池化窗口大小一样。下面使用形状为`(3, 3)`的池化窗口，默认获得形状为`(3, 3)`的步幅。

```{.python .input  n=5}
pool2d = nn.MaxPool2D(3)
pool2d(X)  # 因为池化层没有模型参数，所以不需要调用参数初始化函数。
```

```{.json .output n=5}
[
 {
  "data": {
   "text/plain": "\n[[[[10.]]]]\n<NDArray 1x1x1x1 @cpu(0)>"
  },
  "execution_count": 5,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

我们可以手动指定步幅和填充。

```{.python .input  n=6}
pool2d = nn.MaxPool2D(3, padding=1, strides=2)
pool2d(X)
```

```{.json .output n=6}
[
 {
  "data": {
   "text/plain": "\n[[[[ 5.  7.]\n   [13. 15.]]]]\n<NDArray 1x1x2x2 @cpu(0)>"
  },
  "execution_count": 6,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

当然，我们也可以指定非正方形的池化窗口，并分别指定高和宽上的填充和步幅。

```{.python .input  n=7}
pool2d = nn.MaxPool2D((2, 3), padding=(1, 2), strides=(2, 3))
pool2d(X)
```

```{.json .output n=7}
[
 {
  "data": {
   "text/plain": "\n[[[[ 0.  3.]\n   [ 8. 11.]\n   [12. 15.]]]]\n<NDArray 1x1x3x2 @cpu(0)>"
  },
  "execution_count": 7,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

## 多通道

在处理多通道输入数据时，池化层对每个输入通道分别池化，而不是像卷积层那样对各通道的输入按通道相加。这意味着池化层的输出通道数跟输入通道数相同。下面我们将数组`X`和`X+1`在通道维上连结来构造通道数为2的输入。

```{.python .input  n=8}
X = nd.concat(X, X + 1, dim=1)
X
```

```{.json .output n=8}
[
 {
  "data": {
   "text/plain": "\n[[[[ 0.  1.  2.  3.]\n   [ 4.  5.  6.  7.]\n   [ 8.  9. 10. 11.]\n   [12. 13. 14. 15.]]\n\n  [[ 1.  2.  3.  4.]\n   [ 5.  6.  7.  8.]\n   [ 9. 10. 11. 12.]\n   [13. 14. 15. 16.]]]]\n<NDArray 1x2x4x4 @cpu(0)>"
  },
  "execution_count": 8,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

池化后，我们发现输出通道数仍然是2。

```{.python .input  n=9}
pool2d = nn.MaxPool2D(3, padding=1, strides=2)
pool2d(X)
```

```{.json .output n=9}
[
 {
  "data": {
   "text/plain": "\n[[[[ 5.  7.]\n   [13. 15.]]\n\n  [[ 6.  8.]\n   [14. 16.]]]]\n<NDArray 1x2x2x2 @cpu(0)>"
  },
  "execution_count": 9,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

## 小结

* 池化层取池化窗口中输入元素的最大值或者平均值作为输出。
* 池化层的一个主要作用是缓解卷积层对位置的敏感性。

## 练习

* 分析池化层的计算复杂度。假设输入形状为$c\times h\times w$，我们使用形状为$p_h\times p_w$的池化窗口，而且使用$(p_h, p_w)$填充和$(s_h, s_w)$步幅。这个池化层的前向计算复杂度有多大？
* 想一想，最大池化层和平均池化层在作用上可能有哪些区别？
* 你觉得最小池化层这个想法有没有意义？

## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/6406)

![](../img/qr_pooling.svg)
