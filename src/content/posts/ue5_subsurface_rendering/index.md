---
title: UE5/Subsurface-Rendering
published: 2025-04-13
description: ''
image: ''
tags: [ue5, subsurface, rendering, graphics]
category: 'UE5'
draft: false 
lang: 'zh'
---

# Introduction
Subsurface Scattering，也就是次表面散射，是一种常见的材质特性，比如皮肤、蜡烛、牛奶等材质都具有这种特性。次表面散射的原理是光线在物体内部发生散射，导致光线在物体内部传播一段距离后才会被反射出来。
![20250406223859](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250406223859.png)

由于光线在物体内部传播的距离比较远，一般是指远大于渲染精度，比如说超出了一个像素的范围，所以一般的BRDF模型无法很好地模拟次表面散射的效果，因此我们需要一些专门的模型来模拟次表面散射的效果。
需要模拟次表面散射的效果，最主要的是需要考虑周围的光线对物体的影响，而不是只考虑当前渲染位置的光线，目前比较常用的方案：
* ScreenSpace SSS：通过屏幕空间的方式来模拟次表面散射的效果，主要是通过对周围的像素进行采样来计算次表面散射的效果，这个也是目前3A游戏中比较常用的方案，UE5中也有内置的SSS方案。
![20250406223207](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250406223207.png)
* Preintegrated SSS：通过预计算的方式来模拟次表面散射的效果，假设表面类似于一个球形，然后将次表面散射积分的结果通过纹理的方式存储起来，然后在渲染的时候通过采样纹理的方式来计算次表面散射的效果，这个方面相对比较简单，消耗也比较小，比较适合移动端渲染，UE5中也新增了对Preintegrated SSS的支持。
![20250406223409](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250406223409.png)

# Subsurface Rendering
UE5中Subsurface的材质有两种Shading Model，分别是Subsurface和Subsurface Profile：
* Subsurface ShadingModel：一个比较简单的模型，只在Lighting中通过View/Normal/Light等信息进行计算，模拟大概的次表面散射效果，可以用在一些简单的不需要比较精确的次表面散射效果的材质上，比如牛奶、蜡烛等。
* Subsurface Profile ShadingModel：相对复杂的模型，结果也更精确，这个Model中会使用到Subsurface Profile的纹理，可以通过Subsurface Profile的散射模型参数进行更加复杂精确的次表面散射效果的计算，一般用在高品质的皮肤等材质上。

## Subsurface ShadingModel
Subsurface ShadingModel的实现主要是在BasePass阶段通过CustomData输出Subsurface Color，然后在Lighting阶段进行Subsurface ShadingModel的计算。
![20250408004351](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250408004351.png)

## Subsurface Profile ShadingModel
Subsurface Profile ShadingModel相比较于Subsurface ShadingModel是一个更加复杂但也是更加物理准确的模型，Subsurface Profile ShadingModel需要通过一个Subsurface Profile的Asset配置Subsurface的散射模型参数并且存储在Texture中，在BasePass阶段通过CustomData输出Profile的ID，然后在后续的Subsurface Pass中进行屏幕空间的Subsurface光照计算。

### Subsurface Profile
Profile参数：
* Burley Normalized相关参数: 由Burley提出的一个散射模型，相比之前的模型能更加精确地拟合Reference数据，具体参考
  ["Approximate Reflectance Profiles for Efficient Subsurface Scattering"](https://graphics.pixar.com/library/ApproxBSSRDF/paper.pdf)
* Transmission相关参数：透射相关的参数，需要LightSource上开启Transmission，可以用在一些比较薄的材质上，比如耳朵、手指等
* Dual Specular相关参数：双重高光相关的参数，主要是针对皮肤的高光进行的优化模型，Dual lobes specular能够更加贴近真实皮肤的高光效果，具体参考
  ["Next-Generation Character Rendering"](https://www.iryoku.com/downloads/Next-Generation-Character-Rendering-v6.pptx)

多个Profile的参数会存储在一张大的SSProfile Texture中，然后在后续的Subsurface Pass中通过Profile ID进行采样。
![20250408231953](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250408231953.png)

### Subsurface Pass

* SSS::InitGroupCounter
  
  清理置零TileTypeCountBuffer

* SSS::Setup
  
  初始化一些Buffer，SetupTexture、ProfileIdTexture，切分Tile计算Burley/SeparableGroupBuffer等
  ![20250413014719](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250413014719.png)

* SSS:BuildIndirectArgs

  计算Tile的IndirectArgs

* GenerateMips

  当TileCount大于一定阈值时计算SetupTexture的Mips用来进行后续的优化

* SSS:PassOne_Burley

  通过前面生成的IndirectArgs进行Tile绘制，计算Burley的散射模型，主要的计算就是在屏幕空间采样周围像素的Diffuse，然后根据Profile的参数计算对当前像素的散射影响，这里有个优化就是采样的时候会根据当前像素周围的Variance进行采样，Variance比较小的时候，会减少采样的像素数量，这个Variance信息是根据前帧的SSS Pass计算得来
  ![20250413022255](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250413022255.png)

* SSS:PassTwo_SepHon

  类似Burley的Pass，这里是针对使用Sparable Profile的Tile，Honizontal Pass 

* SSS:PassThree_SepVer

  Separable Profile的第二个Pass, Vertical Pass

* SSS:PassFour_BVar

  更新Variance Buffer，给下帧的Burley Pass用
  ![20250413032026](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250413032026.png)

* SSS:Recombine

  将SSS Lighting的结果结合GBuffer的结果进行合成
  ![20250413034346](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250413034346.png)

* SSS:CopyToSceneColor

  SSS的最终结果Copy到SceneColor中
  ![20250413034431](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250413034431.png)