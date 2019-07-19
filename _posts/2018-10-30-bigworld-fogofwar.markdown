---
layout: post
title:  "大地图战争迷雾实现原理"
date:   2018-10-30 21:45:00 +0800
categories: gameplay
---
# 大地图战争迷雾（未完待续）
## 原理以及难点
战争迷雾这个技术不是特别的难点，很多年前的游戏已经使用了战争迷雾这个技术，比如war3系列等等，但是目前流行起来的**大地图**的战争迷雾确实有不好实现的地方，比如一张贴图无法表示整个地图细节，多个贴图要相互拼接实现效果，贴图之间的过度以及贴图内存管理等等。<br>
本文主要讲以下几点，传统迷雾效果实现，包含迷雾数据结构
、迷雾边缘效果处理（抗锯齿）、迷雾 Shader、如何 debug 迷雾、多线程加速迷雾计算、FOV算法以及大地图下迷雾如果处理；<br>
## 实现以及部分代码
### 1，迷雾数据设计
{% highlight ruby %}
VisionRange&VisionUpRange：迷雾范围和高度
StartX&StartY：玩家起点
LevelWidth&LevelHeight ：地图宽高
Factions：阵营
VisionBlocker(ResistenceMapData)：地图阻挡
Revealed Coverd Undiscover：地图迷雾状态
FogTexture：迷雾贴图数据
BlurRenderTexture：抗锯齿贴图数据
FogColor：迷雾颜色
Scale:地图精度比例
...
{% endhighlight %}
### 2，迷雾算法以及多线程加速迷雾计算
场景根据格子（1mx1m）划分，格子按比例（1:1或者1:2）对应贴图的像素。<br>
场景静态物件以及地形等使用高度图保存，我们的大地图一共有五百多张高度图。<br>
计算迷雾氛围时，根据当前位置获取高度图的高度，然后比较 VisionRange、VisionUpRange，视野可以达到同时高度大于目标高度的的话设置 FogTexture 贴图的 r 通道为 1.0f。<br>
我们把贴图的不同通道分别作不同的用途，r 表示被占领的状态，g 用来记录根据占领状态插值的结果。<br>
迷雾 FOV 算法，八个象限，依次遍历，遇到阻挡（当前视野高度无法穿过目标地形高度）就返回，最后的效果一个个小格子实现的近似圆的范围，比如把这个近似圆变成光滑的圆就要用到另一个算法，抗锯齿算法了。<br>
**多线程加速**，迷雾因为算法的问题，其实计算量是非常大的，尤其是512*512的图。怎么优化呢？我们使用多线程，我们把 Texture 的数据和计算分开，每一帧数据准备之后，把以上的迷雾范围计算，迷雾过渡的计算都移动到Thread里计算，然后再设置回Texture，只有抗锯齿没有办法移动到线程计算里，这样大大提高了性能。<br>
同时，这里提一句，我们因为场景建筑物很多，墙体很薄，因为贴图精度问题，导致了迷雾漏光的效果，大家可以想象一下这个诡异的效果，我尝试了很多办法去解决这个问题，比如建筑独立出来，调整角色碰撞半径使得角色不能靠墙太近，角色靠近墙体时候根据角色的面向动态计算迷雾、墙体加厚等等，非常麻烦同时效果不好，最后怎么解决呢？`提高贴图精度`，用256的贴图表示128的区域，1比0.5，甚至可以1比0.25，最后实现的效果就很完美，策划和美术都很满意。<br>

**地图高度图**，地图某块区域，图中左下角是房子，房子墙体上的明显的矩形白色是门，一个个的斑点是树木或者石头等阻挡物<br>

![ResistenceMap.h](/images/ResistenceMap107.png)<br>

### 3，迷雾抗锯齿
比较常规的抗锯齿算法，是取每一个像素上下左右周围四个像素的颜色值，然后求平均值就是该点的最后值；我们使用 Unity 的 RenderTexture来实现这个功能。<br>

{% highlight ruby %}
fixed4 fragDownsample (v2f_tap i) : SV_Target
{
    fixed color  = tex2d (_MainTex, i.uv20);
    color  = tex2d (_MainTex, i.uv21);//(-0.5h, -0.5h)
    color  = tex2d (_MainTex, i.uv22);//(0.5h, -0.5h)
    color  = tex2d (_MainTex, i.uv23);//(-0.5h, 0.5h)
    return color / 4;
}
{% endhighlight %}
### 4，迷雾 Shader
根据像素 worldPos 以及地图宽高、地图起点，计算出 UV 坐标；<br>
根据上面的迷雾Texture计算出迷雾的比例，0就是没有迷雾，1就是被迷雾覆盖，0.5是覆盖一半；<br>
最后根据计算比例插值算出迷雾颜色就好了。迷雾叠加在片段着色器最后计算得出的颜色值上。<br>

{% highlight ruby %}
fixed4 fragFogOfWar (float3 WorldPos) : SV_Target
{
    float2 UV = float2(Remap(WorldPos.x, Origin.x, Origin.x + LevelWidth * Scale, 0, 1), Remap(WorldPos.z, Origin.z, Origin.z + LevelHeight * Scale, 0, 1));
    fixed4 Fog = tex2d(FogOfWar, UV);
    fixed4 FinalColor = lerp(FogColor, fixed4(1, 1, 1, 1), Fog.g);
    FinalColor.a = Fog.g;
    fixed Result = fixed4(1, 1, 1, 1)
    return Result * FinalColor;
}
{% endhighlight %}
大家如果要修改Unity默认的Shader，比如地形、草这些，从官网下载之后在 Fragment 修改完放入自己项目的 Shader 文件夹就可以了。相对路径不能变。<br>
### 5，Debug 迷雾
我们在运行时把FogTexture的数据复制到另一张 Editor 环境下的 Texture，动态显示，同时加了开关显示单一通道，显示地形高度图等等。<br>

**Debug 迷雾**<br>

![](/images/fogEditor2.png)<br>

![](/images/fogEditor1.png)<br>

### 6，大地图迷雾
地图过大，同时要求精度又很高的情况下，无法用一张贴图表示出地图的迷雾地形，我最后使用了多张贴图（五六百张）来表示地图高度，同时使用一张贴图表示当前地图一块区域，玩家在不同的区域加载不同的地形高度图，然后刷新当前区域的迷雾Texture，但是玩家移动如何处理呢？<br>
我想到办法类似之前做页游时代的**滚屏机制**，玩家身上带一个Rect区域，迷雾 Texture 分内圈和外圈，Rect 跟随玩家移动，Rect 离开到贴图内圈时，开始切换贴图，这样既解决了迷雾贴图不能太大，又解决了切地图的突兀感（无法接受的一闪而过），而且还避免了在两块区域之间频繁切换的问题。<br>
注意的是，切换地图时候关闭掉迷雾从无到有的缓动效果，会加强突兀感。
### 7，优化迷雾算法？
做到迷雾效果渲染没问题了，但是CPU还有优化空间，CPU消耗根据地图Size指数增长，需要优化。尤其是项目有突然要求玩家的视野变大的需求，比如玩家使用狙击枪瞄准的时候。<br>

希望能帮到你。