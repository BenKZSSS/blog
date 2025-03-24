---
title: UE5/Hair-Rendering
published: 2025-03-23
description: ''
image: ''
tags: [ue5, hair, rendering, graphics]
category: 'UE5'
draft: true 
lang: 'zh'
---

毛发是一种常见的特殊材质，一般我们需要使用专门的技术来渲染。毛发渲染的特殊性主要体现在两个方面：
* Geometry: 毛发的几何形状是非常复杂的，通常使用曲线来表示。而且由于毛发的数量非常多，所以需要使用特殊的技术来处理。
* Lighting：毛发的光照效果也是比较特殊的，毛发表面的角质层的特殊结构会产生特殊的BxDF，而毛发本身是透明的，所以也要考虑透射的影响。
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-14.png)

常见的毛发渲染方案有几种：
* Mesh-based：Mesh-based是最简单的一种方案，就是直接使用几何体来表示毛发，通过Teture+Hair Shading Model来模拟毛发的效果，一般很多风格化的游戏会使用这种方案。
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-10.png)
* Card-based：Card-based是另一种比较常见的方案，通过面片来表示毛发的几何形状，通过Teture+Hair Shading Model来模拟毛发的效果，也是目前游戏或者实时渲染中比较常见的方案。
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-11.png)
* Strand-based：Strand-based是一种比较复杂的方案，通过曲线来表示毛发的几何形状，通过模拟真实的毛发发丝来模拟毛发的效果，一般用于CG过场动画或者电影渲染中，实时的话对于性能要求比较高。
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-12.png)
* Shell-based：Shell-based是另外一种常见的方案，通过不断调整毛发覆盖的Mesh的Scale或者Offset进行多次渲染，然后通过Texture表示毛发的覆盖区域，用这种方式模拟毛发的Geometry，但这种方案一般用在短一点的动物绒毛上，头发的话效果不是很好。
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-13.png)

# Mesh-based / Card-based
Mesh-base/Card-based方案中，Hair的Geometry直接通过Mesh进行表达，通过Texture来表达更加细小的发丝效果，在UE5中，Hair的Mesh跟其他普通的Mesh没有太大的区别，最主要的还是在材质上，UE中内置了Hair Shading Model用来进行毛发的渲染。
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-15.png)
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-16.png)

## Hair Shading Model
Hair Shading Model的实现主要在HairBsdf.ush/HairShading中，主要参考的两篇Paper：
* Marschner et al. 2003, "Light Scattering from Human Hair Fibers"

头发由两个主要部分组成：表皮层和皮层。表皮层是一个薄的保护性外鞘，围绕着内部的皮层。表皮层对光散射特别重要，因为它形成了纤维与空气之间的界面。它由像屋顶瓦片一样重叠的扁平细胞组成（见图2），使纤维看起来像是一组嵌套的圆锥。由于它们的重叠排列，鳞片表面会略微但系统地偏离纤维表面的整体正常状态，其表面朝向纤维的根部端倾斜约3°

皮层构成了纤维的大部分。中心是含有色素的核心，即髓质。皮层和髓质中的色素决定了头发的颜色。

Marschner的这篇Paper中，Hair的散射模型主要分三个部分：
* R: 从外表面直接反射，Specular向发根方向偏移

* TT: 穿透头发产生的透射，浅色头发的Forward Scattering更强

* TRT: 从外表面反射，穿透头发，然后再次反射
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-18.png)

发丝Scattering是一个4D的function，可以拆为两个部分M项跟N项，也就是两个方向的切面
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image.png)

Scattering Function：
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-22.png)

M项通过高斯拟合：
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-21.png)

N项：
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-23.png)
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-24.png)
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-26.png)

* Pekelis et al. 2015, "A Data-Driven Light Scattering Model for Hair"

Pekelis的Paper也是基于Marshner的模型，但是将TRT项拆成了TRT+GLINT以解决Marshner中Sigularities的问题：
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-27.png)

同时也用其他更好的方式进行计算拟合

M项，使用Logistic代替Gaussian：
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-28.png)

N项：
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-30.png)

![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-31.png)
![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-32.png)

总的来说，Hair的BSDF还是挺复杂的，一大堆的公式，但是UE5中已经实现了这些公式，我们只需要在材质中选择Hair Shading Model就可以了🤓。

# Strand-based
UE5中Strand-based的毛发渲染方案主要是通过Groom来实现的，Groom是一种新的Asset类型，用于表示毛发的几何形状，可以通过Groom Asset来创建Groom，然后通过Groom Component来渲染Groom。

## Groom

## Strand-based Hair Rendering
下面详细看下UE5中Strand-based的毛发渲染流程：

* `FDeferredShadingSceneRenderer::Render`

	从`FDeferredShadingSceneRenderer::Render`开始，此处省略一些不相关的流程
	
	* `RunHairStrandsBookmark(ProcessStrandsInterpolation)`
	
		在BasePass之后，先通过Bookmark API来做一些HairStrands的预处理，这里主要是做StrandsInterpolation
		
		* `RunHairStrandsInterpolation_Strands`

			这里主要是对Strands进行各种模拟计算，更新Postion等等

	* `RenderHairPrePass`
		
		* `CreateHairStrandsMacroGroups`
			
			根据HairInstance的AABB进行聚类，把HairInstance按照空间进行划分，生成MacroGroup

		* `VoxelizeHairStrands`
			
			对HairStrands进行体素化

			* `AddVirtualVoxelizationRasterPass`
				
				用CS来做三维光栅化，得到Voxel，存储的主要是Coverage以及一些辅助的Mask标记信息，Voxel信息以VoxelPageTexture的形式存储
				![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-5.png)

			* `AddVirtualVoxelInjectOpaquePass`
				
				判断体素是否被Opaque遮挡，采样SceneDepth判断CloestDepth

			* `AddVirtualVoxelGenerateMipPass`
				
				生成Voxel Mip

		* `RenderHairStrandsDeepShadows`

			如果LightSource上开启了Cast Deep Shadow
			
			这里会渲染HairStrands的DeepShadow，DeepShadow后续用于计算HairStrands的Transmittance

			* `AddHairDeepShadowRasterPass(FrontDepth)`
				
				生成HairStrands的FrontDepth，也就是CloestDepth
				![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-6.png)

			* `AddHairDeepShadowRasterPass(DeepOpacityMap)`
				
				生成HairStrands的DeepOpacityMap，计算每个Layer的Coverage
				![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-7.png)

	* `RenderHairBasePass`

		* `RenderHairStrandsVisibilityBuffer`

			这里UE实现了好几种RenderMode，默认为HairVisibilityRenderMode_MSAA_Visibility，这里为了简化流程，先只看这种模式

			MSAA_Visibility模式主要是通过MSAA来计算HairSamples，因为毛发本身的结构太小了，基本都在一个像素内，需要进行SubPixel的Sample来获得比较好的效果

			* `AddHairViewTransmittancePass`
				
				计算Coverage，写入TransmittanceBuffer，这里的Transmittance是ViewSpace的Transmittance
				![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-4.png)

			* `AddHairStrandsGenerateTilesPass`

				生成Tile，区分每个Tile的HairStrands覆盖情况也就是前面生成的View Transmittance，根据Coverate信息确定这个Tile是Partial/Full/None，同时也会生成IndirectDrawBuffer，用于后续的DrawCall

				用Tile进行绘制可以只覆盖Hair区域，可以简化后续的很多Pass

			* `AddHairVisibilityFillOpaqueDepth`

				采样SceneDepth，生成Tile的DepthBuffer

			* `AddHairVisibilityMSAAPass`

				光栅化Hair，写入MaterialID，也就是VisibilityBuffer，因为Hair会造成很高的Overdraw，通过VisibilityBuffer分离Rasterization跟Material BasePass进行优化

			* `AddHairVisibilityControlPointIdCompactionPass`

				裁剪掉一些无效的像素，输出需要被处理的Sample，UE将这些数据称为Node

			* `AddHairMaterialPass`

				走Material计算，得到GBuffer等材质信息，写入NodeDataBuffer，计算Velocity，写入NodeVelocityBuffer

			* `AddHairVelocityPass`

				从NodeVelocityBuffer中采样，输出ScreenSpace的Velocity

			* `AddUpdateSampleCoveragePass`

				按照Depth进行排序，更新Coverage，默认是关闭的状态

			* `AddHairMaterialDataPatchPass`

				把一些其他需要的信息输出到SceneBuffers，比如Depth，这里只会输出Coverage为1的像素，也就是完全覆盖的像素

			* `AddHairOnlyDepthPass`

				* `AddHairAuxilaryPass`

					输出HairOnlyDepth，这里的Depth与前面`AddHairMaterialDataPatchPass`中的Depth不同，HairOnlyDepth会输出所有的Hair像素，包括Partial的像素

			* `AddHairOnlyHZBPass`

				构建HairOnly的HZB

	* `RenderLights`

		* `RenderHairStrandsTransmittanceMask`

			计算TransmittanceMask，这里有两种方式计算TransmittanceMask，一种是DeepShadow，一种是Voxel

			* `AddHairStrandsDeepShadowTransmittanceMaskPass`

			* `AddHairStrandsVoxelTransmittanceMaskPass`

		* `RenderLightForHair`

			计算Hair Lighting，这里就是计算Hair的Lighting，走Hair ShadingModel，计算BxDF

			这里值得注意的是，Lighting的计算没有在ViewSpace进行，而是直接在HairSample Space进行，这样可以减少一些计算量

		![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-8.png)

	* `RenderTranslucency`

		* `RenderHairComposition`

			Hair的Composition，将前面计算的HairSample的Lighting进行合成

			* `AddHairDOFDepthPass`

				这里对DOF的Depth单独做了处理，因为Hair是SubPixle的，对于DOF来说，对于Partial的Hair像素，无论是用Hair的Depth还是直接用SceneDepth都会有问题

				这里的处理方案是根据Coverage在SceneDepth和HairDepth之间进行插值，得到DOF的Depth

			* `AddHairVisibilityComposeSamplePass`

				采样累计HairSample的Lighting结果，输出到SceneBuffer

				![alt text](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/ue5_hair_rendering/image-9.png)

			* `AddHairVisibilityFastResolveMaskPass`

				输出FastResolveMask，这里的FastResolve是在前面`AddHairVelocityPass`阶段输出的，对于Velocity大于阈值的像素，会走FastResolve，这样可以减少TAA的模糊

				```cpp
				// If the velocity is above a certain threshold, the pixel will be resolve with a fast resolved. 
				// This will result into a sharper, but more noisy output. However it sill avoid getting smearing 
				// from TAA.
				bNeedFastResolve = bNeedFastResolve || NeedFastResolve(EncodedVelocity.xy, VelocityThreshold);
				```

			* `AddHairVisibilityGBufferWritePass`

				补全Scene的GBuffer，用于给后续的Pass使用