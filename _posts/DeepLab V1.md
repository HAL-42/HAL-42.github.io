---
layout:     post
title:      DeepLab V1
subtitle:   DeepLab V1 Summary
date:       2019-10-14
author:     HAL-42
header-img: img/42.jpg
catalog: true
tags:
    - CV
---

# Deep Lab V1

## Motivation

作者分析了为什么DCNN在语义分割上表现不好：

CNN有内在的transformation invariance（严格来说，只有translate invariance是内生的）。

 ![“translate invariance”的图片搜索结果](https://github.com/HAL-42/HAL-42.github.io/blob/master/_posts/DeepLab V1.assets/iY5n5.png) 

对于在不同位置的目标，DCNN理论上能够给出近乎相同的特征（translate invariance）。这在需要得到抽象语义（例如分类）的时候很有用。

但是对于分割任务，这种特性反而使得网络在变深的过程中失去关键的较浅的位置信息，起到反效果。

作者希望能解决两个问题：

1. 解决卷积网络为了提高感受野，就不得不牺牲分辨率的问题。
2. 对于DCNN内生的对位置不敏感的问题，用后处理办法加以补偿。

## Contribution

### 提出用带洞卷积在不降低分辨率的情况下提高感受野

作者将一个类似FCN版本的VGG的pool4改为stride=1，conv5的卷积层改为dilation为1。第一层全连接不下采样，反而用空洞卷积进一步扩大感受野。
$$
F_后 = （kernal_size - 1) * (Dilation * Stride_前) + F_前
$$
感受野在本层快速增长（一层增长32），但分辨率却没有下降：

![1571024062351](https://github.com/HAL-42/HAL-42.github.io/blob/master/_posts/DeepLab V1.assets/1571024062351.png)

​																	layer14——layer17为带洞卷积

利用带洞卷积，作者在扩大感受野的同时，避免了了分辨率的进一步下降。最终输出分辨率为输入的1/8.

训练时将预训练的VGG16的权重做fine-tune，损失函数取是输出的特征图与ground truth下采样8倍做交叉熵和；测试时取输出图双线性上采样8倍得到结果。 

### CRF产生准确的语义分割 

 而有工作证明可用全连接的CRF来提升分割精度。

作者设置能量函数为：

 *E*(*x*)=∑*i**θ**i*(*x**i*)+∑*i**j**θ**i**j*(*x**i*,*x**j*) 

其中：

 *θ**i*(*x**i*)=−log*P*(*x**i*) 

 *θ**i**j*(*x**i*,*x**j*)=*μ*(*x**i*,*x**j*)∑*m*=1*K**ω**m**k**m*(*f**i*,*f**j*) 

 *ω*1exp−||*p**i*−*p**j*||22*σ*2*α*−||*I**i*−*I**j*||22*σ*2*β*)+*ω*2exp(−||*p**i*−*p**j*||22*σ*2*γ*) 

如此设置特征函数的大意是说: 如果两个像素相邻比较近，他们的颜色被期待比较接近。

其实CRF我也没大部弄懂，还有回去再翻翻书。

## 实验结果

### 对比其他方法

State of art，且对复杂边缘可以有较好的效果。

### Ablation Experiment

* 数据表明，提高感受野（通过调整第一个FC层的input参数），能够较大提高IoU
* 数据表明，用CRF后处理能够极大提升IoU(总是可以+4%左右)
* 数据表明，Multi-Scale（类似FCN的跳层连接）可以稍微提高IoU
* 数据与可视化表明，Multi-Scale和CRF可以提高对边界的分割精度。其中CRF提升效果更好。

## Cookbook

### 数据
#### 数据集

PASCAL VOC 2012

#### 数据增强与分布

没说，从代码目测似乎就用了一个随机Crop

### 网络结构

如上所属, Finetune from VGG/Googlenet/Alexnet；交叉熵损失；

后处理：CRF

### 实验框架

先训练DCNN，训练完毕后用DCNN的输出训练CRF（即交叉验证寻找超参数）。

### Metric

mean IoU

## 不足

### 【设计】尺寸不变性

Scale Invariance 没有得到很好地考虑

这会在之后的Deeplab中得到部分解决。

### 【实现】使用了Interp层

个人以为用常数卷积核的正反卷积实现双线性插值，效率更高，易于编程。

Caffe的Interp模块来自PSPNet的代码，阅读源码发现：

1. 并非官方代码，性能上比较casual，其CPU/GPU模式都没有利用到矩阵运算加速。
2. 对label下采样时如果值要设置地非常小心，如果出错，很容易在label pixels间取平均，形成无意义地label。用卷积和抽样就不易出现此类问题。

### 【设计】DCNN于CRF的训练相互分离

并非End-to-End

有其他论文又End-to-End方法，还没来得及看

## Problem

### 128感受野是怎么算出来的？

用自己推导的公式，以及下面的感受野计算器，都只能得到308感受野这个结论。代码注释和论文中所说的128感受野至今不知道是怎么算出来的。

https://fomoro.com/research/article/receptive-field-calculator#3,1,1,SAME;3,1,1,SAME;2,2,1,SAME;3,1,1,VALID;3,1,1,VALID;2,2,1,VALID;3,1,1,VALID;3,1,1,VALID;3,1,1,VALID;2,2,1,VALID;3,1,1,VALID;3,1,1,VALID;3,1,1,VALID;2,1,1,VALID;3,1,2,VALID;3,1,2,VALID;3,1,2,VALID;3,1,1,VALID;4,1,4,VALID

