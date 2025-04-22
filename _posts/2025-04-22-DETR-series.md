---

title: DETR系列总结
date: 2025-04-22 19:51:23 +0800
categories: [论文, detr]
tags: [论文, detr]
img_path: /assets/img/detr_summary/
pin: true
---

## Deformable DETR(2021 SenseTime)

原始的DETR有以下问题：

  1. 没有利用多尺度的FPN图像特征
  2. 由于每个query需要attention所有的patch特征，收敛速度和特征分辨率都受到了限制

本文的改进方法受到Deformable Conv的启发，每个query只与固定数量的稀疏的图像特征进行交互，具体做法：

- 每个query对应一个content feature和一个reference point (2d)，其中content feature是用随机初始化的，reference point是由content feature通过linear projection+sigmoid得到;
- 通过linear proj得到N个offset坐标，每个坐标的attention weight也是通过linear proj + softmax得到;
- ref point加上offset坐标，在特征空间做双线性插值，得到keys' set

需要注意，deformable attention中并没所有使用key * query，而是直接用query+线性层得到attention矩阵。

decoder中，每一层会回归bbox的offset，更新ref point的位置输入到下一层。

另外还提出了Two-Stage Deformable DETR，强调query的分布应该是和图像相关的：

在第一阶段，Encoder输出的FPN中的每个pixel都作为query直接预测一个bbox，取topk的bbox和对应的content feature，作为后续decoder的query。训练的时候应该还是一起训练的，第一阶段也会和gt做二分图匹配。

![deformable](deformable_detr_ablation.jpg)
