---
layout: post
title:  "graphics-principle & PBR"
date:   2018-11-21 10:00:00 +0800
categories: engine
---
### graphics-principle & PBR

## 渲染是什么
渲染是一个 CPU 驱动引擎，引擎驱动 OpenGL，OpenGL 驱动 GPU 修改显存的过程。<br>
VRAM（显存）包括图像缓冲区、深度缓冲、顶点缓冲区、纹理图。<br>

Application -> Command -> Geometry -> Rasterization -> Fragment -> Display<br>

## 渲染管线详解

1. 描述了3D模型的几何数据结构;
2. 几何体分为三角形。将变换应用于模型（即模型的三角形）称为模型变换;
3. 我们有一个与眼睛相对应的虚拟相机。在相机上应用的变换称为“视图变换”;
4. 为了提供3D效果，应用透视投影将3D模型转换为2D空间。这些被称为投影变换;
5. 模型视图投影变换转换对象并将其放入屏幕空间。这些由MVP矩阵描述;
6. 栅格化三角形以获得像素（它们被称为片段，直到到达最后阶段）;
7. 片段有阴影。它可以像使用插值颜色或复杂一样简单（应用光照和纹理来获得真实感）;
8. 片段被组合（混合）以获得单个像素。该组合可以是任何仲裁操作，如混合，或者像向观察者显示最近的像素一样简单;

![](/images/graphics-principle1.png)<br>

概括来说，面剔除（遮挡剔除等），光栅化之后片段着色，最后输出颜色到显存中（图像缓存区）。<br>

>A shader is a small computer program that performs the necessary calculations to describe the appearance of a surface. 

![](/images/graphics-principle3.png)<br>

一个简单的着色器。vert&frag <br>

{% highlight ruby %}
sampler2D _MainTex;
float4 _MainTex_ST;

v2f vert (appdata v)
{
	v2f o;
	o.vertex = UnityObjectToClipPos(v.vertex);
	o.uv = TRANSFORM_TEX(v.uv, _MainTex);
	UNITY_TRANSFER_FOG(o, o.vertex);
	return o;
}

fixed4 frag (v2f i) : SV_Target
{
	// sample the texture
	fixed4 col = tex2D(_MainTex, i.uv);
	// apply fog
	UNITY_APPLY_FOG(i.fogCoord, col);
	return col;
}
{% endhighlight %}

>使用shader language 编写的程序称之为shader program（着色程序）。着色程
>序分为两类：vertex shader program（顶点着色程序）和fragment shader program
>（片断着色程序）。为了清楚的解释顶点着色和片断着色的含义，我们首先从阐
>述GPU 上的两个组件：Programmable Vertex Processor（可编程顶点处理器，又
>称为顶点着色器）和 Programmable Fragment Processor（可编程片断处理器，又
>称为片断着色器）

整个渲染过程中，顶点处理器和片段处理器是属于可编程图形渲染管线。<br>
顶点着色器控制顶点坐标转换过程；片段着色器控制像素颜色计算过程。这样就区分出顶点着色程序和片段着色程序的各自分工：<br>
Vertex program 负责顶点坐标变换；<br>
1. 对象空间 -> 相机空间
2. 相机空间 -> 齐次裁剪空间
3. 齐次裁剪空间 -> 窗口空间

一般段数据包括 像素深度、顶点插值的颜色、差值得出的纹理坐标、像素位置等。<br>

Fragment program 负责像素颜色计算；<br>
像素最终的颜色可以简单地设置为顶点颜色的插值结果，也可以是纹理映射图中相应的数值，也可以是 PBR 计算出来的结果；<br>

Vertex program 的输出是 Fragment program 的输入。<br>

输出颜色之前会执行片段测试。<br>

1. 像素包含测试
2. 裁剪测试
3. Alpha测试
4. 模板测试
5. 深度测试
6. 混合 -> 图像缓冲区

部分测试，GPU 会提到段着色之前，避免计算很多不需要的像素，以提高性能。<br>

## CPU 和 GPU 在渲染中承担的角色

1. CPU 处理平截头体剔除和遮挡剔除以及BSP等场景管理算法，识别出潜在可视的网格实例，并把它们以及材质提供给 GPU 渲染（DrawCall）； 
2. GPU 几何阶段和光栅化，上面已经详细介绍过了；

![](/images/graphics-principle2.png)<br>

## 光照模型以及 PBR
1. 漫反射
2. 镜面反射
3. PBR

参考书籍<br>
游戏引擎架构<br>
3D 游戏与计算机图形学中的数学方法<br>
GPU 编程与CG 语言之阳春白雪下里巴人<br>