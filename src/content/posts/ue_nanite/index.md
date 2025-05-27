---
title: UE5/Nanite
published: 2025-05-24
description: ''
image: ''
tags: [ue5, rendering, graphics, nanite, geometry]
category: 'UE5'
draft: true
lang: 'zh'
---

# Introduction
Nanite可以算是UE5最重要的几个新特性之一，可以使得开发者可以使用高精度的几何体进行渲染，而不需要担心性能问题。

关于Nanite想要做的事可以从Karis的分享中了解到：
![20250526230259](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250526230259.png)

如果用一句话总结就是做Virtual Geometry，如果用Virtual Teture来类比，VT技术使得我们可以将高精度的纹理用到我们的渲染中，其核心原理就是只加载跟只用我们需要用到的部分，这其实也是各种Virtual技术最朴素的原理，回到Geometry上，Nanite的核心原理也是如此，当我们渲染一个高精度的模型时，其实只有当这个模型拉得非常近时，我们才需要模型的细节，而距离比较远时，很多三角面已经远小于像素了，其渲染对于画面效果影响很小，但依然占用了性能，这是一种非常大的浪费，在Nanite之前我们可以通过制作多级LOD并通过渲染距离或者屏占比来切换，但这种方式有很多缺点：
* 需要制作LOD，并且调整适配距离参数，成本高
* LOD切换时会有明显的跳变
* 对于一些跨度比较大的模型，前后LOD之间的差异会很大，并不能通过一个单一的LOD模型处理这种情况

Nanite通过将模型切分成很多的Cluster，然后以Cluster为单位生成LOD，然后运行时通过计算Cluster在屏幕上的显示大小来决定LOD的级别，因为LOD是以Cluster为单位的，并且始终保持三角面在屏幕上有大概一致的大小，所以不会有明显的LOD切换跳变，同时也保证了不会出现使用过于精细的LOD模型渲染的情况。

![20250526232550](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250526232550.png)

切分Cluster另一个好处就是可以用GPU Driven渲染解决DC的压力，关于GPU Driven可以看一下Ubisoft之前在大革命时候的分享
[https://advances.realtimerendering.com/s2015/aaltonenhaar_siggraph2015_combined_final_footer_220dpi.pptx]

# Nanite Overview
