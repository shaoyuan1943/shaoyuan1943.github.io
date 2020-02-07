---
layout: post
title: "Unity.Image实现技能冷却遮罩效果"
date:   2016-01-31
categories: Game-Dev
---

前几日听到另外一个项目组在讨论实现技能冷却的遮罩效果，UE那边想要这样的效果：  

![alt text](/img/2016-02-01-1.png) 

其中遮罩的快慢由技能的冷却时间决定。我在旁边听了一会儿最终他们的实现方案是：做一个半透明灰度的遮罩图片放在一个Sprite2D里，然后在单位时间里计算需要遮罩透明的面积，再通过shader渲染时将面积内的取样点的alpha值修正为全透明。我在旁边听了半天感觉这是非正统的方式，总觉得他们搞的太麻烦了，于是找了下Unity3D的文档，发现对于技能冷却遮罩效果，有更加简单方便的实现方式。

先放上文档链接[**这里**](http://docs.unity3d.com/Manual/script-Image.html)，采用Unity.Image即可完美解决这个问题。

在Unity中，Image会作为一个Sprite2D存在。Image有一个很重要的特性：图像填充。如果要实现技能冷却的遮罩效果，只需要在技能图标上放置一个Image组件，然后选择如下属性：

![alt text](/img/2016-02-01-2.png) 

其中：  
1. Image Type：决定当前图片显示方式，Filled为填充方式。  
2. Fill Method：填充方式，这里选择Radial 360，即以当前Image大小的中心360度填充。仅当Image Type为Filled才有效。其他的填充方式看一眼就明白。  
3. Fill Origin：填充起始点，仅当Image Type为Filled才有效。  
4. Fill Amount：填充率，从0到1，0代表无填充，即隐藏了当前图片，1代表图片全部显示。

既然了解了Unity.Image本身就自带了这种效果就好办了：  

    image.fillAmount -= fTime * Time.deltaTime;

代码中fTime为当前技能冷却单位时间。就这样一行代码搞定技能冷却遮罩的效果。