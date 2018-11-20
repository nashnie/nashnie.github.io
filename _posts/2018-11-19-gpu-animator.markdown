---
layout: post
title:  "GPU 加速动画渲染方案"
date:   2018-11-19 10:00:00 +0800
categories: rendering
---
## GPU 加速动画渲染方案

### GPU Animator 解决的问题
同屏动画太多导致的 CPU 蒙皮计算压力太大。比如 MOBA 类游戏的几十个小兵，或者竞技场周围的吃瓜群众等等，<br>
这些动画一般不需要很好的效果，可以尝试使用多种优化手段来降低效果（也可以保持效果基本不变）、占用内存、消耗GPU 来降低 CPU 的压力。<br>

### GPU Animator 原理

有两种 GPU 加速模式：<br>
1. 缓存每一帧顶点坐标，顶点着色器根据帧数和顶点编号获取顶点数据进行渲染，此为方案 A；
2. 缓存每一帧骨骼变换矩阵，顶点着色器计算蒙皮，此为方案 B；

方案A，编辑器每帧运行 AnimationClip，获取转换后的顶点坐标保存到数组中，根据数组长度计算需要的 Texture 最小尺寸，（Texture用来保存顶点数据）。<br>

{% highlight C# %}
private void CalculateTextureSize(ref int textureSize, int totalCount)
{
	textureSize = 2;
	while (textureSize * textureSize < totalCount)
	{
		textureSize = textureSize << 1;
	}
}
{% endhighlight %}

### 总结

A 方案：<br>
1. 优点几乎不占用cpu和gpu消耗；
2. 缺点动画文件体积较大（30帧左右，大概3M大小）；

B 方案：<br>
1. 优点动画文件体积比原生动画文件还小，不占用CPU，GPU计算蒙皮；
2. 缺点GPU压力；
