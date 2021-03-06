# 数据集：快速了解

@{tf.data} 模块包含了一组类，这些类可以让你轻松的加载数据、操作数据，并将数据传送到您的模型中。文档通过如下这两个简单的例子来介绍该 API：

* 从 numpy 数组读取内存数据。
* 逐行读取 csv 文件。

<!-- TODO(markdaoust): Add links to an example reading from multiple-files
(image_retraining), and a from_generator example. -->

## 基本输入

学习如何获取数组的片段，是开始学习 `tf.data` 最简单的方式。

章节 @{$get_started/premade_estimators$Premade Estimators} 在文件 [`iris_data.py`](https://github.com/tensorflow/models/blob/master/samples/core/get_started/iris_data.py) 中定义了 `train_input_fn`，它可以将数据传输到 Estimator：

``` python
def train_input_fn(features, labels, batch_size):
    """一个用来训练的输入函数"""
    # 将输入值转化为数据集。
    dataset = tf.data.Dataset.from_tensor_slices((dict(features), labels))

    # 混排、重复、批处理样本。
    dataset = dataset.shuffle(1000).repeat().batch(batch_size)

    # 返回数据集
    return dataset
```

下面我们来对这个函数做更仔细的分析。

### 参数

这个函数一共需要三个参数。如果一个参数的期望类型是 “array” （数组），那么它将可以接受几乎所有可以用 `numpy.array` 来转化为数组的值。我们可以看到只有一个例外：[`tuple`](https://docs.python.org/3/tutorial/datastructures.html#tuples-and-sequences)，它对 `Datasets` 有特殊的含义。

* `features`：一个形如 `{'feature_name':array}` 的数据字典（或者是 [`DataFrame`](https://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.html)），它包含了原始的输入特征。
* `labels`：一个包含每个样本的 [label](https://developers.google.com/machine-learning/glossary/#label) 的数组。
* `batch_size`：一个指示所需批量大小的整数。

在 [`premade_estimator.py`](https://github.com/tensorflow/models/blob/master/samples/core/get_started/premade_estimator.py) 中，我们使用 `iris_data.load_data()` 函数来检索虹膜数据。
你可以运行该函数，并按如下方式解压结果：

``` python
import iris_data

# 获取数据
train, test = iris_data.load_data()
features, labels = train
```

然后用像下面这样的一行代码，将数据传递给 input 函数：

``` python
batch_size=100
iris_data.train_input_fn(features, labels, batch_size)
```

让我们来具体看看 `train_input_fn()` 函数。

### （数组）片段

[需要翻译]The function starts by using the @{tf.data.Dataset.from_tensor_slices} function to create a @{tf.data.Dataset} representing slices of the array. The array is sliced across the first dimension. For example, an array containing the @{$tutorials/layers$mnist training data} has a shape of `(60000, 28, 28)`. Passing this to `from_tensor_slices` returns a `Dataset` object containing 60000 slices, each one a 28x28 image.

返回这个 `Dataset` 的代码如下所示：

``` python
train, test = tf.keras.datasets.mnist.load_data()
mnist_x, mnist_y = train

mnist_ds = tf.data.Dataset.from_tensor_slices(mnist_x)
print(mnist_ds)
```

[需要翻译]This will print the following line, showing the @{$programmers_guide/tensors#shapes$shapes} and @{$programmers_guide/tensors#data_types$types} of the items in the dataset. Note that a `Dataset` does not know how many items it contains.

``` None
<TensorSliceDataset shapes: (28,28), types: tf.uint8>
```

The `Dataset` above represents a simple collection of arrays, but datasets are much more powerful than this. A `Dataset` can transparently handle any nested combination of dictionaries or tuples (or [`namedtuple`](https://docs.python.org/2/library/collections.html#collections.namedtuple)).

For example after converting the iris `features` to a standard python dictionary, you can then convert the dictionary of arrays to a `Dataset` of dictionaries as follows:

``` python
dataset = tf.data.Dataset.from_tensor_slices(dict(features))
print(dataset)
```
``` None
<TensorSliceDataset

  shapes: {
    SepalLength: (), PetalWidth: (),
    PetalLength: (), SepalWidth: ()},

  types: {
      SepalLength: tf.float64, PetalWidth: tf.float64,
      PetalLength: tf.float64, SepalWidth: tf.float64}
>
```

这里我们可以发现，当 `Dataset` 包含了结构化的元素时，`Dataset` 的 `shapes` 和 `types` 就会采用相同结构。这个数据集包含了 @{$programmers_guide/tensors#rank$scalars} 字典，并且都是 `tf.float64` 类型。

[需要翻译]The first line of the iris `train_input_fn` uses the same functionality，但是增加了一层结构。它创建了一个包含 `(features_dict, label)` 数据对的数据集。

以下代码表明，标签是类型为 `int64` 的标量：

``` python
# 将输入转化为数据集。
dataset = tf.data.Dataset.from_tensor_slices((dict(features), labels))
print(dataset)
```
```
<TensorSliceDataset
    shapes: (
        {
          SepalLength: (), PetalWidth: (),
          PetalLength: (), SepalWidth: ()},
        ()),

    types: (
        {
          SepalLength: tf.float64, PetalWidth: tf.float64,
          PetalLength: tf.float64, SepalWidth: tf.float64},
        tf.int64)>
```

### 操作

目前，`Dataset` 会按照固定顺序遍历数据一次，且一次只能生成一个元素。在可以用于训练之前，它需要进一步的处理。幸运的是，`tf.data.Dataset` 类提供了方法来让数据为训练作出更好的准备。`train_input_fn` 的下一行代码就利用了几个这样的方法：

``` python
# 样本的混排、重复、批处理。
dataset = dataset.shuffle(1000).repeat().batch(batch_size)
```

[需要翻译]The @{tf.data.Dataset.shuffle$`shuffle`} method uses a fixed-size buffer to shuffle the items as they pass through. In this case the `buffer_size` is greater than the number of examples in the `Dataset`, ensuring that the data is completely shuffled (The Iris data set only contains 150 examples).

@{tf.data.Dataset.repeat$`repeat`} 方法在 `Dataset` 结束的时将它重启。如果要限制重复的次数，设置 `count` 参数。

@{tf.data.Dataset.batch$`batch`} 方法将会收集一定数量的样本并入栈，以此创建一个批次。这个操作会为样本的 shape 增加一个维度，且新的维度将作为第一维。如下代码在 MNIST 数据集上相对早地应用了 `batch` 方法，导致 `Dataset` 包含了表示 `(28,28)` 图像的三维数组：

``` python
print(mnist_ds.batch(100))
```

``` none
<BatchDataset
  shapes: (?, 28, 28),
  types: tf.uint8>
```
注意，因为最后一个批次将会有比较少的元素，因此数据集的批量大小是不确定的。

在 `train_input_fn` 中，批处理之后，`数据集` 包含元素们的一维向量，这些一维向量的前面部分是：

```python
print(dataset)
```
```
<TensorSliceDataset
    shapes: (
        {
          SepalLength: (?,), PetalWidth: (?,),
          PetalLength: (?,), SepalWidth: (?,)},
        (?,)),

    types: (
        {
          SepalLength: tf.float64, PetalWidth: tf.float64,
          PetalLength: tf.float64, SepalWidth: tf.float64},
        tf.int64)>
```


### 返回

[需要翻译]At this point the `Dataset` contains `(features_dict, labels)` pairs. This is the format expected by the `train` and `evaluate` methods, so the `input_fn` returns the dataset.

The `labels` can/should be omitted when using the `predict` method.

<!--
  TODO(markdaoust): link to `input_fn` doc when it exists
-->

## 读取 CSV 文件

现实中对 `Dataset` 类最常见的应用是从磁盘的文档中获取数据流。@{tf.data} 模块包括了一系列的文件读取器。我们来看看如何使用 `Dataset` 从 csv 文件中分析虹膜数据集。

如下对 `iris_data.maybe_download` 函数的调用，将会在必要的时候下载数据，并返回结果文件的路径：

``` python
import iris_data
train_path, test_path = iris_data.maybe_download()
```

[`iris_data.csv_input_fn`](https://github.com/tensorflow/models/blob/master/samples/core/get_started/iris_data.py) 函数包括了一个用 `Dataset` 解析 csv 文件的替代方案。

让我们来看看如何构建一个兼容 Estimator 的、可以读取本地文件的输入函数。

### 建立 `Dataset`

我们从建立一个 @{tf.data.TextLineDataset$`TextLineDataset`} 对象开始，这个对象一次只读取文件的一行。之后，调用 @{tf.data.Dataset.skip$`skip`} 方法，跳过文件的第一行——这是文件的头部，而不是样本：

``` python
ds = tf.data.TextLineDataset(train_path).skip(1)
```

### 建立一个 csv 行解析器

我们从建立一个可以解析一行的函数开始。

如下的 `iris_data.parse_line` 函数完成了这个目标，它使用了 @{tf.decode_csv} 方法以及一些简单的 python 代码：

为了生成必需的 `(features, label)` 数据对，我们必须解析数据集内的每一行。如下的 `_parse_line` 函数调用了 @{tf.decode_csv} 来将单独一行解析为特征和标签。因为 Estimators 需要特征以字典的方式展现，我们就依靠 python 内建的 `dict` 和 `zip` 函数来建立这个字典。特征的名字是字典的键值 key。然后，调用字典的 `pop` 方法来从特征字典中移除标签字段：

``` python
# 描述文本列的元数据
COLUMNS = ['SepalLength', 'SepalWidth',
           'PetalLength', 'PetalWidth',
           'label']
FIELD_DEFAULTS = [[0.0], [0.0], [0.0], [0.0], [0]]
def _parse_line(line):
    # 将行解码到 fields 中
    fields = tf.decode_csv(line, FIELD_DEFAULTS)

    # 将结果打包成字典
    features = dict(zip(COLUMNS,fields))

    # 将标签从特征中分离
    label = features.pop('label')

    return features, label
```

### 解析多行

当数据集将被传输到一个模型中时，它有很多操作数据的方法。其中，使用最多的是 @{tf.data.Dataset.map$`map`}，它将转换应用到 Dataset 的每个元素中。

这个 `map` 方法接受一个 `map_func` 参数，这个参数描述了 `Dataset` 中的每一个元素应该如何被转化。

<img style="width:100%" src="https://www.tensorflow.org/images/datasets/map.png">

@{tf.data.Dataset.map$`map`} 方法将会对 `Dataset` 中的每一个元素应用 `map_func` 来完成它们的转化。

因此，为了在多行数据被从 csv 文件中读取出来的时候解析它们，我们为 `map` 方法提供 `_parse_line` 函数：

``` python
ds = ds.map(_parse_line)
print(ds)
```
``` None
<MapDataset
shapes: (
    {SepalLength: (), PetalWidth: (), ...},
    ()),
types: (
    {SepalLength: tf.float32, PetalWidth: tf.float32, ...},
    tf.int32)>
```

现在，数据集中包含的是 `(features, label)` 数据对，而不是简单的字符串标量了。

`iris_data.csv_input_fn` 函数的余下部分和 [Basic input](#basic_input) 中介绍的 `iris_data.train_input_fn` 函数相同。

### 实践

这个函数可以作为 `iris_data.train_input_fn` 的替代。它可以像如下这样，来给 estimator 提供数据：

``` python
train_path, test_path = iris_data.maybe_download()

# 所有的输入都是数字
feature_columns = [
    tf.feature_column.numeric_column(name)
    for name in iris_data.CSV_COLUMN_NAMES[:-1]]

# 构建 estimator
est = tf.estimator.LinearClassifier(feature_columns,
                                    n_classes=3)
# 训练 estimator
batch_size = 100
est.train(
    steps=1000,
    input_fn=lambda : iris_data.csv_input_fn(train_path, batch_size))
```

Estimator 期望 `input_fn` 没有任何参数。要解除这个限制，我们使用 `lambda` 来捕获参数并提供预期的接口。

## 总结

为了从不同的数据源中便捷的读取数据，`tf.data` 模块提供了类和函数的集合。除此之外，`tf.data` 有简单并且强大的方法，来应用各种标准和自定义转换。

现在你已经基本了解了如何为 Estimator 高效的获取数据。（作为扩展）接下来可以思考如下的文档：


* @{$get_started/custom_estimators}: 论述了如何构建自定义的 `Estimator` 模型。
* @{$low_level_intro#datasets$Low Level Introduction}: 论述了如何利用 TensorFlow 的低级 API 来直接使用 `tf.data.Datasets` 进行实验。
* @{$programmers_guide/datasets} 详细介绍了 `Datasets` 的附加功能。

