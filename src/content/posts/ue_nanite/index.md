---
title: UE5/Nanite
published: 2025-05-24
description: ''
image: ''
tags: [ue5, rendering, graphics, nanite, geometry]
category: 'UE5'
draft: true
lang: 'zh'
---

# Introduction
Nanite可以算是UE5最重要的几个新特性之一，可以使得开发者可以使用高精度的几何体进行渲染，而不需要担心性能问题。

关于Nanite想要做的事可以从Karis的分享中了解到：
![20250526230259](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250526230259.png)

如果用一句话总结就是对Geometry做虚拟化，如果用Virtual Teture来类比，VT技术使得我们可以将高精度的纹理用到我们的渲染中，其核心原理就是只加载跟只用我们需要用到的部分，这其实也是各种Virtual技术最朴素的原理，回到Geometry上，Nanite的核心原理也是如此，当我们渲染一个高精度的模型时，其实只有当这个模型拉得非常近时，我们才需要模型的细节，而距离比较远时，很多三角面已经远小于像素了，其渲染对于画面效果影响很小，但依然占用了性能，这是一种非常大的浪费，在Nanite之前我们可以通过制作多级LOD并通过渲染距离或者屏占比来切换，但这种方式有很多缺点：
* 需要制作LOD，并且调整适配距离参数，成本高
* LOD切换时会有明显的跳变
* 对于一些跨度比较大的模型，前后LOD之间的差异会很大，并不能通过一个单一的LOD模型处理这种情况

Nanite通过将模型切分成很多的Cluster，然后以Cluster为单位生成LOD，然后运行时通过计算Cluster在屏幕上的显示大小来决定LOD的级别，因为LOD是以Cluster为单位的，并且始终保持三角面在屏幕上有大概一致的大小，所以不会有明显的LOD切换跳变，同时也保证了不会出现使用过于精细的LOD模型渲染的情况。

![20250526232550](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250526232550.png)

切分Cluster另一个好处就是可以用GPU Driven渲染解决DC的压力，关于GPU Driven可以看一下Ubisoft之前在大革命时候的分享
[https://advances.realtimerendering.com/s2015/aaltonenhaar_siggraph2015_combined_final_footer_220dpi.pptx]

# Nanite Overview
我们先从大概的技术原理看下Nanite的整体流程，这里主要是按照Karis的分享介绍一下Nanite整体的技术方案

## Build
首先Nanite需要离线对模型进行处理，生成一些数据结构然后运行时使用

### Cluster
首先三角面会被分成很多的Cluster，然后为了达到无缝的LOD，最朴素的想法就是对每个Cluster进行LOD，然后运行时根据当前Cluster在屏幕上的大小来决定当前的LOD级别
![20250528113624](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250528113624.png)

但这就会遇到第一个问题就是LOD Cracks，因为相邻的Cluster可能会选择不同的LOD级别，这样就会导致相邻的Cluster之间出现接缝的问题
![20250527101211](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527101211.png)

### Cluster Group
Nanite的解决方案是锁边，也就是将一些相邻的Cluster合并成Group，然后Group之间的边是锁定的，LOD的简化只发生在Group内部，那我们只需要保证，在同一个Group中的Cluster，始终是选择相同的LOD，那么就不会出现Cracks的问题
![20250527112336](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527112336.png)

大致的流程可以看下图
* 将Cluster分成Group
* Group内做简化
* 再重新切分Cluster

![20250527112537](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527112537.png)

这样的话每个Cluster可能有多个父节点（Group重新拆分的Cluster），这样的话会形成一个DAG（有向无环图）

![20250527114039](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527114039.png)

所以，目前我们就得到了一堆Cluster Group，每个Group中包含了一些Cluster，我们在运行时就可以通过计算这些Cluster在屏幕上的大小来决定这个当前这个LOD级别的Cluster要不要渲染，但实际上如果我们对每个级别的Cluster都进行计算，即便每个计算都是可以并行的，但当场景中有很多Cluster时，这个计算的开销也是非常大的，所以Nanite这里又做了一个Hierarchy的处理，这部分Hierarchy结构也是在Build阶段预先生成好的

![20250528113453](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250528113453.png)

## Runtime
运行时，Nanite就需要根据当前View的信息以及离线处理好的一堆Cluster以及Hierarchy结构来决定哪些Cluster需要被渲染，然后进行光栅化计算，得到Visibility Buffer，然后再根据这个Buffer来生成GBuffer，最后进行Lighting计算。

### Culling
第一步，我们需要根据View信息，找到哪些Cluster需要被渲染，这里首先需要解决的就是不同LOD的Cluster的选择问题，因为前面我们已经通过Group的方式解决了LOD Cracks的问题，所以我们再运行时需要保证每个Group内的Cluster在计算LOD时是相同的，这里Nanite通过相同输入来避免进行一些同步的操作

![20250527114701](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527114701.png)

在Karis的分享里面还有提到通过PersistenThread来解决Hierarchy结构不平衡的问题，因为Hierarchy中每个层级的Node处理之后，下一个层级需要处理的任务数量其实是不确定的，如果按照每个Level一个个Pass处理，因为依赖关系，后一个Pass需要等前一个Pass处理完才能开始，这有可能会导致GPU的空闲
![20250527224728](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527224728.png)

Persistent Thread的想法其实就是通过一个MPMC的队列，来把所有Node的处理统一通过队列发送给Worker，这样减少GPU的空闲等待时间

**（不过因为不同硬件跟平台在Thread调度方式有所区别，目前PersistentThread的模式默认只在PS5上开启）**
![20250527224609](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527224609.png)

除了LOD的选择，我们还可以根据Cluster的Bounding Box来进行Culling，把一些不可见的Cluster剔除掉，视锥剔除比较简单，这里就不展开讨论了，值得讨论的主要是遮挡剔除，Nanite这里使用了一个Two-Pass的方式来进行遮挡剔除：
* 首先通过前一帧的HZB（Hierarchical Z Buffer）来进行遮挡剔除，得到一些可见的Cluster
* 然后光栅化这些Cluster，这样我们可以得到一个初步可见的Cluster的深度
* 然后通过这个Depth Buffer重新生成HZB，因为我们前面的第一次剔除是用的上一帧的HZB，所以剔除的结果是不准确的，可能有一些Cluster被错误的剔除掉了，所以这里我们把前面被剔除的Cluster用这个新的HZB再做一次测试，检查是不是真的被剔除，然后把错误剔除的、可见的Cluster再做光栅化
* 得到的Buffer生成最终的HZB

![20250527123433](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527123433.png)

### Rasterization
剔除之后，下一步需要做光栅化，这里Nanite通过软光栅处理小的三角面，避免一些硬光栅的额外开销

![20250527140359](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527140359.png)

对于大的三角面还是沿用硬光栅

![20250527141245](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527141245.png)

光栅化的结果就是生成了一张Visibility Buffer，这个Buffer存储了最基本的信息用于还原几何跟材质信息

![20250527142110](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527142110.png)

### BasePass
有了Visibility Buffer之后，下个阶段就是根据这些信息生成GBuffer，也就是Nanite Base Pass，这里Nanite用了一个非常巧妙的方式，将Materia ID转换成深度值，利用硬件的Depth Test机制快速剔除掉不相干的像素，每个材质在Shading时只会触发对应Pixel的Shader执行

**（不过目前最新的实现已经换成了Shade Binning的方式，虽然这种Depth Test的方式可以快速进行剔除，但是当Material比较多的时候，每个Pixel还是需要进行很多次的Test，Shade Binning的方式每个Pixel只会处理一次，然后将Pixel所在的Bin信息写入到Buffer中再统一做处理）**

![20250527144825](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527144825.png)

## Streaming
类似于VT，只有用到的数据才会被加载，Nanite也只会加载当前渲染到的几何数据，当模型距离较远的时候，细节的Cluster就不会被加载

Nanite的Streaming的单元是基于Cluster Group，原因跟前面LOD Cracks的时候类似，因为我们是已Group为单位进行LOD选择的，所以一个Group内只有所有Cluster都被加载之后才能正常渲染，所以我们加载的时候也应该以Group为单位

![20250527152121](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527152121.png)

因为Group的大小是可变的，所以每个Group会占用不同的内存大小，Nanite以Page为单位管理数据，所以每个Group会被拆分到多个Parts

![20250527152721](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527152721.png)

![20250527152921](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527152921.png)

然后在实际运行时，根据当前Cluster的裁剪结果，找到那些需要被加载的Page，然后CPU回读这些数据，然后再异步从Disk加载数据

![20250527153202](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527153202.png)

# Nanite Implementation
下面我们从5.5的代码看下Nanite的具体实现

## Build
* 入口函数，Nanite::FBuilderModule::Build
  * ![20250527153754](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527153754.png)
  * ClusterTriangles：第一步先对每个Mesh进行处理，生成Cluster
    * ![20250527153937](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527153937.png)
  * BuildDAG：生成Cluster的DAG结构
    * ![20250527154415](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527154415.png)
    * 过程就是重复以下步骤：
      * 把Cluster组合成Group
      * 然后对Group内的Cluster进行简化
      * Group内的Cluster再重新Split成新的Cluster
  * KeepPercentTriangles/TrimRelativeError：控制原始模型导入的三角面的数量
    * ![20250527160251](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527160251.png)
  * Fallback：生成Fallback的模型，用于一些不支持Nanite的情况
    * ![20250527160458](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527160458.png)
  * Encode：编码Cluster以及Group的数据
    * ![20250527161938](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527161938.png)
    * SanitizeVertexData：处理一些非法的数据，比如超出浮点精度范围的一些数值
    * RemoveDegenerateTriangles：清理一些退化的三角面，比如其中有一些两个顶点重合的三角面
    * BuildMaterialRanges：构建Cluster的材质信息
    * ConstrainClusters
    * BuildVertReuseBatches：计算材质Batch的三角面数量（不过目前没有找到这个数据具体是用在哪里的，可能是已经废弃了）
    * CalculateQuantizedPositionsUniformGrid：根据Cluster的Bounding Box范围对顶点位置进行Quantization
    * CalculateEncodingInfos：生成一些Encoding的基本信息
    * AssignClustersToPages：生成Page信息
    * BuildHierarchies：构建Hierarchy结构
    * WritePages：写入最终的Page数据，Page中包含了所有Cluster的信息，并且这里还会对数据进行一些压缩处理

## Runtime
* RenderNanite：在Base Pass之前，进行Nanite的Visibility Buffer的生成
  * DrawGeometry：整个Rasterization阶段的入口函数
    * PrepareRasterizerPasses：按照Raster Bin生成每个Bin的Rasterizer Pass，也就是Pass需要的一些渲染参数（Shader等），每个Bin可以看作是一种光栅化的方式或者代码逻辑，比方说，对于一般的Static Mesh以及标准材质，这些Mesh对应的光栅化逻辑是一样的，就可以放在一个Bin里面统一处理，但如果这个材质加上了WPO，那这个材质就没法跟标准材质放在同一个Pass处理，只能单独处理
    * Main Pass：Two-Pass Occlusion的第一个阶段
      ![20250527164207](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527164207.png)
      * AddPass_InstanceHierarchyAndClusterCull：进行裁剪剔除
        * InstanceCull：先按照每个Primitive Instance进行剔除
          * ![20250527165545](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527165545.png)
        * AddPass_NodeAndClusterCull：对Node以及Cluster进行剔除
          * ![20250527170028](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527170028.png)
          * 这里可以看到，Nanite还是实现了两种方案，PersistentThreadCulling以及按照Level进行层级剔除，而且默认只有在PS5上才会使用PersistentThreadCulling，所以看起来Persistent的这种模式目前只有在PS5上是明确有正向收益的，在其他硬件上可能性能收益并不明确
          * NodeCull：对Node级别进行剔除，Node也就是Build阶段生成的Hierarchy结构中的节点
            * 根据Node节点，根据视锥以及HZB进行剔除，然后计算Screen-Error决定是否需要更细节的Cluster，如果需要进一步细节的Cluster，则输出下一个层级的Node到Buffer中，否则直接输出Cluster信息
          * ClusterCull：对Cluster级别进行剔除，在完成所有Hierarchy的Node处理之后，处理目前得到的所有Cluster，进行视锥以及HZB的遮挡剔除，输出最终需要渲染的Cluster
      * AddPass_Rasterize：进行光栅化，生成Visibility Buffer
        * AddPass_Binning：生成Raster Binning
          * RasterBinCount：从前面裁剪生成的Cluster中读取材质信息，然后统计每个Bin的Cluster的数量
          * RasterBinReserve：根据BinCount的统计，给每个Bin预留出足够的空间用来写入Cluster的信息
          * RasterBinScatter：写入Raster Bin的数据，也就是每个Bin包含的Cluster的信息，然后这里还做了简单的Batch，所以写入的是Cluster的Index以及Range信息
          * RasterBinFinalize：一些最终的处理，主要是处理Mesh Shader的Indirect Args
        * DispatchHW：硬光栅化
          * ![20250527173343](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527173343.png)
          * 处理每个Rasterizer Pass，也就是调用每个Indirect Draw，每个Indirec Buffer里面已经存好了有多少Cluster需要被渲染
          * ![20250527174315](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527174315.png)
        * DispatchSW：软光栅化
          * ![20250527173412](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527173412.png)
          * 结构上是类似的，只不过软光栅化是通过Compute Shader来进行光栅化计算的
          * ![20250527174711](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527174711.png)
    * BuildHZB：通过Main Pass生成的Depth Buffer生成新的HZB
    * Post Pass：Two-Pass Occlusion的第二个阶段
      * ![20250527180243](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527180243.png)
      * 流程上跟Main Pass基本一致，区别是这里使用的新的HZB进行遮挡剔除，处理的是前面被剔除掉的Cluster
  * EmitDepthTargets：从Visibility Buffer中提取出深度信息，生成Depth Buffer
* Nanite::DispatchBasePass：渲染Nanite的Base Pass
  * ShadeBinning：生成Shade Binning，逻辑跟前面的Raster Bin比较类似，只不过前面Bin的是Cluster信息，这里Shade Bin的是Pixel的信息
  * ShadeGBuffer：渲染GBuffer
    * ![20250527182135](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250527182135.png)
    * 处理每个ShadeCommand，这里就是从Visibility Buffer中提取出几何信息以及材质信息，然后执行材质的Shader，生成GBuffer

# Issues
目前已知的一些，Nanite支持还不是很完善，或者会导致一些性能问题需要特别注意的点

## WPO（World Position Offset）
对于WPO材质，Nanite需要一些特殊处理，因为WPO会改变顶点的位置，而Nanite在构建阶段并没有WPO的信息，如果WPO导致顶点偏移过大，超出了原本Cluster/Node的BoundingBox，可能会导致裁剪剔除不正确
![20250528101528](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250528101528.png)

所以目前UE的处理方式是在材质上提供Max World Position Offset Displacement选项，通过这个选项来指定WPO可能偏移的最大距离
![20250528101718](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250528101718.png)

代码实现上就是通过扩大裁剪时候的BoundingBox来保证Culling的结果正确
![20250528102058](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250528102058.png)

第二个问题就是WPO因为改变了顶点坐标，而且计算的逻辑是在材质中独立的代码，而非一些固定的逻辑，所以，我们在光栅化的时候，这些材质的Cluster就没法跟标准的光栅化逻辑在同一个Pass处理，实际上除了WPO，其他所有会导致光栅化的逻辑不同于标准逻辑的材质都需要单独处理

在UE实现中，这部分材质都被归为VertexProgrammable，目前WPO、CustomUVs、Displacement都属于这个类型的材质
![20250528102850](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250528102850.png)

当我们使用了VertexProgrammable材质时，RasterMaterialProxy就会使用ShadingMaterialProxy，这样会导致这些材质的Cluster会被放到单独的Raster Bin中，当数量很多的时候会影响整体光栅化的并行效率
![20250528102738](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250528102738.png)
![20250528103907](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250528103907.png)

## Masking Materials
Masking材质其实会有一些类似的问题，因为Masking材质会改变Pixel级别的深度信息，所以光栅化的时候需要跑完整的Pixel的代码，这会进一步加重Rasterization的消耗

UE实现中，这部分材质被归为PixelProgrammable，目前Masking、PixelDepthOffset都属于这个类型的材质
![20250528110037](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250528110037.png)

## Skinned Meshes
目前（5.5）Skinned Mesh也支持了Nanite
![20250528111213](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250528111213.png)

对于Skinned Mesh，除了Raster Binning需要单独处理之外，在裁剪剔除的时候也做了一些Hack，目前为了保证Cluster不会被错误剔除，Nanite会将Skinned Mesh的Cluster的Bounding Box直接扩大到整个Instance的Bounding Box，因为这样保证了子节点的Bounding Box始终是在父节点的Bounding Box内的，也就满足了Error的单调性，所以在LOD Selection的时候也依然可以得到正确的结果，但这种方式的问题就是会影响Culling的效率，所以目前Skinned Mesh的Nanite官方还是标注的Beta
![20250528112051](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250528112051.png)