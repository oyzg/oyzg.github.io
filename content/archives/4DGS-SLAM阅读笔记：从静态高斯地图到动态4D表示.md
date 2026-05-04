---
title: "4DGS-SLAM阅读笔记：从静态高斯地图到动态4D表示"
date: 2025-04-25T20:00:00+08:00
draft: false
categories: [SLAM]
series: [SLAM学习笔记]
tags: [SLAM, 4DGS, 3DGS, Dynamic SLAM, Gaussian Splatting]
summary: "阅读4D Gaussian Splatting SLAM，理解动态场景中为什么需要从静态3D高斯地图扩展到4D高斯表示。"
---

*4D Gaussian Splatting SLAM* 提交于 2025-03-20。先区分两个名字相近的工作：这里主要记录 Yanyan Li 等人的 *4D Gaussian Splatting SLAM*；另一个相关工作是 *Embracing Dynamics: Dynamics-aware 4D Gaussian Splatting SLAM*，也常被简称为 D4DGS-SLAM，提交于 2025-04-07。两者都关注 4DGS 和动态 SLAM，但具体系统设计不同。

很多动态 SLAM 方法会把动态物体当成干扰，先识别再过滤。4DGS-SLAM 的路线更进一步：动态物体不一定只是干扰，它们本来就是场景的一部分。如果希望理解真实世界，就不能永远只建静态背景，也要尝试建模随时间变化的动态场景。

## 为什么需要4D表示

普通 3DGS 表示的是静态三维场景。每个 Gaussian 有固定的三维位置、形状、颜色和透明度。相机从不同视角看它时，渲染结果应该一致。这个假设在静态房间里没问题，但真实世界显然不是这样。

动态场景破坏了这个假设。例如一个人走过房间，同一个时间段内他的空间位置不断变化。如果仍然用静态 Gaussian 表示，就会出现两种选择：

- 把人当作动态干扰删除，只保留静态背景。
- 把不同时间的人都写进静态地图，产生重影和伪影。

第一种适合定位和静态建图，但丢掉了动态对象信息。第二种会污染地图。4DGS-SLAM 想走第三条路线：显式建模时间，让 Gaussian 不只存在于三维空间，也能表达随时间变化的动态辐射场。

这里的“4D”：空间三维加时间一维。系统要同时估计相机位姿和动态场景表示。

## 系统目标

这篇 4DGS-SLAM 使用 RGB-D 图像序列作为输入。相比单目，RGB-D 提供了深度观测，可以降低动态建图中的几何不确定性。系统目标包括：

- 增量估计相机位姿。
- 构建静态和动态场景的 Gaussian radiance field。
- 对动态物体运动进行建模。
- 在动态真实环境中实现较好的 tracking 和 view synthesis。

它和 WildGS-SLAM 最大的区别是目标不同。WildGS-SLAM 更关注在动态环境中稳定构建静态 Gaussian 地图；4DGS-SLAM 则尝试把动态对象也建进表示中。一个偏“过滤干扰”，一个偏“拥抱动态”。

## 静态和动态先验

系统首先会生成 motion masks，为每个像素提供静态或动态先验。这个步骤很关键，因为后续需要知道哪些观测应该进入静态 Gaussian，哪些观测应该进入动态 Gaussian。

如果一个区域被判断为静态，它可以像普通 3DGS-SLAM 一样进入静态地图。静态 Gaussian 在时间上保持不变，主要表达背景和固定物体。

如果一个区域被判断为动态，它就不适合直接写进静态地图。系统会把相关 Gaussian 归入动态集合，并通过额外的运动建模描述它们随时间的变化。

这种 static / dynamic 分离，是动态 SLAM 中非常常见的思路。区别在于，传统动态 SLAM 通常只把动态点剔除，而这里会继续为动态点建立可渲染表示。

## 动态Gaussian集合

论文中把 Gaussian primitives 分成 static Gaussian set 和 dynamic Gaussian set。静态集合负责稳定背景，动态集合负责运动物体。

动态 Gaussian 不能只有一个固定位置。它需要一个时间相关的变换场，描述从某个规范空间或参考状态到当前时间的形变。论文使用 sparse control points 和 MLP 来建模 dynamic Gaussians 的 transformation fields。

我把它理解为：动态物体不是每一帧重新建一堆完全独立的 Gaussian，而是让一组 Gaussian 通过时间相关变换移动。这样可以利用跨时间的连续性，避免每一帧都从零开始重建。

这个设计的意义在于保持时间一致性。如果不同帧的动态物体表示完全独立，渲染可能每帧都不错，但无法形成稳定的动态场景模型。

## 光流监督

为了更准确学习动态 Gaussian 的运动，论文提出了 2D optical flow map reconstruction。系统会渲染相邻图像之间动态物体的 optical flow，并用它来监督 4D Gaussian radiance fields。

光流在动态场景中非常重要。颜色重建只能说明某一帧看起来像不像，但不能充分约束物体在时间上的运动。深度提供几何，光流提供图像平面上的运动对应。把光流加入监督，可以帮助动态 Gaussian 学到更合理的时序变化。

这点对我理解 4DGS-SLAM 很有帮助：4D 表示不是简单给 Gaussian 加一个时间戳，而是要用跨帧运动线索约束它。否则动态部分很容易变成逐帧拟合，看起来每一帧都能解释，但整体没有连续性。

## Tracking和Mapping的耦合

动态 SLAM 里 tracking 和 mapping 更难分开。相机在动，物体也在动。图像变化既可能来自相机运动，也可能来自动态物体运动。系统必须判断哪些变化应该解释为相机位姿变化，哪些变化应该解释为物体运动。

如果动态物体占据画面大部分区域，tracking 很容易被带偏。因此即使 4DGS-SLAM 要建模动态物体，tracking 仍然需要稳定静态区域作为主要参考。动态区域可以用于动态建模，但不能无条件用于相机位姿估计。

这和 WildGS-SLAM 的思想并不矛盾。可以理解为：

- tracking 更依赖静态、可靠区域。
- mapping 同时维护静态背景和动态对象。
- 动态对象通过额外时间模型优化，而不是写进静态地图。

## 和3DGS-SLAM的区别

普通 3DGS-SLAM 的优化变量主要是相机位姿和静态 Gaussian 参数。4DGS-SLAM 需要额外处理：

- 动态/静态分离。
- 动态 Gaussian 的时间变换。
- 光流、几何、颜色等多种监督。
- 动态区域对位姿估计的影响。
- 不同时间帧之间的一致性。

因此 4DGS-SLAM 的系统复杂度明显更高。它不仅是“把 3DGS 换成 4DGS”，而是整个 SLAM 问题的假设变了：地图不再是静态世界，而是随时间变化的世界。

## 和D4DGS-SLAM的关系

2025-04 的 *Embracing Dynamics: Dynamics-aware 4D Gaussian Splatting SLAM* 也是相关工作。它强调通过 temporal dimension 表示动态场景，并使用 dynamics-aware InfoModule 获取场景点的动态性、可见性和可靠性，从而过滤不稳定动态点用于 tracking，并对不同动态特性的 Gaussian 使用不同正则。

一个明显趋势：动态 Gaussian SLAM 不再满足于 mask 掉动态物体，而是开始研究如何利用动态信息本身。不同论文的具体做法不同，但共同问题差不多是这些：

- 哪些点适合跟踪？
- 哪些点应该进入静态地图？
- 哪些点需要动态表示？
- 时间维度如何约束 Gaussian？
- 动态建图如何不破坏实时性？

## 对研究方向的启发

研究动态 3DGS-SLAM，可以把问题分成三个层次。

第一层是 robust tracking。无论是否建模动态物体，相机位姿必须稳定。动态区域不能随意参与位姿优化。

第二层是 static map quality。系统需要构建干净的静态背景 Gaussian 地图，否则后续定位和渲染都会受影响。

第三层是 dynamic object modeling。只有前两层足够稳定后，才适合进一步建模动态对象。否则动态建模可能只是把跟踪误差和深度误差吸收到运动模型里。

4DGS-SLAM 直接面对第三层问题，因此它比动态剔除路线更难，但也更有潜力。

## 局限和思考

4DGS-SLAM 的难点也很明显。

首先，它依赖 RGB-D 输入。深度能显著降低几何不确定性，但也限制了应用范围。如果换成单目 RGB，尺度、深度和动态运动会更难解耦。

其次，动态/静态分割质量非常关键。motion mask 错误会导致动态对象进入静态地图，或者静态背景被错误建成动态对象。

再次，动态 Gaussian 的运动模型可能很难覆盖复杂非刚体运动。人、衣服、反光物体、遮挡和拓扑变化都会带来挑战。

最后，计算量和内存压力会更大。4D 表示比 3D 静态表示多了时间维度和运动参数，如何保持实时或准实时是工程关键。

## 记录

4DGS-SLAM 的核心意义在于把动态物体从“需要删除的干扰”变成“需要建模的场景组成部分”。它通过 motion mask 区分静态和动态区域，用静态 Gaussian 表达背景，用动态 Gaussian 和变换场表达运动对象，并通过颜色、几何和光流约束优化 4D radiance field。

从理解难度看，可以先掌握传统 SLAM 的 tracking / mapping / BA，再理解 3DGS 的显式可微渲染，接着看动态场景中如何过滤干扰，最后再看 4DGS-SLAM 如何建模动态对象。这样每一步在解决什么问题会比较清楚。

## 参考

- [4D Gaussian Splatting SLAM](https://arxiv.org/abs/2503.16710)
- [Embracing Dynamics: Dynamics-aware 4D Gaussian Splatting SLAM](https://arxiv.org/abs/2504.04844)
- [WildGS-SLAM: Monocular Gaussian Splatting SLAM in Dynamic Environments](https://arxiv.org/abs/2504.03886)
- [3D Gaussian Splatting for Real-Time Radiance Field Rendering](https://arxiv.org/abs/2308.04079)
