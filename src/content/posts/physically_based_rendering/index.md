---
title: Physically Based Rendering
published: 2025-05-13
description: ''
image: ''
tags: [rendering, graphics, realistic, physically based rendering, pbr]
category: 'Computer Graphics'
draft: true 
lang: 'zh'
---

# 背景（Background）
* 目标：照片级真实感渲染（Photorealistic Rendering）

PBR广泛应用之前的渲染方法主要是基于经验的（Empirical）和基于艺术家的（Artist-driven），这种方式类似于古法炼金，依赖于艺术家的经验和直觉，相当于通过大量的实例拟合出一个经验模型。
![20250511175548](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250511175548.png)

而PBR的目标是通过模拟真实世界中光的传播过程来实现真实感渲染。
![20250513104519](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513104519.png)
![20250511182147](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250511182147.png)

* 参数正确（光源、材质...） + 过程正确（传播...） = 结果正确（真实感结果...）
![20250513174333](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513174333.png)
![20250513174427](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513174427.png)

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

RGB空间是大部分显示设备用到的空间，用来控制三种颜色的强度来表示颜色，常见的RGB空间有sRGB（Linear）/Rec.709、DCI-P3、ACEScg等，RGB空间与XYZ空间之间可以通过线性变换互相转换，由于显示器的色域（Gamut）有限，所以一般为这些显示设备设计的RGB空间都只能表示XYZ空间中的一部分颜色。
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
### Directional Light
* Intensity：
    ![20250513145657](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513145657.png)
* Color：
  * RGB
  * HSV
  * Temperature
    ![20250513145842](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513145842.png)
* IES Profile：
  ![20250513150440](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513150440.png)

### Point Light
* Intensity：
  * Unitless，不推荐
  * Candela
  * Lumens
  * EV，不推荐
    ![20250513150017](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513150017.png)
* Color：同上

### Spot Light
跟Point Light类似，多了角度的设定

### Rect Light
* Intensity：同Point Light
* Color：同Point Light
* Source Texture：
![20250513154635](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513154635.png)

# 材质（Material）
当光传播到物体表面时，这个时候物体表面的材质就会影响光的传播，这会决定光是被反射、折射还是吸收，进而决定相机或者人眼看到该物体时候的表现，比如说一个物体如果吸收了所有的光，那么我们看到的就是黑色的物体；如果一个物体反射了所有的光，那么我们看到的就是白色的物体；如果物体只吸收部分频率的光，那么这个物体看起来就会有颜色，所以材质是PBR中非常重要的一个部分。

## BxDF
BxDF是一系列函数的统称，用来描述光在物体表面传播的方式：
![20250513100438](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513100438.png)

### BRDF
BRDF（Bidirectional Reflectance Distribution Function）是双向反射分布函数，用来描述光在物体表面反射的方式。BRDF主要考虑光在很小区域内的反射情况，这其实也是现实中绝大多数材质的表现方式，所以BRDF是PBR中最常用的一个函数。

光接触到物体表面，一部分光会经由外表面反射出去，另一部分光会进入物体内部进行散射，最后再从表面反射出去，BRDF所描述的材质，散射的光是在一个非常小的区域内进行的，小到我们可以认为整个过程是在一个点上进行的
![20250512162658](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512162658.png)

金属材质，进入物体内部的光会被吸收
![20250512162541](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512162541.png)

非金属材质，进入物体内部的光会被散射再出来
![20250512162605](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512162605.png)

所以BRDF一般分为Diffuse和Specular两种类型，Specular用来描述表面反射的光，Diffuse用来描述进入物体内部之后的散射出去的光。
![20250512162752](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512162752.png)

#### Diffuse
Diffuse因为是进入物体内部散射之后的光，这个散射的过程是非常复杂的，有很强的随机性，所以一般情况下从统计学的角度来看，最终散射出来的光是均匀分布的，比较常用的模型是Lambertian模型
##### Lambertian
![20250512163306](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512163306.png)

#### Specular
Specular是表面反射的光，表面反射的光是比较有方向性的，一般需要一些特定的数学模型来描述

##### Microfacet
微表面是假定物体表面是由很多微小的平面组成的，这些微小的平面是随机分布的，通过数学模型来描述这些微小的平面对于光的反射情况
![slide-33](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/slide-33.jpg)

#### Hair
毛发材质由于其微观结构的特殊性，微表面模型并不能很好地描述毛发的反射情况，所以需要一些特定的模型来描述毛发的反射情况
![An-illustration-of-the-geometry-of-our-hair-scattering-model-The](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/An-illustration-of-the-geometry-of-our-hair-scattering-model-The.png)

#### Cloth/Fabric
布料材质也是类似的情况，其微观结构也是由很多有规律的编织物组成的，所以需要一些特定的模型来描述布料的反射情况
![20250513112450](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513112450.png)

### Subsurface Scattering
次表面散射是指光在物体表面进入物体内部之后的散射，之所以区别于BRDF，是因为光在这类材质中散射传播的路径比较长，大于了我们的渲染的尺度，也就是说我们不能假定这个过程是在一个点上进行的，而是需要考虑光在物体内部的传播路径
![20250512182217](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250512182217.png)

## UE5
### Shading Model
* Default Lit：Microfacet GGX Specular + Lambertian Diffuse
  * Base Color：Reflectance，反射率，表示物体表面反射的光的强度
  * Metallic：金属度，会影响Specular跟Diffuse的分布
  * Roughness：粗糙度影响微表面法线的分布
  * Specular：基本上默认值是符合物理的，但提供更多的自由度
    ![20250513160901](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513160901.png)
    ![20250513161324](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513161324.png)
* Subsurface/Preintegrated Skin/Subsurface Profile：Subsurface Scattering
* Hair
* Cloth
* Eye
* Clear Coat
* Foliage
* Translucent
### Substrate
![20250513155217](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513155217.png)

# 几何（Geometry）
## UE5
* Nanite

# 直接/局部光照（Direct/Local Illumination）
局部光照是指光源发出的光经过物体表面反射后，直接到达相机的光照结果
![20250513104048](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513104048.png)
## UE5
* Mega Lights
![20250513161657](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513161657.png)

# 全局光照（Global Illumination）
全局光照是指光源发出的光经过物体表面反射后，经过其他物体表面反射后，最终到达相机的光照结果
![20250513104048](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513104048.png)

## Diffuse
Diffuse GI就是讨论Diffuse反射的光照结果
![20250513104352](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513104352.png)
![20250513104406](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513104406.png)

## Specular
Specular GI就是讨论Specular反射的光照结果，一般表现为高光反射
![20250513105018](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513105018.png)

## UE5
* Diffuse GI
  * Lightmap
  * Irradiance Volume
  * Screen Space GI
  * Lumen
* Specular GI
  * IBL/Reflection Capture
  * Planar Reflection
  * Screen Space Reflection
  * Lumen
* Sky Light

# 参与介质（Participating Media）
跟前面的次表面散射类似，参与介质所描述的材质中，光的传播路径会更长，虽然严格意义上来说所有的材质都可以看作是参与介质，只是传播路径的长短不同而已，但由于不同尺度上渲染的方式以及涉及到的技术不同，需要分开来讨论
![20250513001147](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513001147.png)

常见的参与介质，雾
![20250513001442](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513001442.png)

云
![20250513001502](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513001502.png)

甚至是大气，虽然一般尺度上空气对光的影响是非常小的，但当尺度足够大时，这个影响就会变得明显
![20250513001544](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513001544.png)

## 物理（Physics）
参与介质的物理上的传播原理就不过多赘述，最主要的就是参与介质中的粒子对光传播的四种影响
![20250513001757](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513001757.png)

## UE5
* Fog
  * Exponential Height Fog
  * Local Fog
  * Volume Material
* Atmosphere
  * Sky Atmosphere
* Cloud
  * Volumetric Cloud

# 遮挡（Occlusion）
光在传播中可能会被其他物体遮挡

## Direct Lighting
直接光的遮挡，就是指从光源直接到达物体表面的光线被其他物体遮挡了

### Shadow
对于很多不透明的实体，直接光的遮挡就是阴影（Shadow），阴影是指光线被其他物体遮挡后，无法到达物体表面的部分
![20250513102840](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513102840.png)

### Volumetric Shadow
对于一些不是完全不透明的物体，比如说烟雾、云等，直接光的遮挡就是体积阴影（Volumetric Shadow），光经过这些材质，没有被完全吸收掉
![20250513103030](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513103030.png)

## Indirect Lighting
间接光的遮挡，就是指光源的光经过其他物体表面反射后，

### Ambient Occlusion
![20250513103928](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513103928.png)

## UE5
* Shadow
  * Shadow Map
    * Cascade Shadow Map
    * Per Object Shadow Map
    * Virtual Shadow Map
  * Contact Shadow
  * Capsule Shadow
  * Ray Traced Shadow
* Volumetric Shadow
* Indirect Shadow
  * Screen Space Ambient Occlusion
  * Lumen

# 相机（Camera）
最后一个阶段就是相机的成像过程，光经过物体表面反射后，最终到达相机的传感器，传感器将光转换为电信号，最后通过图像处理算法生成最终的图像，这是一个Scene To Screen的过程
![20250509115726](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509115726.png)
![20250509143833](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509143833.png)

## Exposure
曝光是指相机传感器接收到光的强度，由三个参数决定
* 光圈（Aperture）
* 快门速度（Shutter Speed）
* ISO感光度（ISO Sensitivity）
![20250513105301](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513105301.png)
![20250513105418](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513105418.png)

## Tone Mapping
色调映射是指将Scene-Referred转换到Display-Referred的过程
![20250509143735](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509143735.png)

## Color Grading
最后的色调映射，类似于影像后期处理，主要是对图像进行一些色彩上的调整，比如说饱和度、对比度、色温等
![20250513105908](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513105908.png)

## UE5
* Cine Camera
    ![20250513163535](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513163535.png)
* Auto Exposure
* White Balance
* Tone Mapping
  * Local Tone Mapping
* Color Grading
* ACES Workflow
  * RRT
  * ODT