---
layout: post
title:  "graphics-principle"
date:   2018-11-21 10:00:00 +0800
categories: engine
---
### graphics-principle

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
输出颜色之前会执行片段测试。<br>

1. 像素包含测试
2. 裁剪测试
3. Alpha测试
4. 模板测试
5. 深度测试
6. 混合>>>>>图像缓冲区

>A shader is a small computer program that performs the necessary calculations to describe the appearance of a surface. 

## 渲染语法
可编程的渲染管线。<br>
顶点着色器和片段着色器。<br>

![](/images/graphics-principle3.png)<br>

一个简单的顶点片段着色器。vert&frag。<br>

{% highlight ruby %}
Shader "Unlit/TestShader"
{
	Properties
	{
		_MainTex ("Texture", 2D) = "white" {}
	}
	SubShader
	{
		Tags { "RenderType"="Opaque" }
		LOD 100

		Pass
		{
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			// make fog work
			#pragma multi_compile_fog
			
			#include "UnityCG.cginc"

			struct appdata
			{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			struct v2f
			{
				float2 uv : TEXCOORD0;
				UNITY_FOG_COORDS(1)
				float4 vertex : SV_POSITION;
			};

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
			ENDCG
		}
	}
}
{% endhighlight %}

## CPU 和 GPU 在渲染中承担的角色

![](/images/graphics-principle2.png)<br>

## 渲染优化
