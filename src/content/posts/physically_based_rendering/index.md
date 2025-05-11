---
title: Physically Based Rendering
published: 2025-04-13
description: ''
image: ''
tags: [rendering, graphics, realistic, physically_based_rendering, pbr]
category: 'Computer Graphics'
draft: true 
lang: 'zh'
---

# 背景（Background）
* 目标：照片级真实感渲染（Photorealistic Rendering）

PBR广泛应用之前的渲染方法主要是基于经验的（Empirical）和基于艺术家的（Artist-driven），这种方式类似于古法炼金，依赖于艺术家的经验和直觉，相当于通过大量的实例拟合出一个经验模型。
![20250511175548](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250511175548.png)

而PBR的目标是通过模拟真实世界中光的传播过程来实现真实感渲染。
![20250511182147](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250511182147.png)
* 参数正确（光源、材质...） + 过程正确（传播...） = 结果正确（真实感结果...）

# 光源（Light Source）
光源是渲染的起点，在基于物理的渲染中，我们就需要从物理学的角度来定义光源。
## Quantities
首先我们需要关注光源的物理量（Quantities），物理量的准确性是PBR框架中非常基础也非常重要的部分。换句话说，我们在引擎中设置光源的强度（Intensity）和颜色（Color）时，到底在物理上是代表什么含义，这也直接影响了我们参数设置的正确性。
### 辐射度量学（Radiometry）
首先我们先从物理学的视角来看下光源的辐射度量学（Radiometry），辐射度量学是从电磁辐射的角度来研究光的传播过程

光的本质是电磁波，可见光是电磁波谱中波长在380nm到780nm之间的部分
![20250509104812](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509104812.png)

而一个光源发出的光可能是多种波长的光的组合，这种组合可以用光谱分布（Spectral Distribution）来表示，也就是每种波长的光的能量分布。
![20250511222954](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250511222954.png)

### 光度学（Photometry）
光度学是从人眼的角度来研究光的，因为人眼对不同波长的光的敏感度是不同的，所以光度学中的数值更加符合人眼的感知。

人眼对于不同波长的光的敏感度可以用光谱亮度函数（Spectral Luminous Efficiency Function）来表示，其峰值在555nm处，表示人眼对绿色光的敏感度最高。
![20250509111423](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509111423.png)

下面是辐射度量学和光度学的常用单位的对比，总结来说，辐射度量学关注的是能量，而光度学关注的是亮度：
![20250509111447](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509111447.png)


### 色度学（Colorimetry）
前面我们提到光谱分布（Spectral Distribution），但是光谱分布是一个连续的函数，在实际的运算中比较难处理，而色度学（Colorimetry）是将光谱分布离散化为一个有限的颜色空间（Color Space），通过颜色空间中的坐标数值来表示颜色。

XYZ颜色空间是CIE（国际照明委员会）定义的一个颜色空间用来表示人眼所能感知的颜色
![20250511224906](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250511224906.png)

对于颜色，通常把亮度（Luminance）和色度（Chromaticity）分开来表示，所以CIE定义了XYZ中的一个平面```X + Y + Z = 1```表示色度（Chromaticity）
![20250509111724](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509111724.png)

RGB空间是电视和显示器使用的颜色空间，用来控制三种颜色的强度来表示颜色，常见的RGB空间有sRGB（Linear）/Rec.709、DCI-P3、ACEScg等，RGB空间与XYZ空间之间可以通过线性变换互相转换，由于显示器的色域（Gamut）有限，所以一般为这些显示设备设计的RGB空间都只能表示XYZ空间中的一部分颜色。
![20250509111756](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509111756.png)

### 光源类型（Light Source Types）
* Punctual Light
    * Point Light
    * Spot Light
* Photometric Light
    * IES Light
* Sun/Directional Light
* Area Light
    * Rect Light
    * Disk Light
    * Sphere Light

## 光源参数
* 光源类型
* 强度（Intensity）/亮度（Luminance）
* 颜色（Color）
![20250512000306](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512000306.png)
![20250509115059](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509115059.png)

# Material
## BxDF
### BRDF
### BSDF
## Texture

# Camera
![20250509115726](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509115726.png)
![20250509143833](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509143833.png)
## Exposure
## Tone Mapping
![20250509143735](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509143735.png)
## Color Grading
## Post Process