---
title: UE5/Lumen
published: 2025-06-04
description: ''
image: ''
tags: [rendering, graphics, ue5, lumen, global illumination]
category: 'UE5'
draft: true 
lang: 'zh'
---

# Introduction
Lumen是UE5中全新的全局光照方案

* Scene Representation
* Direct Lighting
* Propogation

# Scene Representation

## Mesh Card
Lumen对于场景的表达主要是通过Mesh Card，Mesh Card可以看作是物体表面的一个简化表达
![20250602160513](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602160513.png)
![20250602161042](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602161042.png)

* 构建入口函数：FMeshUtilities::GenerateCardRepresentationData
  * ![20250602192629](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602192629.png)
  * InitSurfelScene：生成Surfel
    * ![20250602193325](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602193325.png)
    * GenerateSurfelsForDirection：生成某个方向的Surfel，也就是+-X，+-Y，+-Z，6个方向
      * ![20250602193711](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602193711.png)
      * 通过Embree做Raytracing生成Surfel
  * BuildSurfelClusters：对Surfel进行聚类生成Surfel Cluster
  * SerializeLOD：通过Surfel Cluster生成Mesh Card

## Surface Cache
Mesh Card只包含了Direction、OBB这些信息，并不包含表面的材质信息，Lumen选择在运行时Cache表面的材质信息，这个Cache称为Surface Cache
![20250602014911](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602014911.png)

Surface Cache的更新也是通过分帧更新进行优化
![20250602162044](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602162044.png)

通过GPU的使用的信息来决定Surface Cache的更新（Feedback）
![20250603121636](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603121636.png)

Surface Cache的数据类似GBuffer，主要就是包含了表面的基础的材质信息
![20250602163616](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602163616.png)

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
    * ![20250603120956](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603120956.png)
    * 提交MDC，这里就是进行实际的Surface Cache的渲染，将物体表面的材质信息缓存下来

# Lighting
第二个部分是根据前面的简化的场景信息，计算光照，比较简单直接的就是计算直接光照，一般第一次反弹的贡献是最大的，如果不考虑Multi-bounce的话，直接光照就可以满足大部分的需求。如果需要Multi-bounce，一般需要缓存Indirect Lighting的结果，然后用Indirect Lighting再进行Propogation达到Multi-bounce的结果。

Lumen会对Surface Cache进行直接光照的计算，也支持了缓存Indirect Lighting的结果用于Multi-bounce的计算。
![20250602170159](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602170159.png)

这里同样也是做了分帧的优化，每帧通过Budget控制最多更新的Pages数量，通过优先级排序，优先更新当前需要用到的哪些Pages
![20250602171041](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602171041.png)

Direct Lighting的计算
* 将Pages拆分成8x8的Tile
* 每个Tile选择优先级最高的N个Light计算光照
* 生成ShadowMask
* 计算光照结果
![20250602171102](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602171102.png)

Indirect Lighting的计算，多帧迭代的结果，通过缓存上一帧的Indirect Lighting结果在下一帧使用
![20250602171119](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602171119.png)

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
  * DispatchAsyncLumenIndirectLightingWork
    * ![20250603184436](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603184436.png)
    * 开启r.Lumen.DiffuseIndirect.AsyncCompute的情况下，提前开始做

# Propagation
最后一个阶段就是Propogation，也就是将光照信息从之前Lighting的物体表面传播到其他物体表面，这个一般也是整个全局光照中最复杂，消耗最高的部分。

Lumen中，Propagation是基于Tracing来实现的，为了性能，整个Tracing管线综合了Screen Tracing、Hardware Ray Tracing、Software Ray Tracing等多种方式
![20250603184903](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603184903.png)

Screen Tracing基于GBuffer和HZB，可以处理一些高频的、细节的信息
![20250603185123](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603185123.png)

Software Raytracing，相比Hardware Raytracing更加可控，也可以用于一些无法支持Hardware Raytracing的平台或者设备
![20250603185346](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603185346.png)

Software Raytracing主要是基于SDF
![20250603185538](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603185538.png)

Hardware Raytracing相比来说，可以生成更加精确的结果，可以用来做高精度反射
![20250603185924](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250603185924.png)