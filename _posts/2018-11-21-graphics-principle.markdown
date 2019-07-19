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

{% highlight CG %}
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
### 光照模型
**漫反射<br>**

粗糙的物体表面向各个方向等强度地反射光，这种等同地向各个方向散射的现象称为光的漫反射（diffuse reflection）。<br>
diffuse = Kd x lightColor x max(N · L, 0)<br>
1. Kd is the material's diffuse color,
2. lightColor is the color of the incoming diffuse light,
3. N is the normalized surface normal,
4. L is the normalized vector toward the light source, and
5. P is the point being shaded.

**自发光<br>**

emissive = Ke<br>
1. Ke is the material's emissive color.

**高光<br>**

specular = Ks x lightColor x facing x (max(N · H, 0)) shininess <br>
(pow shininess) <br>
1. Ks is the material's specular color,
2. lightColor is the color of the incoming specular light,
3. N is the normalized surface normal,
4. V is the normalized vector toward the viewpoint,
5. L is the normalized vector toward the light source,
6. H is the normalized vector that is halfway between V and L,
7. P is the point being shaded, and
8. facing is 1 if N · L is greater than 0, and 0 otherwise.

![](/images/graphics-principle6.png)<br>

**环境光<br>**

ambient = Ka x globalAmbient<br>
1. Ka is the material's ambient reflectance and
2. globalAmbient is the color of the incoming ambient light.

**最终像素<br>**

surfaceColor = emissive + ambient + diffuse + specular<br>
基于上面的算法，我写了个测试 Shader，通过 Shader Toggle 切换显示 Final Surface Color 各个组成部分。<br>

{% highlight CG %}
fixed3 lightDir = normalize(UnityWorldSpaceLightDir(worldPos));
float3 viewDir = normalize(UnityWorldSpaceViewDir(worldPos));
float3 halfDir = normalize(lightDir + viewDir); 

// sample the texture
fixed4 albedo = tex2D(_MainTex, i.uv.xy);
albedo = lerp(albedo, _Color * albedo, i.color.a);
//emissive = Ke
//fixed3 emissive = tex2D(_EmissiveTex, i.uv);
half rim = 1.0 - saturate(dot (normalize(viewDir), normal));
fixed3 emissive = _RimColor.rgb * pow (rim, _RimPower);
//ambient = Ka x globalAmbient
fixed3 ambient = albedo.xyz * UNITY_LIGHTMODEL_AMBIENT.xyz;
//diffuse = Kd x lightColor x max(N · L, 0)
fixed3 diffuse = albedo.xyz * _LightColor0.rgb * max(0, dot(normal, lightDir));
//specular = Ks x lightColor x facing x (max(N · H, 0)) shininess
fixed3 specular = _Specular * _LightColor0.rgb * pow(max(dot(normal, halfDir), 0), _Shininess * 128) * albedo.a;
//fixed3 specular = halfDir * 0.5 + 0.5;
// apply fog

#if ENABLE_EMISSIVE
	return fixed4(emissive, 1.0); 
#elif ENABLE_AMBIENT
	return fixed4(ambient, 1.0);
#elif ENABLE_DIFFUSE
	return fixed4(diffuse, 1.0);
#elif ENABLE_SPECULAR
	return fixed4(specular, 1.0);
#endif

fixed4 color = fixed4(emissive + ambient + diffuse + specular, 1.0)
UNITY_APPLY_FOG(i.fogCoord, color);
return color;
{% endhighlight %}

See more detail...
[LowpolyPBR](https://github.com/nashnie/Shader/blob/master/LowpolyPBR.shader)<br>

**PBR<br>**
PBR 关键点：<br>
1. 光照现象，漫反射并不是各个方面平均发散，微表面模型(NDF)；
2. 菲涅尔定理(Fresnel)，光源在边角处有更明亮的反光；
3. 能量守恒，反射的光不能超过入射的光，遮挡因素，越光滑镜面越集中越亮；
4. 线性空间；

PBR 算法主要有三项，F & D & G<br>
![](/images/graphics-principle4.png)<br>

F 是 Fresnel 反射系数（Fresnel reflect term），表示反射方向上的光强占原始光强的比率；<br>
D 表示微平面分布函数（Beckmann distribution factor），返回的是“给定方向上的微平面的分数值”；<br>
G 是几何衰减系数（Geometric attenuation term），衡量微平面自身遮蔽光强的影响。N 、V 、L 分别表示法向量、视线方向（从顶点到视点）和入射光方向（从顶点向外）；<br>

UNITY_BRDF_PBS 需要传入以下参数：<br>

{% highlight CG %}
albedo 反射率
specularTint 镜面高光颜色
oneMinusReflectivity 1 - Metallic（1 - _Metallic）
Smoothness 1 - 粗糙度
normal 法线
viewDir 镜头方向
light 直接光
indirectLight 间接光
{% endhighlight %}

网上有很多 PBR 的 shader 我就不班门弄斧了，因为我也不太熟。<br>

To be continue...

参考书籍<br>
游戏引擎架构<br>
3D 游戏与计算机图形学中的数学方法<br>
GPU 编程与CG 语言之阳春白雪下里巴人<br>
http://developer.download.nvidia.com/CgTutorial/cg_tutorial_chapter05.html