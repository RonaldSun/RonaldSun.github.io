---

title: BEV特征提取系列总结
date: 2025-05-02 17:34:36 +0800
categories: [论文, bev]
tags: [论文, bev]
pin: true
---
## LSS(2020 NVIDIA)

Lift, Splat, Shoot

- 对每个像素预测一个D维的深度分布，和一个context feature；
- 把深度的概率值和context feature相乘，把特征转换成3D空间中的点云，这一步叫"Lift"；
- 使用pointpillars的方式，对每个pillar中的特征进行相加的操作，得到BEV的feature，这一步成为"Splat"
- shoot相当于是motion planning的decoder
  
splat的过程中采用了sort+cumsum来避免padding导致的额外显存占用，这也是用sum pooling的原因。

## BEVFormer(2022 Shanghai AI lab)

与LSS从2D转换到3D特征不同，BEVFormer是直接从3D BEV特征作为出发点，投影到2D中去聚合特征的过程。

- BEV空间中的每个格子都是一个query，每个query会用N个高度值往图像上投影，然后使用deformable attention提取特征，投影得到的坐标就是reference point，N个特征加起来，不同view的特征取平均得到最终的特征；
- 对于时序的feature，会保留上一个时刻的BEV feature，然后通过ego的运动转到当前帧中，与当前的bev特征concat作为value做deformable attention，其中offset是由当前时刻和上一时刻concat的结果来算的，因为考虑到运动物体，前后两个时刻offset不是等价的。

需要注意的是，再图像中提取BEV特征时，是用bev query取预测2D空间中的offset，然后获取图像特征的，而不是获取3D offset，这一点文中没有进行深入讨论。

## GKT(2022 Horizon)

和BEVFormer的区别：

- 没有使用deformable attention，直接以投影点为中心取一个kernel的特征做attention；
- 投影的过程中对外参的旋转和平移部分增加了噪音，使得模型对于外参的抖动具有一定的鲁棒性；
- 计算了一个BEV-2D的查找表来进行加速特征查找的过程。

## CVT(2022 UT Austin)

和BEVFormer的区别：

- 对图像的2d feature做了特殊的postion encoding：用投影射线的方向作为position encoding；
- 没有使用Deformable attention，直接用cross attention来和加了PE的图像特征交互，提取bev的特征；
- 定义了一个新的基于余弦相似度的特征相似度计算公式，来做attention;

这篇文章的主要的好处是对相机内外参做了比较好的处理，把相机内外参的影响全都体现在position encoding里面了。

## BEVDepth(2023 MEGVII)

提出了LSS的问题主要在于缺少真正的depth监督过程，而是通过最终decoder的检测Loss来隐式的监督前面的depth预测模块，这样导致的问题：
  
1. 模型的深度预测精度远不如一些专门的深度预测模型；
2. 容易出现过拟合的现象，比如改变图像分辨率之后性能就会大幅下降；
3. BEV的语义预测不准确，会包含大量的噪声。

BEVDepth的改进：

1. 引入了lidar对深度进行显式的监督；
2. 预测深度的head加入了相机内外惨的编码；
3. 在深度*图像宽度的维度（可以理解为不同高度的BEV平面），添加了一些3*3的conv层，使得在深度预测没那么准的情况下，在splat之前有一个深度信息交互的过程，这一步称为"Depth Refine"

## BEVFormer v2(2023 Sensetime)

- 提出BEVFormer一个显著的问题是对于backbone的监督太差了，尤其是使用了deformable attention之后，梯度从decoder传递到encoder的过程中会损失信息；
- 主要的改进是加了一个FCOS3D的decoder对encoder进行监督，得到更好的backbone；
- FCOS3D的结果经过后处理之后会送到decoder中作为补充的query
- 还指出了原始BEVFormer使用recurrent的模式提取时序信息，对于长时间的信息容易丢失，因此用了一个时间窗口的特征来做加强。

在不依赖lidar点云做深度监督的情况下，加强了backbone的能力。
