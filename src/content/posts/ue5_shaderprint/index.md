---
title: UE5/Shader Print
published: 2025-06-21
description: ''
image: ''
tags: [rendering, graphics, ue5, shaderprint]
category: 'UE5'
draft: false 
lang: 'zh'
---

# Introduction
Shader Print是UE5中一个非常有用的调试工具，它可以帮助开发者在Shader中输出调试信息，方便进行Shader的调试和优化。

相比于传统的调试方式，Shader Print可以解决一些基于截帧的调试方式的痛点：
* 只能调试单帧，如果问题是偶现在某些帧上，就很难截到有问题的帧
* 目前越来越多的渲染方案大量使用CS进行计算，而CS并不直接对应屏幕上的像素，很难直接定位到出问题的Thread
* 图形渲染中很多数值都是坐标、颜色之类的数据，直接从数值上很难判断是否有问题

而Shader Print可以通过在Shader中直接通过代码进行一些判断，这样可以只针对有问题的情况进行输出，而且输出的方式也更加直观，比如可以输出字符串、颜色、绘制射线等等。

![img_v3_02nc_ea228a79-0957-4649-b728-4b73cab26cbg](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/img_v3_02nc_ea228a79-0957-4649-b728-4b73cab26cbg.jpg)

# Shader Print的使用

## 绑定Shader Print的Uniform参数
首先需要在Shader中绑定Shader Print的Uniform数据

![20250621192841](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621192841.png)
![20250621193239](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621193239.png)

## 打开Shader Print
* 通过CVars打开Shader Print
r.ShaderPrint 1
* 在代码中调用ShaderPrint::SetEnabled
![20250621202734](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621202734.png)

## 在Shader中使用Shader Print
include Shader Print的头文件
![20250621193716](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621193716.png)

初始化Context结构
![20250621193745](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621193745.png)

然后就可以通过这个Context调用Shader Print的方法进行调式信息的打印输出

* Print
![20250621200503](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621200503.png)
![20250621200423](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621200423.png)

* CheckBox
![20250621201726](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621201726.png)
![20250621201700](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621201700.png)

* Slider
![20250621202152](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621202152.png)
![20250621202138](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621202138.png)

* Triangle
![20250621202526](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621202526.png)
![20250621202514](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621202514.png)

* Quad
![20250621202904](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621202904.png)
![20250621202850](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621202850.png)

* Line
![20250621203059](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621203059.png)
![20250621203046](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621203046.png)

* AABB
![20250621203259](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621203259.png)
![20250621203247](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621203247.png)

* Cross
![20250621203433](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621203433.png)
![20250621203401](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621203401.png)

* Circle
![20250621203603](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621203603.png)
![20250621203552](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621203552.png)

* Sphere
![20250621203717](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621203717.png)
![20250621203700](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621203700.png)

* Referential
![20250621211643](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621211643.png)
![20250621211623](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621211623.png)

# Shader Print的实现原理
以AddLineWS为例，当我们在Shader中调用AddLineWS方法时，实际上是往一个Buffer中添加Line的数据
![20250621212847](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621212847.png)

AllocateLineElement通过InterLockedAdd原子操作获取新的Line的Index，本质上就是一个无锁队列，这样就可以保证多线程同时调用的时候不会出现冲突
![20250621213050](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621213050.png)

![20250621213303](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621213303.png)

然后在Frame的末尾，再通过单独的一些Pass将这些Buffer中的数据进行渲染输出
![20250621214948](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621214948.png)

这里还有一种比较特殊的数据类型，就是字符串，Shader中本身是没有字符串类型的，所以最终肯定需要将字符串类型转换成整型然后再存储到Buffer中，UE的实现是在Shader的编译的预处理阶段解析TEXT宏，然后将TEXT宏中的字符串转换成一个StringID，再通过这个ID去索引到对应的字符串数据
![20250621214431](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621214431.png)

然后生成一些全局的字符串表数据，再通过字符串ID索引相关数据
![20250621214647](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621214647.png)

最终生成的代码类似这样
![20250621214811](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621214811.png)
![20250621214829](https://image-1258012845.cos.ap-guangzhou.myqcloud.com/20250621214829.png)