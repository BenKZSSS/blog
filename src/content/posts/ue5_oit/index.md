---
title: UE5/Ordered Independent Transparency
published: 2025-05-16
description: ''
image: ''
tags: [rendering, graphics, ue5, oit]
category: 'UE5'
draft: false 
lang: 'zh'
---

# Introduction
透明物体的渲染顺序是一个比较典型的问题，这种可以分成两种大的情况讨论：
* 不同物体之间的渲染顺序问题
  
  不同物体之间如果没有穿插，可以简单根据距离从远到近的顺序进行渲染，但如果有穿插的情况，就没办法通过物体之间的距离来进行排序了

  ![20250523185909](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250523185909.png)

* 同一个物体不同面之间的渲染顺序问题
  
  如果是同一个物体的不同面之间的渲染顺序问题，这个是由提交时候的三角面的顺序决定的

  ![20250523185534](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250523185534.png)

# Ordered Independent Transparency
Ordered Independent Transparency（OIT）就是一类解决透明物体渲染顺序问题的算法，常见的OIT算法有：
* Stochastic OIT / Dither
  * 第一种方案是用Opaque的方式进行渲染，但根据透明度进行Dither，并且每帧进行抖动，然后利用TAA累计多帧的结果，这样的话利用深度进行排序，然后多帧的结果累计达到半透的效果

* Depth Peeling
  * 第二种方案是Depth Peeling，这种方案是通过多次渲染来进行排序，第一遍渲染最远的物体，然后第二遍渲染第二远的物体，依次类推，直到渲染到最近的物体为止，这种方案的缺点是需要多次渲染，所以性能上会比较差

* Weighted Blended OIT
  * Weighted Blended OIT是通过简化跟估算原本的混合计算方程，将原本的计算转化成commutative的计算，这样就可以通过任意顺序进行渲染了，不过结果会有一些偏差
  [https://jcgt.org/published/0002/02/09/paper.pdf]

* A-Buffer / Per-Pixel List
  * Per-Pixel List是通过每个像素维护一个透明物体的列表，然后在渲染的时候将透明物体添加到列表中，最后再进行排序
  [https://www.slideshare.net/slideshow/oit-and-indirect-illumination-using-dx11-linked-lists/3443500]

* Multi-Layer Alpha Blending
  * Multi-Layer Alpha Blending跟A-Buffer/Per-Pixel List方案有点类似，但不同的是MLAB给每个Pixel的List固定了最大上限，然后通过Merge合并一些结果，相比A-Buffer/Per-Pixel List方案，MLAB的内存占用会更加可控
  [https://moscow.sci-hub.se/3056/14625f9c551425dcd8c8e63aac9074f6/salvi2014.pdf?download=true]


# UE5 OIT
UE5目前实现了两种OIT方案，一种是不太常见的直接对Index Buffer进行排序，然后利用三角面提交保序的机制达到单个Mesh的OIT，另一种是MLAB OIT

## Index Buffer OIT
对于Index Buffer的OIT方案，会在RenderTranslucency前先调用OIT::AddSortTrianglesPass对Index Buffer进行排序
![20250524010730](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250524010730.png)

![20250523141120](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250523141120.png)

排序分为三个Pass
* OIT::SortTriangleIndices(Scan)
  * Scan Pass先根据View空间的深度计算每个Triangle所在的Slice，统计每个Slice总的Triangle的数量

* OIT::SortTriangleIndices(PrefixedSum)
  * 计算Slice的PrefixSum

* OIT::SortTriangleIndices(Write)
  * Write Pass会根据PrefixSum的结果，计算每个Triangle最终的排序过后的Index，然后写入到新的Index Buffer中

## MLAB OIT
MLAB OIT的实现中，会预先分配配个像素固定上限大小的Buffer（r.OIT.SortedPixels.MaxSampleCount），默认是4，也就是每个像素最多有4个存储的Sample，超出的会Merge成一个Sample

在Translucency Pass中，半透渲染的结果不会直接写入Scene Color，而是写入OIT的Buffer中，然后根据深度排序，超出MaxSampleCount的话会进行Merge，最终通过一个Combine Pass将OIT的Buffer中的结果合并到Scene Color中

![20250524015024](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250524015024.png)
![20250523150835](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250523150835.png)