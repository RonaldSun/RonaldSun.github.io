---

title: Map感知系列总结
date: 2025-07-25 11:40:00 +0800
categories: [论文, map]
tags: [论文, map]
pin: true
---
## HDMapNet(2021 Tsinghua)

把图像和lidar都在bev空间进行encode、融合。然后直接在bev空间用FCN进行分割、instance embeding、方向的预测，最后通过一个后处理（聚类、NMS、greedily connecting），把地图要素恢复出来。

## VecotrMapNet（2023 Tsinghua）

BEV Encoder得到BEV features，然后通过DETR的方式，得到instance，文中定义了几种instance表示方式（例如bbox, start-mid-end等），然后再拿instance去refine出最后的element。refine的方式：transformer encoder输入所有的keypoint，然后decoder输出refine后的polyline，用一个EOS占位符表示结束来表达不同长度的polyline。

## MapTR（2023 HUST & Horizon）

采用了BEV的encoder（GKT）+ deformable detr（在BEV空间）的框架来检测地图要素，主要包括线状要素和多边形要素。

重点提出了Hierarchical matching的方式，解决地图要素表达上的歧义问题。具体做法：

- 对于GT，每个polyline有两种形态（正反向），每个polygon有2*k种形态，k为点集的数量（k种不同的起点，还有2种方向）
- 二分图匹配计算loss时，会从GT的形态里选择loss最小的那一种，匹配loss包含类别和点集loss两种
  
对于匹配使用的点集loss，做了实验对比了Chamfer distance和point2point distacne两种方式，结果表明point2point更好。point2point使用了曼哈顿距离计算方式。
另外，提出point2point没有考虑边的效果，还添加了一个edge loss，对每个edge的方向计算了余弦相似度loss。

MapTR的query分为N个instance query和M个point query，两者相加，每个instance都有M个query在BEV空间中进行deformable attention。

## MapTRv2（2024 HUST & Horizon）

1. self-attention在instance的纬度和point的纬度做了解耦，做两次attention，减少了复杂度，同时也使得模型更好训练，有一定的性能提升；
2. 研究了不同的cross-attention策略，包括BEV、PV、BEV+PV，结果是PV不如另外两种，尤其是当地图GT没有高度的情况下；
3. 参考了Group-DETR/CO-DETR的做法，在训练阶段用了one-to-many的匹配，把GT复制了K份，point query和decoder都是共享的；
4. 添加了几个辅助训练的Loss:
   - 图像空间的深度预测loss
   - bev空间的前景分割loss
   - pv空间的前景分割loss
5. 添加了道路拓扑（中心线）的预测，参考lane graph as path

## Mask2Map（2024 Hanyang University）

可以看成Mask2Former+MapTR的一个方案，对比MapTRv2有巨大的性能提升（10%+mAP），但是计算量也有所增加。

1. 把BEV feature升级成了multi-scale的形式；
2. 参考Mask2Former，每个instance通过一个learnable query生成mask，这一步被称为IMPNet（Instance-Level Mask Prediction Network）；
3. 计算instance的positional query(类似deformable detr中的reference point，2d检测中又称object query)：
   - 上一步生成的Mask取大于阈值的区域，把所有有效坐标的PE求均值得到q_pos；
   - 用N个learnable query和这个q_pos相加，每个instance对应N个位置query
4. Geometric Feature Extractor：用Mask来采样BEV feature
   - 用一个G*G的kernel不相交滑窗，保留mask最大位置的feature；
   - Farthest Point Sampling保留K个feature
5. Mask-Guided Map Decoder:
   - 用第3步的query去和第4步的feature做cross-attention，生成hybrid query
   - 再用hybrid query去和BEV feature做deformable cross attention
6. 借鉴DN-DETR，添加了DeNoising的步骤，对GT添加了一些噪声添加到query中加强模型的回归监督效果

## MapQR (2024 Shanghai Jiao Tong & Huixi tech)

1. Scatter-and-Gather Query: scatter指的是把instance query复制n份，和ref points的PE相加得到point query；gather指的是把所有的point query cat之后经过MLP得到新的instance query；
2. 先用instance query做self attention，scatter操作之后跟BEV feature做cross attention；做self attn的时候发现给instance加了pe没有提升效果，最后就没有加；
3. BEV encoder用了改进后的GKT算法，用query+linear预测高度，替换原来的固定高度。
