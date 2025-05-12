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

## 物理量（Quantities）
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

常见光照强度：
![20250512000306](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512000306.png)
![20250509115059](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509115059.png)

### 色度学（Colorimetry）
前面我们提到光谱分布（Spectral Distribution），但是光谱分布是一个连续的函数，在实际的运算中比较难处理，而色度学（Colorimetry）是将光谱分布离散化为一个有限的颜色空间（Color Space），通过颜色空间中的坐标数值来表示颜色。

XYZ颜色空间是CIE（国际照明委员会）定义的一个颜色空间用来表示人眼所能感知的颜色
![20250511224906](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250511224906.png)

对于颜色，通常把亮度（Luminance）和色度（Chromaticity）分开来表示，所以CIE定义了XYZ中的一个平面```X + Y + Z = 1```表示色度（Chromaticity）
![20250509111724](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509111724.png)

RGB空间是电视和显示器使用的颜色空间，用来控制三种颜色的强度来表示颜色，常见的RGB空间有sRGB（Linear）/Rec.709、DCI-P3、ACEScg等，RGB空间与XYZ空间之间可以通过线性变换互相转换，由于显示器的色域（Gamut）有限，所以一般为这些显示设备设计的RGB空间都只能表示XYZ空间中的一部分颜色。
![20250509111756](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509111756.png)

### 色温（Color Temperature）
色温是用来描述光源颜色的一种方式，色温是指一个黑体辐射体在一定温度下发出的光的颜色，通常用开尔文（Kelvin）来表示。色温越高，光的颜色越偏蓝；色温越低，光的颜色越偏红。
![20250512223717](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512223717.png)

现实中很多光源的颜色都可以用色温来表示，引入色温作为光源的颜色可以方便对齐现实中的光源：
![20250512224118](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512224118.png)
![20250512151719](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512151719.png)

## 光源类型（Light Source Types）
前面我们基本上解决了关于光的度量问题，也就是说我们可以回答什么是光的强度跟颜色具体是什么含义了。关于光源的最后一个问题就是这些光是从哪里发出来的？如果从非常物理的角度来看，这个问题其实非常复杂，因为光产生的形式很多，比如燃烧的化学能、电子的跃迁、黑体辐射等，这些都可以产生光，但对于渲染来说，我们其实并不关心光源产生的形式，而是更多关心光是如何从光源传播的。所以我们需要对光源按照传播方式进行分类：

* Punctual Light

    Punctual Light是用来模拟光从一个点发散出来的光源，通常是一个点光源（Point Light）或者聚光灯（Spot Light），这种光源的特点是光线是从一个点向外发散的，光线的强度随着距离的增加而衰减。最常见的两种Punctual Light是点光源（Point Light）和聚光灯（Spot Light），它们的区别在于点光源是从一个点向外发散的，而聚光灯的区别就是加上了方向的参数。
    ![20250512185217](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512185217.png)

* Photometric Light

	Photometric Light是用Photometric profile文件记录各个方向的光强度分布
    
    * IES Light
    ![20250512231951](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512231951.png)

* Sun/Directional Light

    方向光主要用来模拟太阳光或者其他远处的光源，由于太阳离地球非常远，所以我们可以认为太阳光是平行的，方向光的特点是光线是平行的，光线的强度不随距离的增加而衰减。

* Area Light

    面光源是用来模拟一个面发出的光源，现实世界中的光源通常都是一个面，比如说灯泡、荧光灯等，相比于点光源，面光源能更好地模拟现实中的光源。但是由于面光源需要考虑面上所有点对于光照结果的贡献，所以一般计算比较复杂，大部分情况下都是用规则的几何体来近似面光源，并通过解析解法来计算光照结果，或者通过一些提前预计算的方式，比如LTC（Light Transport Coefficient）来近似计算光照结果。

    * Rect Light
    * Disk Light
    * Sphere Light
    * LTC Light
    ![20250512233345](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512233345.png)

## UE5
* TODO

# 材质（Material）
当光传播到物体表面时，这个时候物体表面的材质就会影响光的传播，这会决定光是被反射、折射还是吸收，进而决定相机或者人眼看到该物体时候的表现，比如说一个物体如果吸收了所有的光，那么我们看到的就是黑色的物体；如果一个物体反射了所有的光，那么我们看到的就是白色的物体；如果物体只吸收部分频率的光，那么这个物体看起来就会有颜色，所以材质是PBR中非常重要的一个部分。

## BxDF
BxDF是一系列函数的统称，用来描述光在物体表面传播的方式

### BRDF
BRDF（Bidirectional Reflectance Distribution Function）是双向反射分布函数，用来描述光在物体表面反射的方式。BRDF主要考虑光在很小区域内的反射情况，这其实也是现实中绝大多数材质的表现方式，所以BRDF是PBR中最常用的一个函数。
![20250512162541](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512162541.png)
![20250512162605](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512162605.png)
![20250512162658](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512162658.png)
![20250512162727](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512162727.png)
![20250512162752](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512162752.png)

#### Diffuse
![20250512163247](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512163247.png)

##### Lambertian
![20250512163306](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512163306.png)
![20250512163510](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512163510.png)

#### Specular

##### Microfacet
![slide-33](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/slide-33.jpg)

#### Hair
![An-illustration-of-the-geometry-of-our-hair-scattering-model-The](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/An-illustration-of-the-geometry-of-our-hair-scattering-model-The.png)

#### Cloth/Fabric

### Subsurface Scattering
![20250512182217](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512182217.png)

## UE5
* TODO

# 参与介质（Participating Media）
![20250513001147](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513001147.png)
![20250513001442](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513001442.png)
![20250513001502](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513001502.png)
![20250513001544](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513001544.png)
## 物理（Physics）
![20250513001757](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513001757.png)
## UE5
* TODO

# 遮挡（Occlusion）
## Direct Lighting
### Shadow
## Indirect Lighting
### Ambient Occlusion
## UE5

# 全局光照（Global Illumination）
## Diffuse
## Specular

# 相机（Camera）
![20250509115726](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509115726.png)
![20250509143833](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509143833.png)
## Exposure
## Tone Mapping
![20250509143735](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509143735.png)
## Color Grading
## Post Process