---

title: DETR系列总结
date: 2025-04-22 19:51:23 +0800
categories: [论文, detr]
tags: [论文, detr]
pin: true
---

## DETR (2020 Facebook AI)

在训练的阶段，利用和GT的二分图匹配，使得最终模型不需要使用NMS，同时因 为有learnable query，也不需要显式的anchor生成步骤。

需要注意的点：

1. 预测的bbox和GT匹配时，既考虑了bbox的loss，也考虑了类别的loss；但是匹配用的Loss和监督Loss不完全一致；
2. decoder中添加了辅助loss，具体来说，decoder的每一层都会进行预测、和gt匹配、计算loss的过程，其中预测头FFN是共享的。

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

![deformable](/assets/img/detr_summary/deformable_detr_ablation.jpg)

在COCO数据集上，原始DETR需要500个epoch，deformable DETR只需要50个epoch。

## Conditional DETR(2021 USTC)

出发点也是为了解决DETR训练收敛太慢的问题，本文观察到DETR训练了50个epoch后，spatial attention（就是position embeding之间的attention）还是不够好，训练500以后才会比较好，并且最后预测时，即使去掉Postion embeding对最终的指标影响也不大，说明模型没有很好的利用空间信息。

![conditional](/assets/img/detr_summary/conditional_detr.png)

提出了Conditional attention:

1. 每个query分为两个部分，一个是content query, 包括语义、尺寸，另一个是object query, 可以理解为position embeding，包含bbox中心点信息；
2. 计算attention时，content query和图像特征做atttn，object query和图像的position embeding单独做attn，两部分加起来作为最终的attention matrix;
3. object query的计算过程是同时融合了上一个decoder的content query和object query，这样objection query中就包含了四个边界的位置信息。

根据COCO实验结果来看，Conditional DETR只需要50个epoch就可以接近收敛，原始DETR需要500个epoch。

## Anchor DETR（2022 megvii）

1. 把原始DETR的query换成了显式的postion anchor，包含两种初始化策略：
  a. 在图像空间均匀分布
  b. 在图像空间随机撒点

2. 直接用这些position anchor进行编码，用来当作object query，另外，定义了3个pattern query附加在原来的position query上，使得同一个位置可以有多个检测结果；

3. 简化cross attention的计算量，把图像空间的attention拆分为横向和纵向，减少计算量。

最终效果也是提升了收敛速度，只需要50个epoch
![conditional](/assets/img/detr_summary/anchor_detr.png)

## DAB-DETR (2022 Tsinghua)

![DAB_detr](/assets/img/detr_summary/dab_detr_cmp.png)

DAB:Dynamic anchor boxes

对DETR训练收敛慢的问题做了一些分析：首先可以确定的是，导致DETR收敛慢的原因是cross attention，所以有两种猜想：

1. query的embeding特征太难学了；
2. query的position encoding的方式和image特征不一致。

针对第一个问题，作者直接用训练好的DETR query特征，fix住开始训练，但是结果也是收敛很慢，说明不是query特征不好学的问题；所以问题还是出在postion信息上面。

核心思想是在condition detr的基础上，进一步考虑bbox尺寸对postion attention的影响，object query中除了中心点也把尺寸给加上了：
![DAB_detr_math1](/assets/img/detr_summary/dab_detr_math.png)

应该算是condition detr的改进，相同模型结构下提升1~2%的mAP。