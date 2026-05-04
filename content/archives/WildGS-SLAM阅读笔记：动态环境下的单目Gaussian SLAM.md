---
title: "WildGS-SLAM阅读笔记：动态环境下的单目Gaussian SLAM"
date: 2025-04-12T20:00:00+08:00
draft: false
categories: [SLAM]
series: [SLAM学习笔记]
tags: [SLAM, 3DGS, WildGS-SLAM, Dynamic SLAM]
summary: "阅读WildGS-SLAM，理解它如何用不确定性建模和DINOv2特征处理动态环境下的单目Gaussian SLAM。"
---

# WildGS-SLAM阅读笔记：动态环境下的单目Gaussian SLAM

WildGS-SLAM 提交于 2025-04-04，题目是 *WildGS-SLAM: Monocular Gaussian Splatting SLAM in Dynamic Environments*。这篇论文的背景很好理解：3DGS-SLAM 在静态场景里已经能做比较好的跟踪、建图和渲染，但真实环境往往有动态物体。如果系统还假设整个场景都是静止的，动态物体会同时破坏位姿估计和 Gaussian 地图。

WildGS-SLAM 的目标是做一个面向动态环境的单目 RGB Gaussian SLAM 系统。两个关键词：单目和动态。单目意味着没有真实深度传感器，尺度和几何更难；动态意味着有移动物体，会产生不满足静态世界假设的观测。它的核心思路是引入 uncertainty-aware geometric mapping，用不确定性指导 tracking 和 mapping 中的动态区域处理。

## 问题背景

普通 3DGS-SLAM 通常依赖静态场景假设。相机移动时，同一个空间点在不同帧中应该保持一致。这个假设在室内静态场景里还好，但一旦画面里有人走动、车经过，就很容易出问题。

动态物体会带来几类问题：

- 跟踪错误：系统把动态物体当作静态背景，位姿会被错误约束拉偏。
- 地图污染：移动物体被写进静态 Gaussian 地图，后续视角会出现重影或漂浮物。
- 深度错误：单目深度估计在动态区域可能不稳定，影响 Gaussian 初始化。
- 渲染伪影：动态物体在不同帧位置不同，静态地图无法同时解释这些观测。

传统动态 SLAM 常用语义分割、光流、几何一致性等方式剔除动态物体。WildGS-SLAM 的特别之处在于，它不是只做一个硬 mask，而是用不确定性来指导几何建图。

## 系统输入和目标

WildGS-SLAM 使用的是 monocular RGB 输入，也就是普通单目视频。相比 RGB-D 动态 SLAM，它不能直接依赖传感器深度。系统需要从图像中估计深度、位姿和 Gaussian 地图，同时识别哪些区域不可靠。

输出包括：

- 相机轨迹。
- 3D Gaussian 地图。
- 可用于新视角合成的渲染结果。
- 对动态或不可靠区域的过滤结果。

论文强调的不只是 tracking accuracy，也包括 mapping 和 rendering quality。对于 Gaussian SLAM 来说，这三件事是耦合的：位姿影响地图，地图影响渲染，渲染误差又反过来影响优化。

## Uncertainty-aware geometric mapping

WildGS-SLAM 的核心是 uncertainty-aware geometric mapping。直觉上，不是所有像素都应该同等相信。静态背景、纹理稳定、深度可信的区域可以作为 tracking 和 mapping 的强约束；动态物体、遮挡边界、深度不准区域则应该降低权重或剔除。

系统会预测一个 uncertainty map，用来描述图像中不同区域的不确定性。这个不确定性不是简单的语义类别，而是结合了几何和视觉特征后得到的可靠性判断。

有了 uncertainty map 后，它可以参与两个地方：

- tracking：位姿估计时少相信高不确定区域，避免动态物体拉偏相机位姿。
- mapping：Gaussian 优化和插入时减少动态区域贡献，避免污染静态地图。

这种方式比简单语义 mask 更灵活。因为“人”和“车”通常是动态类别，但它们也可能静止；而某些非典型物体也可能运动。只依赖语义类别会太粗糙。不确定性建模更接近 SLAM 真正关心的问题：这个观测到底适不适合拿来约束当前静态地图？

## DINOv2特征和浅层MLP

论文中提到，uncertainty map 由 DINOv2 features 和一个 shallow MLP 预测。DINOv2 是一种强视觉表征模型，它能提供比传统局部特征更丰富的语义和结构信息。

动态区域和不可靠区域不一定只靠低层像素差异能判断。比如一个人站在背景前，颜色和边缘可能很清楚，但它不应该被写进静态地图。DINOv2 特征能提供更高层的视觉上下文，帮助系统判断区域可靠性。

浅层 MLP 则负责把这些特征映射成不确定性估计。相比端到端训练一个庞大模块，这种设计更轻量，也更容易嵌入 SLAM pipeline。

## Tracking中的作用

在 tracking 中，系统需要估计当前相机位姿。普通 photometric tracking 会最小化当前图像和渲染图像之间的颜色差。如果动态物体出现在当前帧，而地图里没有对应静态结构，颜色残差就会很大。直接优化这些残差会让位姿向错误方向移动。

WildGS-SLAM 的做法是利用 uncertainty map 调整这些残差的影响。高不确定区域对位姿优化贡献小，低不确定区域贡献大。这样位姿主要由稳定背景约束。

这和传统鲁棒核函数有相似直觉，但粒度更细，也更有语义和几何先验。鲁棒核通常根据残差大小降低外点权重，而 uncertainty map 可以在优化前就预判哪些区域不可靠。

## Mapping中的作用

Mapping 阶段更容易被动态物体污染。假设一个人从画面中走过，如果系统把这个人对应的像素反投影成 Gaussian 并写进地图，那么当人走开后，地图里仍然会残留这个人形结构。后续渲染就会出现 ghost artifact。

WildGS-SLAM 在 Gaussian map optimization 中使用不确定性来引导动态物体移除。高不确定区域不会被强行解释为静态 Gaussian，或者其优化权重会降低。这样地图更倾向于表达稳定场景。

这对单目系统尤其重要。RGB-D 系统至少有深度传感器提供几何约束，单目系统的几何本来就更脆弱。如果动态区域被错误写入，会进一步破坏尺度和结构。

## 和传统动态SLAM的区别

传统动态 SLAM 经常走两条路线：

- 语义分割：检测人、车等动态类别，然后剔除。
- 几何一致性：通过多视图约束或光流判断哪些点不符合静态模型。

WildGS-SLAM 的方法更适合 Gaussian SLAM，因为它不仅要做 tracking，还要做 photorealistic mapping。对 Gaussian 地图而言，动态区域不只是外点，还会影响渲染质量。简单删除可能造成空洞，完全保留又会产生伪影。因此用 uncertainty 软性调节，比二值 mask 更自然。

当然，这并不意味着语义或几何方法不重要。WildGS-SLAM 更像是把视觉先验、深度估计、不确定性和 Gaussian 优化放到同一个系统中。

## 对动态3DGS-SLAM的启发

WildGS-SLAM 给我最大的启发是：动态场景处理不一定一上来就要做完整 4D 建模。对于很多机器人定位和静态地图重建任务，把动态物体当作干扰剔除，反而是更实际的路线。

这个思路可以分成三层：

1. 找到不可靠区域：通过 DINOv2、深度、不确定性、光流或语义分割。
2. 在 tracking 中降权：避免动态区域影响位姿。
3. 在 mapping 中过滤：避免动态区域污染 Gaussian 地图。

如果系统目标是构建稳定静态地图，这条路线很合理。如果目标是理解动态物体本身，比如重建人的运动或车辆轨迹，就需要 4DGS-SLAM 这类动态建模方法。

## 局限和思考

需要注意的点：

第一，单目输入的尺度和深度仍然是难点。不确定性可以帮助判断可靠区域，但不能完全解决单目几何本身的不可观问题。

第二，对 DINOv2 特征和 MLP 预测质量有依赖。如果场景和训练分布差异很大，不确定性估计可能失效。

第三，它主要还是倾向于构建静态地图。动态物体被过滤掉后，系统不会真正重建动态对象的时间变化。这对定位很有用，但对完整动态场景理解还不够。

第四，不确定性如何和 Gaussian densification、pruning、opacity 更新协同，是工程上很关键的问题。如果过滤太强，地图可能缺失；过滤太弱，又会有伪影。

## 记录

WildGS-SLAM 的核心思想：在动态单目场景中，不要平等相信所有像素，而是用 uncertainty-aware geometric mapping 区分可靠和不可靠区域。可靠区域用于跟踪和建图，不可靠区域在 tracking 和 Gaussian 优化中被降权或过滤。

它没有直接跳到复杂的 4D 表示，而是先解决一个更基础也更实用的问题：怎样在真实动态环境里构建稳定的静态 Gaussian 地图。

## 参考

- [WildGS-SLAM: Monocular Gaussian Splatting SLAM in Dynamic Environments](https://arxiv.org/abs/2504.03886)
- [3D Gaussian Splatting for Real-Time Radiance Field Rendering](https://arxiv.org/abs/2308.04079)
- [DROID-SLAM: Deep Visual SLAM for Monocular, Stereo, and RGB-D Cameras](https://arxiv.org/abs/2108.10869)
