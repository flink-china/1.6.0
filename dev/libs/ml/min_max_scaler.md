# 最小最大值标准化

* [描述](#描述)
* [操作](#操作)
    * [拟合](#拟合)
    * [转换](#转换)
* [参数](#参数)
* [示例](#示例)

## 描述

`最小最大值标准化` 通过对给定数据集进行缩放，使得所有值都落在指定的区间 \[min,max\]内。如果用户没有指定区间的最大和最小值，则 MinMax Scaler将会把输入特征缩放到 \[0,1\] 区间内。 给定输入数据集：`$x_1, x_2,… x_n$`，其中最小值为：
```math
x_{min} = min({x_1, x_2,..., x_n})
```
最大值为：
```math
x_{max} = max({x_1, x_2,..., x_n})
```
经过缩放的数据集 `$ z_1, z_2,…,z_n $` 为：
```math
z_{i}= \frac{x_{i} - x_{min}} {x_{max} - x_{min}} ( max - min) + min
```
其中 `$\textit{min}$` 和 `$\textit{max}$` 是用户指定的最小值和最大值。

## 操作

`最小最大值标准化` 是一个 `转换器`。因此支持拟合和转换操作。

### 拟合

`最小最大值标准化` 可以在所有 `Vector` 或 `LabeledVector`的子类型上进行训练:

*   `fit[T <: Vector]: DataSet[T] => Unit`
*   `fit: DataSet[LabeledVector] => Unit`

### 转换

`最小最大值标准化` 把 `Vector` 或 `LabeledVector` 的子类型数据集转换到对应的相同类型的数据集：

*   `transform[T <: Vector]: DataSet[T] => DataSet[T]`
*   `transform: DataSet[LabeledVector] => DataSet[LabeledVector]`

## 参数

`最小最大值标准化` 可由下列两个参数进行控制：

参数 | 描述
---|---
**Min** | 缩放数据集范围的最小值 (默认值: **0.0**)
**Max** | 缩放数据集范围的最大值 (默认值: **1.0**)

## 示例

```scala
// Create MinMax scaler transformer
val minMaxscaler = MinMaxScaler()
    .setMin(-1.0)

// Obtain data set to be scaled
val dataSet: DataSet[Vector] = ...

// Learn the minimum and maximum values of the training data
minMaxscaler.fit(dataSet)

// Scale the provided data set to have min=-1.0 and max=1.0
val scaledDS = minMaxscaler.transform(dataSet)
```