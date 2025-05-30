---
title: UE5/Light Source
published: 2025-05-16
description: ''
image: ''
tags: [rendering, graphics, ue5, light]
category: 'UE5'
draft: true 
lang: 'zh'
---

# Introduction

# Quantities
光的物理度量方式，一般常见的有两种，辐射度量学跟光度学，辐射度量学一般是从电磁波的角度对光进行度量，主要关注的是能量，而光度学加入了人眼的亮度感知曲线，更能反应人眼所感知的光的亮度信息，所以一般渲染领域都是用光度学的度量方式。
![20250509111423](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509111423.png)

## Intensity
对于渲染来说，一般都是用光度学的度量方式，因为相比
![20250509111447](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509111447.png)

## Color
UE5中可以设定Woking Color Space，默认是sRGB/Rec.709，在做写实渲染的项目中，要注意颜色空间的设置，另外就是需要跟其他DCC软件保持一致。
![20250516182003](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250516182003.png)

光源上的颜色提供RGB、HSV、Temperature的设置方式，在基于PBR的写实项目中，推荐把色度跟亮度分离开，所以推荐用HSV跟Temperature的设置方式，RGB的话因为色度信息跟亮度是混在一起的，所以不是很推荐

# Directional Light
Directional Light是一个平行光源，主要用于模拟太阳光等远处的光源。
![20250516184420](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250516184420.png)

## Intensity
UE5中Directional Light的强度单位是Lux，度量是illuminance，也就是单位面积上接收到的光通量。
![20250516182437](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250516182437.png)

## Color
RGB/HSV/Temperature

# Point Light
光源近似为一个点光源
![20250516184446](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250516184446.png)

## Intensity
支持四种单位
* Unitless：无单位的
* Candelas：luminous intensity，单位立体角上的光通量
* Lumens：luminous flux，光源总的光通量
* EV：用曝光值来表示光源的强度
在写实项目中，推荐用Candelas或者Lumens来表示光源的强度，Unitless的话没有单位，对于PBR渲染来说，没有具体的单位很难于现实世界进行对应，所以不推荐使用，EV的话算是一个滥用的单位，EV是相机的曝光值，跟光源的强度没有直接的关系，所以也不推荐使用。

## Color
RGB/HSV/Temperature

# Spot Light
跟Point Light基本上一样，除了多了Spot相关的方向跟角度的设置
![20250516184515](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250516184515.png)

# Rect Light
UE的矩形面光源只有一个Rect Light类型
![20250516184639](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250516184639.png)

## Intensity
支持的单位跟Point Light一样