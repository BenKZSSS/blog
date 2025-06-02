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

Surface Cache的数据类似GBuffer，主要就是包含了表面的基础的材质信息
![20250602163616](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602163616.png)

# Direct Lighting
![20250602170159](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602170159.png)
![20250602171041](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602171041.png)
![20250602171102](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602171102.png)
![20250602171119](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250602171119.png)

# Propagation