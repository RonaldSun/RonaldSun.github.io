---

title: CornerNet & CenterNet
date: 2025-05-05 00:41:26 +0800
categories: [论文, 2D, detection]
tags: [论文, 2D, detection]
pin: true
---

## CornerNet(2018 UT Austin)

通过识别corner、匹配corner来完成bbox的检测，避免了anchor的生成步骤。跟这篇论人体关键点检测的论文几乎是一样的方法：Associative embedding: End-to-end learning for joint detection and grouping。

- 使用hourglass网络作为backbone，结构上看就是FPN的最后一层进行多次堆叠；
- 分别预测top left和bottom right的corner heatmap，同时还会预测一个offset弥补精度；
- GT heatmap是由高斯分布生成的，sigma取半径的1/3;
- 提出了Corner pooling（连接在hourglass之后，head之前），对于top left的feature，每个像素坐标会对右边和下方的所有点取max pooling，bottom right同理；
- 还会预测一个Embedding的图层，添加Pull&push loss，把属于一个bbox的embeding靠近两者的均值，不属于同一个bbox的embeding远离均值；
- inference时，在corner heatmap上用了一个3*3的maxpooling，取topk个corner，进入下一步匹配，匹配就是取embeding距离小于0.5的类别一样的点对。

## CenterNet(2019 Huawei)

指出了CornerNet的问题：对于object的整体信息提取不够完整，过于关注物体的边缘信息，导致有一些明显错误的bbox检测结果。提出再增加一个center point的heatmap预测来加强推理过程。

实际预测的时候就是在CornerNet的基础上加了一个后处理，判断bbox的中心是否有center point，如果没有就过滤掉。

进一步改进了corner pooling:

1. center pooling：每个像素在横纵向都取max pooling;
2. cascade corner pooling: 对于一个corner feature，先横向找max，再在最大值的位置往纵向找max，两个max相加。