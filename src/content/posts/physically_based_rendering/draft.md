---
title: Physically Based Rendering (Draft)
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
* Why PBR?

# 光源（Light Source）
* Quantities
  * Raidometry
  * Photometry
  * Colorimetry
* 光源类型（Light Source Types）
  * Punctual Light
  * Sun/Directional Light
  * Area Light
* UE5
* 光源参数对照表

# 材质（Material）
* BxDF
  * BRDF
  * Subsurface Scattering
  * Hair
* UE5
  * ShadingModel
  * Substrate

# 参与介质（Participating Media）
* Physics
* UE5
  * Fog
    * Exponential Height Fog
    * Volumetric Fog
    * Local Volumetric Fog
  * Atmosphere
  * Cloud

# 遮挡（Occlusion）
* Direct Lighting
  * Shadow
* Indirect Lighting
  * Ambient Occlusion
* UE5
  * Shadow
    * Shadow Map
      * Cascade Shadow Map
      * Virtual Shadow Map
    * Ray Traced Shadow
  * Ambient Occlusion
    * Screen Space Ambient Occlusion
    * Ray Traced Ambient Occlusion

# 全局光照（Global Illumination）
* Diffuse
  * UE5
    * Lightmap
    * Lumen
* Specular
  * UE5
    * Screen Space Reflections
    * IBL
    * Lumen

# 相机（Camera）
![20250509115726](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250509115726.png)
* Exposure
* Tonemapping
* White Balance
* Post Process
* UE5
  * Auto Exposure
  * ACES Tonemapper
  * Post Process Volume
    * Color Grading
    * ...