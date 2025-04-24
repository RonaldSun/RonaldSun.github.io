---

title: MaskFormer & Mask2Former
date: 2025-04-24 20:31:33 +0800
categories: [论文, segmentation]
tags: [论文, segmentation]
pin: true
---

## MaskFormer (2021 Facebook AI)

Per-Pixel Classification is Not All You Need for Semantic Segmentation.

本文为实例分割和语义分割提出了一个统一的框架：mask预测。可以看成是DETR在分割领域的姊妹篇。

- 图像先下采样encode，然后用一些learnable query（instance query）去提取图像特征，得到的特征经过class head得到类别信息；
- 上一步的query通过mlp，和上采样之后的图像特征做dot，然后通过sigmoid激活得到mask图层；
- 上一步的mask图层与padding后的gt进行二分图匹配，然后计算dice loss

inference的方法：

1. 对于一般的情况，特别是实例分割、全景分割，用class概率乘以mask概率，最大的那个就是pixel所属的类别；
2. 对于语义分割，不是取最大的，而是取所有mask求和最大的那个类别，这么做是因为每个像素都属于某一个类别；

需要注意，同一个像素在不同instance之间没有用softmax。

## Mask2Former（2022 Facebook AI）

1. 把mask运用到了attention中，只和前景做cross attention，加快模型的收敛；
2. attention加入了FPN的相关机制
3. 做了一些效率优化，针对匹配和最终的训练loss，加入了采样mask的策略，使得计算量降低。


