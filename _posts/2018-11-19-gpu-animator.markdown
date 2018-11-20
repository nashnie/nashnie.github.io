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

### 方案A
**Step1** 编辑器每帧运行 AnimationClip，获取转换后的顶点坐标保存到数组中，根据数组长度计算需要的 Texture 最小尺寸，（Texture用来保存顶点数据）。<br>

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

然后把顶点数据 xyz 当成 rgb，保存到 Texture 中，保存成引擎可以支持的任意资源格式都可以。<br>

**Step2** 运行时，设置我们想要播放动画的速度，Update 驱动帧 CurrentFrame，计算顶点的开始 Index，设置给 Shader，<br>

{% highlight C# %}
//因为每一帧都要记录所有定点的位置，所以顶点的开始Index就是totalVerts * currentFrame
currentPixelIndex = totalVerts * currentFrame;
{% endhighlight %}

**Step3** Shader 渲染部分，<br>

SV_VertexID，我们用他来区分每一个顶点，加上我们上一步计算的出来的 PixelIndex，就是我们 Vertex 的坐标，如何转成 UV 呢？<br>
根据我们顶点保存的规则，来计算，很简单，如下<br>

{% highlight C# %}
inline float4 getUV(float startIndex)
{
	float y = (int)(startIndex / _SkinningTexSize);
	float u = (startIndex - y * _SkinningTexSize) / _SkinningTexSize;
	float v = y / _SkinningTexSize;
	return float4(u, v, 0, 0);
}
{% endhighlight %}

获取 UV 之后，根据 UV 坐标取我们计算好缓存起来的 vertex 坐标输出到片段着色器中。<br>

{% highlight C# %}
VertexOutput vert(VertexInput input, uint vid : SV_VertexID)
{
	VertexOutput output;
	UNITY_SETUP_INSTANCE_ID(input);
	float startPixelIndex = _StartPixelIndex + vid;
	float4 uv = getUV(startPixelIndex);
	float4 vertex = tex2Dlod(_SkinningTex, uv);
	output.vertex = UnityObjectToClipPos(vertex);
	output.uv = input.uv;
	return output;
}
{% endhighlight %}

这个方案：<br>
1. 优点几乎不占用cpu和gpu消耗；
2. 缺点动画文件体积较大（30帧左右，大概3M大小）；

### 方案B
**Step1** 类似方案A，记录每一帧的关节 matrix，使用三个 rgb 来保存 matrix 的三行。<br>

{% highlight C# %}
meshTexturePixels[pixelIndex] = new Color(matrix.m00, matrix.m01, matrix.m02, matrix.m03);
pixelIndex++;
meshTexturePixels[pixelIndex] = new Color(matrix.m10, matrix.m11, matrix.m12, matrix.m13);
pixelIndex++;
meshTexturePixels[pixelIndex] = new Color(matrix.m20, matrix.m21, matrix.m22, matrix.m23);
pixelIndex++;
{% endhighlight %}

最后保存到 Texture 里，不一样的一点是，需要保存一个新的 Mesh 文件，因为骨骼的顺序必须和我们保存的 Texture 对应上。<br>

**Step2** <br>
perFramePixelsCount，上文我们讲了需要三个像素。totalJoints，是我们Mesh的总关节数。同样的，Update 驱动帧 CurrentFrame。<br>

{% highlight C# %}
currentPixelIndex = pixelsStartIndex + totalJoints * currentFrame * perFramePixelsCount;
{% endhighlight %}

**Step3** Shader 渲染部分，<br>
根据设置的 StartPixelIndex，取Matrix，然后计算顶点、法线等<br>

{% highlight C# %}
inline float4x4 getMatrix(float startIndex)
{
	float4 row0 = tex2Dlod(_SkinningTex, getUV(startIndex));
	float4 row1 = tex2Dlod(_SkinningTex, getUV(startIndex + 1));
	float4 row2 = tex2Dlod(_SkinningTex, getUV(startIndex + 2));
	return float4x4(row0, row1, row2, float4(0, 0, 0, 1));
}

VertexOutput vert(VertexInput input)
{
	VertexOutput output;
	UNITY_SETUP_INSTANCE_ID(input);
	float startPixelIndex = _StartPixelIndex;
	float4 index = input.uv1;
	float4 weight = input.uv2;
	float4x4 matrix1 = getMatrix(startPixelIndex + index.x * 3);
	float4x4 matrix2 = getMatrix(startPixelIndex + index.y * 3);
	float4x4 matrix3 = getMatrix(startPixelIndex + index.z * 3);
	float4x4 matrix4 = getMatrix(startPixelIndex + index.w * 3);
	float4 vertex = mul(matrix1, input.vertex) * weight.x;
	vertex = vertex + mul(matrix2, input.vertex) * weight.y;
	vertex = vertex + mul(matrix3, input.vertex) * weight.z;
	vertex = vertex + mul(matrix4, input.vertex) * weight.w;
	float4 normal = mul(matrix1, input.normal) * weight.x;
	normal = normal + mul(matrix2, input.normal) * weight.y;
	normal = normal + mul(matrix3, input.normal) * weight.z;
	normal = normal + mul(matrix4, input.normal) * weight.w;
	output.vertex = UnityObjectToClipPos(vertex);
	output.normal = UnityObjectToWorldNormal(normal);
	output.uv = input.uv;
	return output;
}
{% endhighlight %}

这个方案：<br>
1. 优点动画文件体积比原生动画文件还小，不占用CPU，GPU计算蒙皮；
2. 缺点GPU压力；

### 总结

处理大规模角色，同时不要求动画效果很精细，内存够用的情况，可以采用方案A。<br>
处理逻辑压力大、CPU消耗多并且渲染压力小的情况，可以选择方案B。<br>

For more details see <br>
[GPUAnimatorPlugin](https://github.com/nashnie/GPUAnimatorPlugin)<br>
