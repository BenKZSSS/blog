---
title: UE5/Rendering Pipeline
published: 2025-05-16
description: ''
image: ''
tags: [rendering, graphics, ue5, pipeline]
category: 'UE5'
draft: true 
lang: 'zh'
---

# Introduction
UE的渲染管线是一个非常复杂的系统，包含了很多特性，很难在一个文档中深入到每个细节，所以这里最主要还是关注一些最核心的方案跟技术，主要分成两个大的部分来展开：
* PBR Pipeline：PBR是写实渲染的一个重要的基础，基本目前的很多新技术也都是在这个框架下，进一步改进各种细节跟实现，使得结果能够更加接近Ground Truth，所以这个部分主要是罗列UE5中PBR下的各种技术方案跟特性
* PC/Console Rendering Pipeline：这个部分主要从具体的代码实现上来分析UE5的渲染管线，主要关注各种特性是如何在引擎里面实现的，包含了哪些主要的阶段

# PBR Pipeline
对于写实渲染，PBR（Physically Based Rendering）是一个非常重要的基础，相比于传统的渲染管线，PBR从物理的角度出发，使用物理上更合理的模型来描述光的传播、材质的反射等，使得渲染结果更加接近真实世界的光照效果。

## Light Source
* 亮度：采用光度学的度量单位
![20250521101747](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250521101747.png)
* 颜色：采用默认的工作空间，sRGB/Rec.709，光源颜色使用HSV/Temperature来表示，将亮度跟色度信息分离
* 光源类型
  * Point Light
  * Spot Light
  * Directional Light
  * Rect Light

## Material
* Shading Model：默认的Shading Model可以覆盖大多数常见材质的BxDF，后续有特殊材质再进行扩展
  * Default Lit
  * Unlit
  * Subsurface
  * Hair
  * Cloth
  * Eye
  * Foliage
  * Clear Coat
  * Thin Translucent
* Substrate：UE5的一套新的材质系统，能够组合多个BxDF做出更加复杂的材质，是目前还处于实验性质的特性，而且从目前的实现方案来看，消耗会比较高，暂时没有这种材质上的需求，所以暂时先不考虑。
![20250513155217](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513155217.png)

## Geometry
* Nanite：UE5的新的几何系统，能够支持非常高的多边形数量的模型，Nanite会将模型的多边形数量压缩到一个合理的范围内，并且在渲染的时候动态地加载需要的细节，避免了传统的LOD系统带来的麻烦。
![20250521112017](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250521112017.png)

## Direct/Local Lighting
* Mega Lights：UE5的一个新的直接光照系统，能够支持大量的光源的直接光照
![20250513161657](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250513161657.png)

## Indirect/Global Illumination
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

## Participating Media
* Fog
  * Exponential Height Fog
  * Local Fog
  * Volume Material
* Atmosphere
  * Sky Atmosphere
* Cloud
  * Volumetric Cloud

## Occlusion
* Shadow Map
  * Point/Spot Shadow Map
  * Cascade Shadow Map
  * Virtual Shadow Map
  * Per Object Shadow Map
* Contact Shadow
* Ray Tracing Shadow
* Ambient Occlusion
  * Screen Space Ambient Occlusion
  * Ray Tracing Ambient Occlusion

## Camera
* Exposure
  * Auto Exposure
  * Local Exposure
* White Balance
* Tone Mapping
* Color Grading
* ACES Workflow

# PC/Console Rendering Pipeline
完整的PC/Console的渲染管线非常复杂，包含了很多特性跟阶段，这里先从最基础的Deferred Shading开始，屏蔽掉一些特性，逐步深入到更复杂的管线。

## Basic Deferred Shading
最基础的Deferred Shading管线
* OnRenderBegin
* BeginInitViews
* EndInitViews
* RenderPrePass
* RenderVelocities（r.VelocityOutputPass=0）
* RenderBasePass
* RenderVelocities（r.VelocityOutputPass=2）
* RenderLights
* RenderTranslucency
* AddPostProcessingPasses

### OnRenderBegin
在Render非常早的阶段，OnRenderBegin会被调用，这里会做很多初始化工作，对于最基础的管线来说，最主要的工作就是LaunchVisibilityTasks。

![20250522144458](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522144458.png)

LaunchVisibilityTasks会创建一系列的裁剪任务：
* LightVisibility
* FrustumCull
* OcclusionCull
* ComputeRelevance
这个阶段会创建这些任务，并且设置好所有任务的依赖关系，然后Trigger，这样可以尽可能跟后面阶段的一些逻辑并行执行，提高性能。

### BeginInitViews
为了进一步提升并行度，InitViews也被拆分成了BeginInitViews和EndInitViews两个阶段。Begin阶段会Trigger GatherDynamicMeshElements任务。

### EndInitViews
End阶段会等待前面各种裁剪以及DynamicMeshElements的任务完成，以及其他一些Feature的任务。

### RenderPrePass
主要是做一些提前的Pass，最主要的是DepthPass，用于后续AO等效果，以及防止Overdraw

![20250526110115](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250526110115.png)

### RenderVelocities（r.VelocityOutputPass=0）
如果开启了r.VelocityOutputPass=0，在PrePass阶段会渲染Velocity Pass，这个Pass会输出每个像素的速度信息，后续可以用于Motion Blur、TAA等效果

### RenderBasePass
Base Pass阶段，主要任务就是提交Mesh Draw Commands，然后生成GBuffer
* Clear GBuffer
* Submit Mesh Draw Commands
  * 提交Mesh Draw Commands到RHI
* Anisotropic Pass
  * 提交各向异性材质的Mesh Draw Commands到RHI

![20250522144748](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522144748.png)

### RenderVelocities（r.VelocityOutputPass=2）
如果开启了r.VelocityOutputPass=2，在Base Pass之后会渲染Velocity Pass

### RenderLights
RenderLights也就是延迟渲染中的Lighting/Shading阶段，主要的工作就是根据GBuffer已经光源信息计算光照结果，Lighting主要分两个大的部分：
* SimpleLights：
  * 处理最简化的光源，没有Shadow、LightFunction等，一般都是特效上使用的光源
* BatchedLights
  * 处理没有Shadow、LightFunction的光源，相比SimpleLights，支持的功能会多一些，比如IES Profile等
* UnbatchedLights
  * 处理最复杂的光源，包含Shadow、LightFunction等

![20250522144844](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522144844.png)

### RenderTranslucency
渲染透明物体，因为延迟渲染的GBuffer只能存储一层，也就是最前面的物体表面信息，透明物体因为会透射后面物体的信息，所以只能单独渲染。

而且默认情况下PC管线里面，为了支持DOF之后的透明物体，不同阶段的Translucency会被分开渲染，最终在后处理阶段再做Composite

![20250522145804](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522145804.png)
![20250522145956](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522145956.png)

### AddPostProcessingPasses
PostProcessing阶段，最主要的后处理：
* DepthOfField
  * 景深
* TAA/TSR
  * Temporal Anti-Aliasing
* MotionBlur
  * 动态模糊
* Histogram
  * 计算亮度直方图，用于自动曝光
* EyeAdaptation
  * 自动曝光
* Bloom
  * 泛光
* PostProcessMaterial
  * 自定义的各种后处理材质
* ToneMapping
  * 色调映射，从Scene-Referred到Display-Referred
* VisualizeXXX
  * 各种调试用的后处理
* PrimaryUpscale/SecondaryUpscale
  * 上采样，比如各种动态分辨率或者超分会在这个阶段进行，把低分辨率的渲染结果上采样到最终的分辨率

![20250522150029](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522150029.png)

## Cluster Deferred Lighting
Clustered Deferred Shading是将视锥切分成Grid，每个Grid计算相关的Lights，通过这种方式优化场景中有大量Local Lights的光照计算的性能
* OnRenderBegin
* BeginInitViews
* EndInitViews
* RenderPrePass
* RenderVelocities（r.VelocityOutputPass=0）
* **GatherLightsAndComputeLightGrid**
* RenderBasePass
* RenderVelocities（r.VelocityOutputPass=2）
* RenderLights
  * **AddClusteredDeferredShadingPass (r.UseClusteredDeferredShading=1)**
* RenderTranslucency
* AddPostProcessingPasses

### GatherLightsAndComputeLightGrid
将视锥切分成Grid，每个Grid计算相关的Lights，通过这种方式优化场景中有大量Local Lights的光照计算的性能，以及优化其他一些需要获取Light信息的Feature，比如Lumen、MegaLights等

![20250522150850](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522150850.png)

### AddClusteredDeferredShadingPass
如果开启了r.UseClusteredDeferredShading=1，这部分被分类到Grid中的Lights会走Clustered Deferred Shading的渲染流程，这个Pass会获取Grid中存储的相关Light数据然后计算光照结果

![20250522150944](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522150944.png)

## Shadow Map
基于Shadow Map的阴影，包含主光源的阴影（Directional Light）以及其他的局部光源的阴影（Point Light、Spot Light等）
* OnRenderBegin
* BeginInitViews
  * **BeginInitDynamicShadows**
* EndInitViews
  * **BeginShadowGatherDynamicMeshElements**
* RenderPrePass
* RenderVelocities（r.VelocityOutputPass=0）
* **FinishInitDynamicShadows**
* GatherLightsAndComputeLightGrid
* **RenderShadowDepthMaps (r.shadow.ShadowMapsRenderEarly=1 && VSM disabled)**
* RenderBasePass
* **RenderShadowDepthMaps (Not rendered early)**
* RenderVelocities（r.VelocityOutputPass=2）
* RenderLights
  * **RenderDeferredShadowProjections**
* RenderTranslucency
* AddPostProcessingPasses

### BeginInitDynamicShadows
在BeginInitViews阶段，BeginInitDynamicShadows会被调用，这里会创建Shadow相关的一些初始化任务，比如裁剪剔除、创建FProjectedShadowInfo等。与前面的视锥剔除的任务类似，这些任务默认情况下也会通过异步的方式开始执行。

### BeginShadowGatherDynamicMeshElements
生成DynamicMeshElements的Shadow的DrawCommand

### FinishInitDynamicShadows
等待前面的各种异步任务完成

### RenderShadowDepthMaps
渲染Shadow Depth Maps，有两种渲染的时序，一个是在BasePass之前渲染，需要开启r.Shadow.ShadowMapsRenderEarly=1，并且没有开启VSM的情况，否则会延迟到BasePass之后渲染。

* RenderShadowDepthMapAtlases
  * 渲染非VSM的Shadow Depth Maps
* ShadowMapCubemaps
  * 渲染Point Light的Shadow Depth Maps
* PreshadowCache
* TransculencyShadowMapAtlases
  * 渲染半透的Shadow Maps，这里用的技术方案是Fourier Opacity Map[https://volumetricshadows.wordpress.com/wp-content/uploads/2011/06/fourier-opacity-mapping.pdf]

![20250522151359](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522151359.png)

### RenderDeferredShadowProjections
在Lighting阶段，生成屏幕空间的Shadow Mask，通过屏幕空间深度还原世界坐标，然后采样Shadow Map计算遮挡关系，Shadow Mask最终会在Lighting阶段采样并参与光照计算

![20250522151451](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522151451.png)

## SkyLight
SkyLight是一个特殊的光源，它主要用于模拟天空大气的光照效果
* OnRenderBegin
* BeginInitViews
  * BeginInitDynamicShadows
  * **UpdateSkyIrradianceGpuBuffer**
* EndInitViews
  * BeginShadowGatherDynamicMeshElements
* RenderPrePass
* RenderVelocities（r.VelocityOutputPass=0）
* FinishInitDynamicShadows
* GatherLightsAndComputeLightGrid
* RenderShadowDepthMaps (r.shadow.ShadowMapsRenderEarly=1 && VSM disabled)
* RenderBasePass
* RenderShadowDepthMaps (Not rendered early)
* RenderVelocities（r.VelocityOutputPass=2）
* RenderLights
  * RenderDeferredShadowProjections
* **RenderDeferredReflectionsAndSkyLighting**
* RenderTranslucency
* AddPostProcessingPasses

### UpdateSkyIrradianceGpuBuffer
从SkyLight中获取SkyIrradiance的GPU Buffer，里面存储了SkyLight的SH系数，如果不是Dynamic SkyLight的话，会在BasePass阶段计算光照
![20250522144308](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522144308.png)

### RenderDeferredReflectionsAndSkyLighting
这里计算Dynamic SkyLight的光照结果
![20250522144015](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522144015.png)

## Fog
雾效渲染
* OnRenderBegin
* BeginInitViews
  * BeginInitDynamicShadows
  * UpdateSkyIrradianceGpuBuffer
* EndInitViews
  * BeginShadowGatherDynamicMeshElements
* RenderPrePass
* RenderVelocities（r.VelocityOutputPass=0）
* FinishInitDynamicShadows
* GatherLightsAndComputeLightGrid
* RenderShadowDepthMaps (r.shadow.ShadowMapsRenderEarly=1 && VSM disabled)
* RenderBasePass
* RenderShadowDepthMaps (Not rendered early)
* RenderVelocities（r.VelocityOutputPass=2）
* RenderLights
  * RenderDeferredShadowProjections
* RenderDeferredReflectionsAndSkyLighting
* **ComputeVolumetricFog**
* **RenderFog**
* RenderTranslucency
* AddPostProcessingPasses

### ComputeVolumetricFog
如果开启了体积雾，在RenderFog之前会先计算体积雾的相关信息，UE5的体积雾采用了Froxel的方案[https://advances.realtimerendering.com/s2015/Frostbite%20PB%20and%20unified%20volumetrics.pptx]

![20250523100525](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250523100525.png)

### RenderFog
在这个阶段计算Exponential Height Fog的光照结果，因为Exponential Height Fog是有解析解的，所以直接通过深度还原世界坐标计算雾效结果，如果开起了体积雾，这里也会结合体积雾的结果

![20250522164933](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250522164933.png)

## TSR
Temporal Super Resolution（TSR）是UE5中一个新的超分辨率技术
* OnRenderBegin
  * **PrepareViewStateForVisibility**
* BeginInitViews
  * BeginInitDynamicShadows
  * UpdateSkyIrradianceGpuBuffer
* EndInitViews
  * BeginShadowGatherDynamicMeshElements
* RenderPrePass
* RenderVelocities（r.VelocityOutputPass=0）
* FinishInitDynamicShadows
* GatherLightsAndComputeLightGrid
* RenderShadowDepthMaps (r.shadow.ShadowMapsRenderEarly=1 && VSM disabled)
* RenderBasePass
* RenderShadowDepthMaps (Not rendered early)
* RenderVelocities（r.VelocityOutputPass=2）
* RenderLights
  * RenderDeferredShadowProjections
* RenderDeferredReflectionsAndSkyLighting
* ComputeVolumetricFog
* RenderFog
* RenderTranslucency
* AddPostProcessingPasses
  * **AddTemporalSuperResolutionPasses**

### PrepareViewStateForVisibility
在OnRenderBegin/PrepareViewStateForVisibility阶段，这里会计算TAA/TSR的采样信息，生成JitterX/Y的值，然后重新计算ViewMatrix等

![20250523105729](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250523105729.png)

### AddTemporalSuperResolutionPasses
在后处理阶段，TSR利用当前帧的信息加上历史帧的信息来计算超分结果

![20250523114156](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250523114156.png)

## OIT（Order Independent Transparency）
Order Independent Transparency（OIT）是UE5中一个新的透明物体渲染技术，能够使得透明物体的渲染有正确的前后关系。
* OnRenderBegin
  * PrepareViewStateForVisibility
* BeginInitViews
  * BeginInitDynamicShadows
  * UpdateSkyIrradianceGpuBuffer
* EndInitViews
  * BeginShadowGatherDynamicMeshElements
* RenderPrePass
* RenderVelocities（r.VelocityOutputPass=0）
* FinishInitDynamicShadows
* GatherLightsAndComputeLightGrid
* RenderShadowDepthMaps (r.shadow.ShadowMapsRenderEarly=1 && VSM disabled)
* RenderBasePass
* RenderShadowDepthMaps (Not rendered early)
* RenderVelocities（r.VelocityOutputPass=2）
* RenderLights
  * RenderDeferredShadowProjections
* RenderDeferredReflectionsAndSkyLighting
* ComputeVolumetricFog
* RenderFog
* **OIT::AddSortTrianglesPass**
* RenderTranslucency
  * **OIT::AddOITComposePass**
* AddPostProcessingPasses
  * AddTemporalSuperResolutionPasses

### OIT::AddSortTrianglesPass
UE5的OIT其中一个方案是对三角面按照视角进行排序，然后将排序后的三角面输出到新的Index Buffer中

![20250523141120](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250523141120.png)

### RenderTranslucency
在RenderTranslucency阶段，透明物体的BasePass不会直接输出结果到OM进行Blend，而是会写入到Buffer中
![20250523143125](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250523143125.png)

### OIT::AddOITComposePass
等待所有的透明物体都渲染完成之后，OIT::AddOITComposePass会被调用，这里会将Buffer中的存储的Sample按照顺序排序然后进行Blend

![20250523150835](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250523150835.png)


# TODO
* Nanite
* Lumen
* MegaLights
* VSM
* TSR