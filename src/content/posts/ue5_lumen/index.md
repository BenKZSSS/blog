---
title: UE5/Lumen
published: 2025-06-13
description: ''
image: ''
tags: [rendering, graphics, ue5, lumen, global illumination]
category: 'UE5'
draft: true 
lang: 'zh'
---

# Introduction
Lumen是UE5中全新的全局光照方案，用于支持实时动态的全局光照效果。这篇将主要介绍Lumen主要的原理方案和具体的实现上的流程。

主要按照三个大的部分来说明，这也是很多全局光照方案共通的需要解决的几个问题：
* Scene Representation
  * 因为全局光照是一个比较复杂的计算过程，通常需要获取场景的各种几何跟材质信息，如果直接用原始的场景信息，数据会非常庞大跟复杂，性能会很差，而且一般全局光照对于场景的精度要求不高，所以一般都会用另外一种简化的数据结构对场景进行粗略的表达
* Lighting
  * 然后是光照计算，Indirect Lighting是通过其他物体表面Bounce得到的，所以需要先从光源计算直接光照，另外一般为了支持Multi-bounce的Indirect Lighting，还需要缓存一些Indirect Lighting的结果
* Propogation
  * 最后一个部分就是Propogation，也就是将光照信息从一个物体表面传播到另一个物体表面，这个一般也是整个全局光照中最复杂，消耗最高的部分。

Lumen整体上是一套基于Ray Tracing的方案，但因为目前硬件性能上的限制，每帧能够处理的Tracing数量是有限的，所以Lumen通过各种混合的方式来实现，大致的流程如下图所示：
![20250611164501](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250611164501.png)

# Scene Representation
第一个部分是关于Scene Representation，Lumen分了两个部分来处理
* 几何表面材质
  * Lumen使用了Mesh Card对场景内的几何体进行简化的表达，一个Card就是一个简单的平面，这些Card可以离线的时候就生成好，但Card本身并不包含材质信息，而是通过运行时进行Capture生成Surface Cache来缓存材质信息
* Tracing的加速结构
  * Software Ray Tracing
    * 对于Software Ray Tracing，需要一个数据结构来加速Ray Tracing的过程，Lumen使用了SDF（Signed Distance Field）来做加速结构
  * Hardware Ray Tracing
    * 如果是Hardware Ray Tracing就直接使用了Ray Tracing的Acceleration Structure

## Mesh Card
Mesh Card可以看作是物体表面的一个简化表达
![20250602160513](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602160513.png)

Mesh Card是离线生成的，在导入的时候就会生成Mesh Card，Card存储了最基本的物体表面的几何信息
![20250602161042](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602161042.png)

Mesh Card的生成代码逻辑：
* 构建入口函数：FMeshUtilities::GenerateCardRepresentationData
  * ![20250602192629](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602192629.png)
  * InitSurfelScene：生成Surfel
    * ![20250602193325](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602193325.png)
    * GenerateSurfelsForDirection：生成某个方向的Surfel，也就是+-X，+-Y，+-Z，6个方向
      * 通过Embree做Raytracing生成Surfel
      * ![20250602193711](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602193711.png)
  * BuildSurfelClusters：对Surfel进行聚类生成Surfel Cluster
  * SerializeLOD：通过Surfel Cluster生成Mesh Card

## Surface Cache
第二个部分是Surface Cache，Mesh Card只包含了Direction、OBB这些信息，并不包含表面的材质信息，Lumen选择在运行时Cache表面的材质信息，这个Cache称为Surface Cache，之所以选择在运行时Cache是为了兼容一些动态的情况，比如UV流动这些材质信息可能会发生变化的情况，通过运行动态获取材质信息能够兼容更多的情况。
![20250602014911](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602014911.png)

为了性能，Surface Cache的更新是通过分帧更新处理的，通过Budget限制每帧最多更新的数量。
![20250602162044](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602162044.png)

通过GPU的使用的信息来决定Surface Cache的更新（Feedback），通过Feedback可以减少Surface Cache的更新数量，只更新哪些需要用到的Surface Cache。
![20250603121636](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603121636.png)

Surface Cache的数据主要就是包含了表面的基础的材质信息，类似GBuffer。
![20250602163616](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602163616.png)

## Implementation
Surface Cache的更新流程：
* FDeferredShadingSceneRenderer::Render
  * BeginUpdateLumenSceneTasks：创建一个异步的任务更新Lumen Scene
    * UpdateSurfaceCachePrimitives：从PrimitiveProxy上更新MeshCards信息
    * UpdateSurfaceCacheMeshCards：根据MeshCards生成Surface Cache Requests
      * UpdateSurfaceCacheFeedback：根据Feedback信息生成Surface Cache Requests
      * 这里也会限制每帧更新的Request数量（r.LumenScene.SurfaceCache.CardCapturesPerFrame）
      * ![20250603103803](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603103803.png)
    * ProcessLumenSurfaceCacheRequests：处理Surface Cache Requests，生成FCardPageRenderData
    * AddCardCaptureDraws：根据FCardPageRenderData生成MDC或者是Nanite绘制需要的信息
  * BeginGatherLumenLights：创建异步任务，获取Lumen相关的Light信息
  * UpdateLumenScene：提交FCardPageRenderData的MDC
    * ResampleLightingHistory：将缓存的Lighting信息重新映射到新的Card Pages上
    * 提交MDC，这里就是进行实际的Surface Cache的渲染，将物体表面的材质信息缓存下来
    * ![20250603120956](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603120956.png)

# Lighting
第二个部分就是光照计算，在Lumen中就是对前面的Surface Cache进行光照计算并且缓存光照的结果，然后用于后续的Propagation达到Indirect Lighting的效果。

为了达到Multi-bounce的Indirect Lighting，Lumen在对Surface Cache进行光照计算的时候还会缓存一些Indirect Lighting的结果，这样多帧累积之后可以达到Infinite Bounce的Indirect Lighting效果。
![20250602170159](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602170159.png)

这里同样也是做了分帧的优化，每帧通过Budget控制最多更新的Pages数量，通过优先级排序，优先更新当前需要用到的哪些Pages
![20250602171041](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602171041.png)

Lumen中Direct Lighting的计算
* 将Surface Cachde的Pages拆分成8x8的Tile
* 每个Tile选择优先级最高的N个Light计算光照
* 生成ShadowMask
* 计算光照结果
![20250602171102](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602171102.png)

Indirect Lighting的计算，多帧迭代的结果，通过缓存上一帧的Indirect Lighting结果在下一帧使用
![20250602171119](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602171119.png)

## Implementation
Lumen中Lighting的计算流程：
* FDeferredShadingSceneRenderer::Render
  * RenderLumenSceneLighting：计算Surface Cache的光照
    * RenderDirectLightingForLumenScene：计算直接光照
      * CullDirectLightingTiles：生成需要光照的Tile
        * SpliceCardPagesIntoTiles：将Card Pages切分成Tile
        * CalculateCardTileDepthRanges：计算每个Tile的Depth Range，用于后续做Light Culling
        * BuildLightTiles：计算每个Tile相关的Light信息，生成Light Tile
        * ComputeLightTileOffsetsPerLight：统计Light相关的Tile数量的Prefix Sum
        * InitializeLightTileIndirectArgs：写入Indirect Args
        * CompactLightTiles：合并Light Tile
      * ComputeShadowMaskFromLightAttenuation：为Light Tile计算Shadow Mask，这里是用的ShadowMap确定遮挡关系，如果没有ShadowMap没有覆盖到，会写入到一个Buffer中，后续再做Ray Tracing进一步确认遮挡关系
      * InitShadowTraceIndirectArgs：生成Shadow Trace的Indirect Args
      * Offscreen shadows：生成Offscreen Shadows，也就是Shadow Map没有覆盖到的部分
        * TraceLumenHardwareRayTracedDirectLightingShadows：如果开启了HW Tracing，使用硬件光追计算Shadow
        * TraceDistanceFieldShadows：否则使用Distance Field Shadows
      * RenderDirectLightIntoLumenCardsBatched：结合前面生成的Shadow Mask计算直接光照
      * CombineLumenSceneLighting：更新最终光照结果
    * RenderRadiosityForLumenScene：计算Indirect Lighting
      * SpliceCardPagesIntoTiles：将Card Pages切分成Tile
      * AddRadiosityPass：添加Radiosity Pass
        * BuildRadiosityTiles：构建Radiosity Tiles
        * IndirectArgs：生成Indirect Args
        * HardwareRayTracingRGS/DistanceFieldTracing：使用硬件光追或者是Distance Field Tracing计算Indirect Lighting
        * SpatialFilterProbes：对Indirect Lighting进行空间滤波
        * ConvertToSH：将Tracing的结果转成SH系数
        * Integrate：根据前面的Probe SH计算Tile的Indirect Lighting
      * CombineLumenSceneLighting：更新最终光照结果

# Propagation
最后一个阶段就是Propogation，也就是将光照结果从之前Lighting的物体表面传播到其他物体表面，这个一般也是整个全局光照中最复杂，消耗最高的部分。

Lumen的方案大致是通过在屏幕空间摆放一些Probe，然后通过Tracing计算Probe的光照结果，最终通过Probe的插值结合全分辨率的GBuffer信息得到最终的光照结果。
![20250612102911](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250612102911.png)
![20250612102903](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250612102903.png)

整体的Pipeline结构如下图所示：
![20250612103126](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250612103126.png)

## Screen Probe
Screen Probe以Octahedral atlas的形式存储Probe的结果
![20250612103602](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250612103602.png)

Adaptive Placement，在一些高频变化的区域，Probe的分布会更密集一些，比如深度变化比较大的区域，提高Probe的利用率
![20250612103827](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250612103827.png)

每帧通过Jitter抖动Probe的位置，结合Temporal Filter
![20250612104221](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250612104221.png)

## Tracing
Lumen的整个Tracing管线综合了Screen Tracing、Hardware Ray Tracing、Software Ray Tracing等多种方式，是一个Hybrid ray tracing方案。
![20250603184903](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603184903.png)

按照Tracing的距离，首先是屏幕空间的Screen Tracing，可以直接通过深度信息HZB进行Tracing，性能比较好，而且由于屏幕空间信息是比较高频，可以在一些细节的部分得到比较好的效果，但是屏幕空间的Tracing只能处理屏幕上可见的部分，所以只能用在距离比较近的部分。
![20250603185123](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603185123.png)

如果Screen Tracing没有Hit的情况就需要用其他方案处理，Lumen这里实现了两种Tracing方式，一种是Software Raytracing，另一种是Hardware Raytracing。

Software Raytracing也就是不依赖硬件的光追特性，所以可以兼容更多的设备，另外Software Raytracing也是完全自主可控的，有更多的扩展性。
![20250603185346](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603185346.png)

Software Raytracing主要是基于SDF进行Tracing，Lumen用到了两种SDF：
* Mesh SDF：每个Mesh单独的SDF，可以得到更精确的结果
* Global SDF：全局的SDF，通常是一个Voxel Grid，Tracing的精度会低一些，但性能会更好
![20250603185538](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603185538.png)

另一个方案就是依赖硬件的光追特性的Hardware Raytracing，相比Software Raytracing，Hardware Raytracing可以得到更精确的Hit结果，但需要硬件支持。
![20250603185924](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603185924.png)

所以整体上Lumen的Tracing是一个层级结构，按照距离依次进行处理，只有前面没有Tracing到的部分才会继续往下继续Tracing
![20250603203350](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603203350.png)

在Tracing打到物体表面之后，我们需要获取到Hit表面的光照结果，Lumen实现了两种方式
* Surface Cache：第一种就是用前面Surface Cache的Lighting结果
* Hit Lighting：第二种就是直接在Hit的时候进行光照计算
![20250603202630](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603202630.png)

## Importance Sampling
Lumen做的一些优化，第一个是Importance Sampling，Lumen通过BRDF以及Lighting计算PDF来进行Importance Sampling，这样可以显著减少采样结果的方差，得到噪声更低的结果。
![20250603204717](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603204717.png)
![20250603204736](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603204736.png)

## World Space Radiance Cache
World Space Radiance Cache，也就是世界空间的Radiance Cache，类似Irradiance Volume，相比屏幕空间，World Space更加稳定，可以优化一些距离比较远的Indirect Lighting的结果。
![20250612110649](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250612110649.png)
![20250603204830](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603204830.png)

## Implementation
Lumen的Final Gather逻辑流程：
* FDeferredShadingSceneRenderer::Render
  * DispatchAsyncLumenIndirectLightingWork
    * ![20250603184436](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603184436.png)
    * 开启r.Lumen.DiffuseIndirect.AsyncCompute的情况下，提前创建异步任务开始做Final Gather
    * RenderLumenFinalGather：Lumen的Final Gather，也就是生成屏幕空间的最终的Indirect Lighting
      * RenderLumenIrradianceFieldGather/RenderLumenReSTIRGather/RenderLumenScreenProbeGather：根据配置执行Gather，默认是Screen Probe
        * RenderLumenScreenProbeGather
          * UniformPlacement：使用均匀分布生成Screen Probe
          * AdaptivePlacement：使用自适应分布生成Screen Probe
          * SetupAdaptiveProbeIndirectArgs：生成Adaptive Probe的Indirect Args
          * GenerateBRDF_PDF：生成BRDF的PDF
          * UpdateRadianceCaches：更新World Space Radiance Cache
            * MarkUsedRadianceCache：标记需要用的Probe
            * UpdateCacheForUsedProbes：复用上一帧的Radiance Cache结果
            * ClearRadianceCacheUpdateResources
            * AllocateUsedProbes：分配需要Tracing的Probe，并且根据Probe的距离计算优先级
            * SelectMaxPriorityBucket：根据Budget以及前面计算的Probe优先级，选择优先级最高的N个Probe
            * AllocateProbeTraces：根据前面的Budget限制之后的数量，写入Probe的Tracing信息
            * SetupProbeIndirectArgsCS：构建Indirect Args
            * ClearProbePDFs：清理PDF Buffer
            * ScatterScreenProbeBRDFToRadianceProbes：用Screen Probe的BRDF PDF信息更新Radiance Cache的PDF
            * GenerateProbeTraceTiles：生成Probe的Tracing Tile信息
            * SetupTraceFromProbesCS
            * RenderLumenHardwareRayTracingRadianceCache/TraceFromProbes：计算Probe的Tracing的结果
            * FilterProbeRadiance：用附近的Probe进行滤波
            * FixupBordersAndGenerateMips
          * GenerateImportanceSamplingRays
            * ComputeLightingPDF：生成Lighting的PDF
            * GenerateRays：生成采样的Ray，如果开启了Importance Sampling的话，会根据PDF进行采样
          * TraceScreenProbes：对Screen Probe进行Tracing
            * TraceScreen：用HZB进行屏幕空间的Tracing
            * Hardware：
              * RenderHardwareRayTracingScreenProbe：Hardware的Tracing
                * NearField：近距离的Tracing，只处理前面ScreenTracing Miss的部分
                  * SurfaceCache/HitLighting
                    * ![20250603235011](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603235011.png)
                    * 这里会根据开关决定Lighting的方式，SurfaceCache就是直接复用前面Surface Cache的Lighting结果，如果是HitLighting的话，就会重新计算Lighting，消耗也是会更高
                * FarField：远距离的Tracing，处理前面Screen跟Near都Miss的部分
            * Software：
              * TraceMeshSDFs：Mesh SDF Tracing
              * TraceVoxels：Global SDF Tracing
          * FilterScreenProbes
            * CompositeTraces：计算Probe的Radiance
            * CalculateMoving：计算Probe的Moving
            * TemporallyAccumulateRadiance：Temporal Filter
            * FilterRadianceWithGather：Spatial Filter
            * ScreenProbeConvertToIrradiance：从Radiance转换到Irradiance
            * FixupBorders
            * GenerateMip：生成Mips
          * ComputeScreenSpaceShortRangeAO：根据开关r.Lumen.ScreenProbeGather.ShortRangeAO确定是否计算短距离的AO
            * RenderHardwareRayTracingShortRangeAO：用Hardware Tracing计算AO
            * ShortRangeAO_ScreenSpace：屏幕空间的Tracing计算AO
          * InterpolateAndIntegrate：采样Screen Probe的结果，生成最终的Indirect Lighting结果
            * TileClassificationMark：因为不同材质可能需要不同的Integrate方式，用Tile进行分类加速优化
            * TileClassificationBuildLists：构建List
            * ScreenProbeIntegrate：采样Screen Probe的结果，生成最终的Indirect Lighting结果
          * UpdateHistoryScreenProbeGather：更新History数据
      * ComputeLumenTranslucencyGIVolume
        * UpdateRadianceCaches：更新Translucency的Radiance Cache
        * HardwareRayTraceTranslucencyVolume/TraceVoxelsTranslucencyVolume：Tracing
        * SpatialFilter：对Tracing结果进行空间滤波
        * Integrate：生成最终的Translucency Volume的Indirect Lighting结果
    * RenderLumenReflections：渲染Lumen Reflections
      * ReflectionTileClassification：进行Tile分类，标记需要Reflection处理的部分
      * GenerateRays：生成Ray信息
      * TraceReflections：
        * TraceScreen：屏幕空间Tracing
        * RenderLumenHardwareRayTracingReflections/TraceMeshSDFs+TraceVoxels：根据开关配置选择用Hardware Ray Tracing或者是Software Ray Tracing
      * ReflectionsResolve：Upscaling Resolve
      * Temporal Denoise：做Temporal降噪
      * Spatial Denoise：做Spatial降噪
  * RenderDiffuseIndirectAndAmbientOcclusion
    * RenderLumenFinalGather：如果前面没有走Async的异步流程，这里会计算Final Gather
    * RenderLumenReflections：如果前面没有走Async的异步流程，这里会计算Reflections
  * RenderDiffuseIndirectAndAmbientOcclusion
    * DiffuseIndirectComposite：Composite最终结果