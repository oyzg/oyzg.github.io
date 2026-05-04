---
title: "DROID-SLAM阅读笔记：Dense Bundle Adjustment与深度SLAM"
date: 2024-11-10T20:00:00+08:00
draft: false
categories: [SLAM]
series: [SLAM学习笔记]
tags: [SLAM, DROID-SLAM, 深度学习, Bundle Adjustment]
summary: "阅读DROID-SLAM，理解它如何用循环更新和Dense Bundle Adjustment统一单目、双目和RGB-D视觉SLAM。"
---

# DROID-SLAM阅读笔记：Dense Bundle Adjustment与深度SLAM

DROID-SLAM 论文提交于 2021-08，题目是 *DROID-SLAM: Deep Visual SLAM for Monocular, Stereo, and RGB-D Cameras*。它把深度学习和几何优化结合得比较紧，不是简单地用网络直接回归位姿，而是把 Dense Bundle Adjustment 放进了系统里。

传统特征点法 SLAM 依赖特征提取、匹配、PnP、BA、回环等模块。DROID-SLAM 没有把几何全扔掉，而是把传统几何优化里的核心结构保留下来，再用神经网络学习更新规则，并把优化做得更稠密。

## 核心问题

DROID-SLAM 想解决的问题：能不能设计一个统一的深度 SLAM 系统，在单目、双目、RGB-D 输入下都能工作，并且比传统方法更鲁棒？

传统 SLAM 的失败经常来自前端匹配不稳定，例如弱纹理、运动模糊、光照变化、特征点不足。直接法虽然利用更多像素，但对光度一致性和初始化敏感。深度学习方法可以学习更鲁棒的图像匹配和几何更新，但如果完全抛弃几何约束，又容易缺少可解释性和泛化能力。

DROID-SLAM 的特点在于它没有简单地让网络直接回归位姿，而是把问题组织成一个反复迭代的优化过程。网络预测更新量，Dense Bundle Adjustment 层执行几何一致性更新。这样既利用了学习特征，又保留了 BA 的结构。

## 系统变量

DROID-SLAM 维护的核心变量包括：

- 相机位姿：每个关键帧或活动帧的 SE(3) 位姿。
- 像素级深度：不是稀疏地图点，而是较稠密的 depth。
- 图像特征：用于建立帧间 correspondence。
- 帧图结构：决定哪些帧之间建立约束。

和传统稀疏 BA 不同，DROID-SLAM 的优化对象更稠密。它不是只在几百个或几千个特征点上做重投影，而是在更密集的像素或特征网格上估计几何关系。

这也是它和 ORB-SLAM 这类系统最直观的区别：ORB-SLAM 通过稀疏特征点表达观测，DROID-SLAM 通过 learned dense correspondence 和 dense depth 表达观测。

## Recurrent update

DROID-SLAM 的一个核心设计是 recurrent iterative update。可以把它理解为：系统不是一次性预测最终位姿和深度，而是反复查看当前估计和图像证据之间的差异，然后逐步修正。

这个过程和 RAFT 光流方法有相似直觉。RAFT 通过 correlation volume 和循环更新逐步优化光流，DROID-SLAM 则把类似思想扩展到 SLAM 中的位姿和深度估计。

每次迭代中，系统会根据当前位姿和深度，把一个帧中的像素投影到另一个帧，查询图像特征之间的相关性，得到当前几何估计的残差信息。网络根据这些残差和隐藏状态预测修正量，随后通过 Dense BA 更新位姿和深度。

关键是“迭代”。单次预测不一定准，但反复修正可以逐步接近几何一致的结果。传统 SLAM 里的非线性优化也是这个味道，只是 DROID-SLAM 把一部分更新策略交给网络学习。

## Correlation volume

为了判断两帧之间哪些像素对应，DROID-SLAM 使用特征相关性。图像经过特征网络后，每个像素或网格位置都有一个特征向量。不同帧之间可以计算特征相关性，形成 correlation volume。

如果当前位姿和深度估计正确，一个像素投影到另一帧后，应该落在特征相似的位置。反过来，如果投影位置附近的相关性分布显示更好的匹配在别处，就说明当前几何估计需要调整。

这和传统特征匹配有相似目标，但方式不同。传统方法先离散地找特征点和匹配关系，再把匹配关系交给几何优化。DROID-SLAM 则用稠密特征相关性为优化提供连续的匹配证据。

## Dense Bundle Adjustment

DROID-SLAM 最重要的模块是 Dense Bundle Adjustment。传统 BA 最小化的是稀疏特征点的重投影误差，而 Dense BA 使用更密集的像素级或网格级约束，同时优化相机位姿和深度。

直觉上，Dense BA 做的是：当前系统认为某个像素有一个深度，在当前帧位姿下它对应空间中的一个三维点。把这个点投影到其他帧，应当和其他帧中的对应观测一致。如果不一致，就需要调整位姿和深度。

DROID-SLAM 的 Dense BA 是可微的，可以放进神经网络训练流程中。这一点很重要。它不是在网络外部单独跑一个传统优化器，而是成为模型结构的一部分，使网络能学习如何产生适合几何优化的更新。

这也是它比简单深度回归更像 SLAM 的地方：SLAM 的核心不是单帧深度估计，而是多帧之间的几何一致性。

## 单目、双目和RGB-D

DROID-SLAM 的另一个亮点是统一多种输入形式。论文强调，虽然训练时使用单目视频，但测试时可以利用双目或 RGB-D 输入获得更好效果。

单目输入下，尺度本身存在不确定性。系统只能从视频运动中恢复相对尺度。双目输入提供左右相机基线，可以约束真实尺度。RGB-D 输入直接提供深度观测，也能增强深度估计稳定性。

从工程角度看，这种统一性很有价值。很多传统系统会为单目、双目、RGB-D 分别设计不同流程，而 DROID-SLAM 试图用同一套 recurrent update + Dense BA 处理不同传感器条件。

## 和传统SLAM的关系

DROID-SLAM 并不是完全抛弃传统 SLAM，更像是把传统 SLAM 中最核心的两个部分保留下来：

- 多视图几何一致性。
- Bundle Adjustment。

然后用深度网络替代或增强前端 correspondence 和优化更新。传统 SLAM 里，匹配关系主要由手工特征和几何筛选得到；DROID-SLAM 里，匹配证据来自 learned feature correlation。传统 BA 需要手写残差和求解流程；DROID-SLAM 把 dense BA 做成可微模块，并和网络循环更新结合。

所以我学习 DROID-SLAM 时，不想把它看成一个“黑盒位姿回归网络”。它本质上仍然是几何优化，只是优化过程被学习方法增强了。

## 对3DGS-SLAM的意义

3DGS-SLAM 的一个关键难点是 tracking。如果相机位姿不准，Gaussian 地图很容易被写坏。尤其是单目 3DGS-SLAM，既要估计尺度和深度，又要优化地图外观，问题非常耦合。

DROID-SLAM 给了一个强前端思路：先用学习式稠密 SLAM 得到比较稳的位姿和深度，再把这些结果接到 Gaussian mapping 或 rendering 模块中。后来的 DROID-Splat 就是类似方向：用 end-to-end tracker 提供强 tracking，再用 3DGS renderer 提升地图表达和渲染质量。

从系统设计看，DROID-SLAM 可以作为 3DGS-SLAM 的 tracking backbone，也可以作为初始化器。它估计的深度和位姿能为 Gaussian 初始化提供几何基础，减少纯 photometric optimization 的不稳定性。

## 局限

DROID-SLAM 虽然鲁棒，但也有局限。

第一，它依赖训练数据和网络泛化。遇到和训练分布差异很大的场景，性能可能下降。

第二，稠密优化带来更高算力需求。相比 ORB-SLAM 这类 CPU 友好的系统，DROID-SLAM 更依赖 GPU。

第三，它的地图表达并不是为了高质量渲染设计的。DROID-SLAM 更关注位姿和深度估计，而不是生成 photorealistic map。如果想要更好的地图外观表达，还需要接入其他表示方式。

第四，动态场景仍然是挑战。学习式 correspondence 可能更鲁棒，但如果动态物体占据大量视野，多视图一致性仍然会被破坏。WildGS-SLAM 和 4DGS-SLAM 继续处理的就是这个问题。

## 记录

DROID-SLAM 的核心可以概括成三件事：用 learned dense correspondence 提供匹配证据，用 recurrent update 逐步修正估计，用 Dense Bundle Adjustment 保持几何一致性。它不是简单的端到端位姿回归，而是学习和几何优化结合的 SLAM 系统。

DROID-SLAM 对我最大的启发主要在 tracking。高质量地图依赖稳定位姿，而 DROID-SLAM 这样的系统提供了一条强跟踪路线。理解这一点后，再设计 SLAM 系统时，就能更清楚地区分 tracking 和 mapping 的职责。

## 参考

- [DROID-SLAM: Deep Visual SLAM for Monocular, Stereo, and RGB-D Cameras](https://arxiv.org/abs/2108.10869)
- [RAFT: Recurrent All-Pairs Field Transforms for Optical Flow](https://arxiv.org/abs/2003.12039)
- [DROID-Splat: Combining end-to-end SLAM with 3D Gaussian Splatting](https://arxiv.org/abs/2411.17660)
