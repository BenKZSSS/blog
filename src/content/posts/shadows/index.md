---
title: Shadows
published: 2025-03-22
description: ''
image: ''
tags: []
category: ''
draft: true 
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

## 总结
* VSM作为基础
* Contact Shadow补充Pixel级别的阴影细节
* Distance Field Shadow作为远距离非Nanite物体阴影的补充
* Ray Traced Shadow作为特殊情况下的补充

# 间接光阴影（Indirect Shadow）
## 环境光遮蔽（Ambient Occlusion）

### 屏幕空间环境光遮蔽（Screen Space Ambient Occlusion）
通过屏幕深度信息计算AO
![img_v3_02pg_dd6f1ac5-7c1b-48d7-b880-763c3677bccg](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/img_v3_02pg_dd6f1ac5-7c1b-48d7-b880-763c3677bccg.jpg)
![img_v3_02pg_e05a8462-dddd-498f-ba7e-cbaa1eaabfbg](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/img_v3_02pg_e05a8462-dddd-498f-ba7e-cbaa1eaabfbg.jpg)

### Lumen Short Range AO
Lumen中的短距离AO，对比SSAO效果不是很明显
![img_v3_02pg_18995910-c34f-4526-9b7e-53519c6c41fg](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/img_v3_02pg_18995910-c34f-4526-9b7e-53519c6c41fg.jpg)

### 距离场AO（Distance Field Ambient Occlusion）
与Lumen冲突，在开启Lumen的情况下无法使用
![20250825214543](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250825214543.png)

## 方向性的环境光遮蔽（Directional Ambient Occlusion）

### Lumen
Indirect Shadow本质上是全局光照的一部分，Lumen一定程度上已经解决了Indirect Shadow的问题
![img_v3_02pg_46b8996c-a702-424c-b082-ddaf4175a49g](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/img_v3_02pg_46b8996c-a702-424c-b082-ddaf4175a49g.jpg)

### 距离场Indirect Shadow（Distance Field Indirect Shadow）
Distance Field Indirect Shadow是通过距离场的Tracing计算的间接光阴影，可以跟Lumen结合使用，不过在没有VLM的情况下，Tracing的方向是通过SkyLight的方向来计算的，效果跟GroundTruth还是有一定差距，暂时作为一种补充手段。
![img_v3_02pg_efbe5b74-baa8-4316-83fd-a04c9e9b741g](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/img_v3_02pg_efbe5b74-baa8-4316-83fd-a04c9e9b741g.jpg)

## 总结
* AO
    * SSAO作为基础AO
* Directional AO
    * Lumen作为基础
    * Distance Field Indirect Shadow作为补充