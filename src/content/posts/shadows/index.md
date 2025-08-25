---
title: Shadows
published: 2025-03-22
description: ''
image: ''
tags: []
category: ''
draft: false 
lang: ''
---

# 直接光阴影（Direct Shadow）

## 虚拟阴影贴图（Virtual Shadow Map）
https://dev.epicgames.com/documentation/en-us/unreal-engine/virtual-shadow-maps-in-unreal-engine
VSM能够根据场景的实际需求动态分配和渲染阴影贴图，能够最大限度的提高ShadowMap的利用率，用于取代其他基于ShadowMap的阴影技术，比如CSM、PerObject Shadow Map等。

## 级联阴影贴图（Cascaded Shadow Maps）
VSM开启之后CSM会自动被禁用，作为一些低端机器以及移动端的替代方案。

## PerObject Shadow Map
与CSM类似，VSM开启之后PerObject Shadow Map会自动被禁用，作为一些低端机器以及移动端的替代方案。

## 距离场阴影（Distance Field Shadow）
距离场阴影在开启VSM之后依然可以使用，作为VSM在远距离非Nanite物体阴影的补充。
![img_v3_02pg_b8685e66-d051-4ac4-9aca-f130cfb7670g](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/img_v3_02pg_b8685e66-d051-4ac4-9aca-f130cfb7670g.jpg)
![img_v3_02pg_c477a994-4f32-4ab1-bc37-4c4b8192486g](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/img_v3_02pg_c477a994-4f32-4ab1-bc37-4c4b8192486g.jpg)
![img_v3_02pg_73842504-928e-44e1-99d5-0350cea5746g](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/img_v3_02pg_73842504-928e-44e1-99d5-0350cea5746g.jpg)

## 胶囊体阴影（Capsule Shadow）
官方文档中没有明确说明胶囊体阴影在VSM开启之后是否会被禁用，但从实际效果以及代码来看，Capsule Shadow在VSM开启之后没有生效。
而且Capsule Shadow主要是为了获得非常强的Soft Shadow效果，效果也依赖Physics Assets中的胶囊体的形状，如果Capsule摆放不合理，阴影效果并不是很好。
![20250825180900](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250825180900.png)
目前先不启用，看后续的效果上的需求。

## 接触阴影（Contact Shadow）
接触阴影在VSM开启之后依然可以使用，可以作为VSM的补充，增加Pixel级别的阴影细节。
![20250825183524](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250825183524.png)
![20250825183647](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250825183647.png)

## 光追阴影（Ray Traced Shadow）
光追阴影应该是精度最高的一档，但性能开销也比较大，而且由于性能问题，可能会剔除调一些比较小的物件，导致一些细节上的缺失，而且如果是使用的Nanite，在面数非常高的情况下，如果直接Tracing高模，性能开销会非常大，但如果是Tracing低模，可能会导致不精确的问题，所以目前暂定作为某些特殊情况下的补充。
![img_v3_02pg_954d6c21-5599-46a8-b4e5-7945392e283g](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/img_v3_02pg_954d6c21-5599-46a8-b4e5-7945392e283g.jpg)
![img_v3_02pg_da330656-7914-4f76-bb36-a19059a1482g](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/img_v3_02pg_da330656-7914-4f76-bb36-a19059a1482g.jpg)

# 间接光阴影（Indirect Shadow）
## 环境光遮蔽（Ambient Occlusion）

## 屏幕空间环境光遮蔽（Screen Space Ambient Occlusion）
