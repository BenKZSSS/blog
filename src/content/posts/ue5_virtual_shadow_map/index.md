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

VSM本质上也还是Shadow Map的一种变体，但相比一些传统的Shadow Map技术（如CSM、PerObject Shadow Map等），VSM在效果和性能上都有了很大的提升。

其实对于Shadow Map这一类技术，很多之前的技术比如CSM跟PerObject Shadow Map等，本质上都是在提高Shadow Map的分辨率跟利用率，本质上就是让ShadowMap的分辨率匹配其实际需要的分辨率。而VSM则是在这个思路上更近了一步，VSM从概念上来看其实就是一个超高分辨率的Shadow Map，但如果直接使用超高分辨率的Shadow Map肯定是不行的，性能上会有很大的问题，所以VSM则是将这个超高分辨率的Shadow Map进行了虚拟化处理，跟其他很多Virtual类技术一样，VSM的虚拟化也就是根据实际需要用到的部分对这个超高分辨率的Shadow Map进行切分，只处理那些真正需要的部分。

![20250819214200](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250819214200.png)

VSM从设计来说可以用于取代之前很多的Shadow Map技术，所以在开启VSM之后，很多之前的Shadow Map技术都会被禁用，比如CSM、PerObject Shadow Map等。
![20250820112637](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250820112637.png)

# Virtual Pages & Physical Pages
为了实现对超高分辨率Shadow Map的虚拟化，也就是对Shadow Map进行切分，从而能够够只处理那些真正需要被处理的部分，VSM将Shadow Map划分成了很多个小块，每个小块就是一个Page（128x128），这个概念可以类比虚拟内存里面按照页进行管理的概念。然后概念上的超高分辨率Shadow Map就变成了很多个Page的集合，这些Page就是Virtual Pages，而这些Virtual Pages当中只有一部分是我们需要用到的，而这些需要用到的Virtual Pages才会被映射到Physical Pages上，Physical Pages就是实际存储Shadow Map数据的地方。类似与虚拟内存，从Virtual Pages到Physical Pages的映射关系也需要一个中间的数据结构来进行管理，这个数据结构就是Page Table。

从Visualizer里面可以查看Page的覆盖情况，可以看到每个Page覆盖了光源阴影的一部分，然后靠近摄像机的地方单个Page覆盖的区域会更小一些，说明越靠近摄像机的地方用到的Shadow Map分辨率越高。
![20250819220601](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250819220601.png)
![20250819220548](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250819220548.png)

# Clipmap
对于Directional Light来说比较特殊一些，因为Directional Light的Shadow Map覆盖的是整个场景，对于这么大的范围，即便是超高分辨率的Shadow Map分辨率也是不足的，所以VSM对于Directional Light的Shadow Map使用了Clipmap的方案，也就是使用多个Shadow Map来覆盖不同距离范围的阴影信息，这样的话靠近相机的地方，同样是一个Page，其覆盖的区域会更小一些，从而达到更高的分辨率。这个思路其实是跟CSM类似的，但相比CSM，Clipmap只会根据相机距离来计算Clipmap Level，相比CSM还需要依赖视锥，这种方式会更加稳定，不会出现视角变化时大量Cache失效的问题。

![20250819221446](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250819221446.png)
![20250819221413](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250819221413.png)
![20250819221434](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250819221434.png)

# Render Shadow Map
下面我们看下VSM的渲染流程，主要分为两个部分，一个是Shadow Map的渲染，另一个是Shadow Projection，也就是生成ShadowMask信息。

渲染Shadow Map的流程：
* FShadowSceneRenderer::RenderVirtualShadowMaps:渲染VSM的入口
  * FVirtualShadowMapArray::BuildPageAllocations:构建Pages，这里会找到那些需要用到的Pages，然后给这些Pages分配Physical Pages
    * InitPageRectBounds:做一些初始化
    * MarkCoarsePages:标记一些粗略的Pages，比如Clipmap较高层级的Level以及LocalLight的最后一级Mip的Page
    * PruneLightGrid:对LightGrid进行剔除，只保留VSM相关的Light
    * MarkPixelPages:根据Pixel的Depth信息还原坐标，然后标记需要用到的Pages
    * UpdatePhysicalPages:更新Physical Pages
      * 这里会根据之前的缓存信息更新Physical Pages的状态，然后那些空闲的Pages会被Push到Empty Pages列表中
    * PackAvailablePages:将LRU列表中的Pages Push到Available Pages列表中
    * FVirtualShadowMapArray::AppendPhysicalPageList(EmptyToAvailable=true):将Empty Pages列表中的Pages Push到Available Pages列表中
      * 因为前面已经先将LRU列表中的Pages Push到Available Pages列表中，这里再将Empty Pages列表中的Pages Push到Available Pages列表中，这样就可以保证在Pop时会先用那些空闲的Pages，然后当没有空闲的Pages时再使用LRU列表中的Pages
    * AllocateNewPageMappings:给新Page分配Physical Page
      * 这里会给那些还没有分配Physical Page的Pages分配Physical Page，也就是从Available Pages列表中Pop出一个Physical Page来分配给这些Pages
    * GenerateHierarchicalPageFlags:生成Hierarchical Page Flags
    * PropagateMappedMips
    * InitializePhysicalPages
      * SelectPagesToInitialize:选择需要初始化的Pages
      * InitializePhysicalMemoryIndirect
    * Feedback Status:写入一些Feedback信息，Available Pages的数量以及GlobalResolutionLodBias
    * FVirtualShadowMapArray::AppendPhysicalPageList(EmptyToAvailable=false):将剩余的Available Pages列表中的Pages Push到Requested Pages列表中，给下一帧使用
  * FShadowSceneRenderer::RenderVirtualShadowMaps
    * FVirtualShadowMapArray::RenderVirtualShadowMapsNanite
      * CompactViewsVSM:生成Nanite Views
      * Nanite::FRenderer::DrawGeometry:通过Nanite渲染VSM
        * Nanite对于VSM渲染做了特殊的处理，在Culling的过程中会根据VSM的Page信息来进行Culling，从而只渲染那些真正需要的部分
    * FVirtualShadowMapArray::RenderVirtualShadowMapsNonNanite
    * FVirtualShadowMapArray::PostRender

# Shadow Projection
得到了Shadow Map之后，接下来就是生成ShadowMask，也就是Projection的过程

Projection的CPU侧的流程如下：
* FShadowSceneRenderer::RenderVirtualShadowMapProjectionMaskBits
  * RenderVirtualShadowMapProjectionOnePass:如果LocalLights开启了OnePass Projection
    * RenderVirtualShadowMapProjectionCommon
    * ![20250820103641](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250820103641.png)
* FDeferredShadingSceneRenderer::RenderDeferredShadowProjections
  * FShadowSceneRenderer::ApplyVirtualShadowMapProjectionForLight
    * Directional Light:处理方向光的VSM投影
      * RenderVirtualShadowMapProjection:生成VSM Projection
        * RenderVirtualShadowMapProjectionCommon
          * VirtualShadowMapProjection:从深度还原位置信息，然后从VSM生成Projection
          * ![20250820103942](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250820103942.png)
        * CompositeVirtualShadowMapMask:将VSM的Projection合并到最终的ShadowMask
    * Non Directional Light:处理非方向光的VSM投影
      * OnePass Projection:处理开启了OnePass Projection的情况
        * CompositeVirtualShadowMapFromMaskBits:将VSM的Projection合并到最终的ShadowMask
      * Non OnePass Projection:处理没有开启OnePass Projection的情况
        * RenderVirtualShadowMapProjection
          * RenderVirtualShadowMapProjectionCommon
            * VirtualShadowMapProjection:从深度还原位置信息，然后从VSM生成Projection
          * CompositeVirtualShadowMapMask

# Shadow Map Ray Tracing(SMRT)
VSM对于软阴影做了特殊的方案，也就是Shadow Map Ray Tracing(SMRT)，不同于传统的PCF，SMRT是通过在Shadow Map上进行Ray Tracing来实现软阴影的效果。

相比于PCF，SMRT更加贴近真实的阴影效果，在靠近遮挡体的地方，阴影会更加锐利，而在远离遮挡体的地方，阴影会更加柔和。

![20250820104334](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250820104334.png)
![20250820112436](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250820112436.png)

在做实际的SMRT之前，会先用DepthBuffer做一个ScreenRayCast来计算RayOffset，防止Self Shadowing的问题。
![20250820111344](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250820111344.png)

然后根据RayCount的配置，进行SMRTRayCast，也就是在Shadow Map上进行Ray Tracing，类似各种屏幕空间上的Tracing技术，比如SSR之类的。
![20250820111518](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250820111518.png)
