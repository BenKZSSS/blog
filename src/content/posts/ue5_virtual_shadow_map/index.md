---
title: UE5/Virtual Shadow Map
published: 2025-08-18
description: ''
image: ''
tags: [rendering, graphics, ue5, shadow, virtual shadow map, vsm]
category: 'UE5'
draft: true 
lang: 'zh'
---

# Introduction
Virtual Shadow Map (VSM) 是UE5中一个新的阴影技术（https://dev.epicgames.com/documentation/en-us/unreal-engine/virtual-shadow-maps-in-unreal-engine）

VSM本质上也还是Shadow Map的一种变体，但相比一些

# Implementation

## Virtual Pages & Physical Pages

## Clipmap

## Caches

## Render Shadow Map

* FShadowSceneRenderer::RenderVirtualShadowMaps
  * FVirtualShadowMapArray::BuildPageAllocations
    * InitPageRectBounds:初始化Pages的RectBounds
    * MarkCoarsePages:先粗略标记哪些Pages是需要用到的
    * PruneLightGrid:对LightGrid进行剔除，只保留VSM相关的Light
    * MarkPixelPages:根据Pixel的Depth信息，标记哪些Pages是需要用到的
    * UpdatePhysicalPages
    * PackAvailablePages
    * FVirtualShadowMapArray::AppendPhysicalPageList(EmptyToAvailable=true)
      * AppendPhysicalPageList
      * AppendPhysicalPageList(Counts)
    * AllocateNewPageMappings:给新Page分配Physical Page
    * GenerateHierarchicalPageFlags:生成Hierarchical Page Flags
    * PropagateMappedMips
    * InitializePhysicalPages
      * SelectPagesToInitialize:选择需要初始化的Pages
      * InitializePhysicalMemoryIndirect
    * Feedback Status
    * FVirtualShadowMapArray::AppendPhysicalPageList(EmptyToAvailable=false)
      * AppendPhysicalPageList
      * AppendPhysicalPageList(Counts)
  * FShadowSceneRenderer::RenderVirtualShadowMaps
    * FVirtualShadowMapArray::RenderVirtualShadowMapsNanite
      * CompactViewsVSM:生成Nanite Views
      * Nanite::FRenderer::DrawGeometry:通过Nanite渲染VSM
    * FVirtualShadowMapArray::RenderVirtualShadowMapsNonNanite
    * FVirtualShadowMapArray::PostRender

## Shadow Projection
* FShadowSceneRenderer::RenderVirtualShadowMapProjectionMaskBits
  * RenderVirtualShadowMapProjectionOnePass:如果LocalLights开启了OnePass Projection
    * RenderVirtualShadowMapProjectionCommon
* FDeferredShadingSceneRenderer::RenderDeferredShadowProjections
  * FShadowSceneRenderer::ApplyVirtualShadowMapProjectionForLight
    * Directional Light:处理方向光的VSM投影
      * RenderVirtualShadowMapProjection:生成VSM Projection
        * RenderVirtualShadowMapProjectionCommon
          * VirtualShadowMapProjection:从深度还原位置信息，然后从VSM生成Projection
        * CompositeVirtualShadowMapMask:将VSM的Projection合并到最终的ShadowMask
    * Non Directional Light:处理非方向光的VSM投影
      * OnePass Projection
        * CompositeVirtualShadowMapFromMaskBits
      * Non OnePass Projection
        * RenderVirtualShadowMapProjection
          * RenderVirtualShadowMapProjectionCommon
          * CompositeVirtualShadowMapMask

## Shadow Map Ray Tracing(SMRT)