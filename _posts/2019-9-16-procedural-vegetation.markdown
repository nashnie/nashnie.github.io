---
layout: post
title:  "大地形程序化植被生成方案"
date:   2019-9-16 13:50:00 +0800
categories: none
---
## 大地形程序化植被生成方案

手游品质越来越高，大地形的游戏越来越多，人力生成植被越来越满足不了策划日渐膨胀的内心，程序化的植被生成技术就运用到项目里来了，所以“策划推动技术发展”……<br>
这项技术具体要解决什么样的策划需求呢？<br>
上图，大地形环境下的大自然地貌。<br>
![](/images/procedural-vegetation1.png)<br>
再细分一下需求。<br>
不同地形，比如高度、温度、湿度、阳光甚至经纬度等决定了植被的生长，同时植被在斜坡上有比较规律但是又有一些随机的朝向，有不同密度，有不同的树龄等等。<br>
我们细化这些表现需求之后，得出具体要实现的细节。<br>
1. 如何定义区域；
2. 如何定义植被生长区域；
3. 如何决定植被密度；
4. 如何决定植被缩放(树龄)；
5. 如何决定植被旋转；

<br>
#### 生长区域
一个大的方案细化之后好像也没那么复杂了。<br>
我们学习了育碧 FarCry5 分享的植被技术方案，受益良多，感谢育碧 FarCry5 团队。<br>
育碧团队使用 Houdini 引擎结合游戏引擎的方案，导出一系列的坐标点也就是区域给游戏引擎使用。<br>
Houdini 使用美术同学给出的 Mask 贴图，比如Humidity、Flow、Slope、Curvature、Illumination 等等决定区域等，然后定义不同种类的植被以及植被在不同高度、不同光照下的生存能力 viability，这是概念非常启发我。<br>
有了 viability 之后，我们同时也知道大地图每一个点的属性，比如高度、湿度、温度等，我们可以很轻松就知道不同植被在这个点的 viability，然后存活能力强的植被就可以保留。<br>
感慨一句，我曾经被 UE 抽象的重要度概念 significance 惊讶到，这次育碧的存活能力 viability 同样。看上去复杂、模糊不清的问题一下子鲜明了。<br>
不过……<br>
Houdini 引擎和游戏引擎配合使用的工作流貌似对很多团队来说还是有点太硬核了。我们采用了折中方案，尤其是在我们的大地形还不是那么大的情况下(1kmx1km)。<br>
我们使用了游戏引擎内地形本身材质属性来决定植被的区域。比如草地、沙地、泥地等等。<br>
我们知道这些 layer 共同决定了地形 surface 的表现，那么我们可以通过获取地表点的某个属性占比属性来决定植被的生长，<br>
具体说比如椰子树喜欢在草地 layer 占比超过30%的区域生长，我们检测地表点 layer weight 符合条件就可以了，如果不同的植被都符合条件，加一个优先级就可以解决。<br>
我们还有第三个方案，这个方案就更简单了，在这里说一下，供大家参考。<br>
我们可以让美术同事出一张图，这个图使用不同的颜色来区分不同的区域。
如果用通道的话，就只能分四个区域，我们曾经的做法是使用颜色来区分不同的区域。这个颜色值和美术同事约定好就可以了，区域连接的部分记得模糊处理一下。加个误差值什么的都可以。<br>
我们的方案和 Houdini 方案劣势在哪里呢？<br>
在 layer 种类有限的情况下，不能表现出过于复杂的地表植被需求。layer 不够真实，比如陡璧，比如河边，比如山背等等。<br>
区域方案有了之后，我们接下来继续确定植被的密度、缩放、旋转。<br>
#### Rotation
默认情况下，植被的旋转都是朝上的，我们会根据坡度（坡面法线）倾斜植被，当然我们设置了一个最大的倾斜角度，防止角度太大，效果太假，同时树根在斜面上暴露出来的问题也可以缓解。<br>
树根问题的根本解决方案，我们是通过提高树木的Root点以及树木的碰撞来解决的。<br>
然后我们会随机自旋转植被，效果看起来自然一点。<br>
如果效果还想更逼真的话，比如靠近河边的草会倾向朝向河边以及风力的影响等等，这些细节我们目前还没有做。<br>
#### Scale
根据树龄插值缩放的值，树龄越大，树木越粗壮，然后我们会随机一部分树木的初始化树龄，然后迭代几次，保证有大树出现。这样保证了一丛树之间树龄分布的合理性，小树苗和大树都有。<br>
FarCry5 的细节更逼真，他们会根据植被所在区域的点的生存能力来影响树木的粗壮。<br>
区域边缘部分植被会比区域中心更弱小，这个也是个很好的细节，后续会做。<br>
#### Density
每一类植被都会有单独的密度设置。我们采用了正太分布算法来影响一丛树的分布。
我们观察了吃鸡类游戏的大地图，发现他们的树的部分，除了树丛还经常有孤零零的的树，这个我们是留有一定的百分比，正太分布只影响一部分树木，剩余的树只是简单的随机位置。<br>
其他的细节，比如在斜坡甚至悬崖上植被密度很低等等也是还没有做...

{% highlight C++ %}
float GetSeedMinDistance(const FProceduralFoliageInstance* Instance, const float NewInstanceAge, const int32 SimulationStep)
{
	const UFoliageType_InstancedStaticMesh* Type = Instance->Type;
	const int32 StepsLeft = Type->MaxAge - SimulationStep;
	const float InstanceMaxAge = Type->GetNextAge(Instance->Age, StepsLeft);
	const float NewInstanceMaxAge = Type->GetNextAge(NewInstanceAge, StepsLeft);

	const float InstanceMaxScale = Type->GetScaleForAge(InstanceMaxAge);
	const float NewInstanceMaxScale = Type->GetScaleForAge(NewInstanceMaxAge);

	const float InstanceMaxRadius = InstanceMaxScale * Type->GetMaxRadius();
	const float NewInstanceMaxRadius = NewInstanceMaxScale * Type->GetMaxRadius();

	return InstanceMaxRadius + NewInstanceMaxRadius;
}

/** Generates a random number with a normal distribution with mean=0 and variance = 1. Uses Box-Muller transformation http://mathworld.wolfram.com/Box-MullerTransformation.html */
float UProceduralFoliageTile::GetRandomGaussian()
{
	const float Rand1 = FMath::Max<float>(RandomStream.FRand(), SMALL_NUMBER);
	const float Rand2 = FMath::Max<float>(RandomStream.FRand(), SMALL_NUMBER);
	const float SqrtLn = FMath::Sqrt(-2.f * FMath::Loge(Rand1));
	const float Rand2TwoPi = Rand2 * 2.f * PI;
	const float Z1 = SqrtLn * FMath::Cos(Rand2TwoPi);
	return Z1;
}
{% endhighlight %}

#### Color
我们目前采用的是根据植被世界坐标随机颜色的做法。<br>
FarCry5 采用了根据树龄或者生存能力两项来决定颜色的做法。<br>
#### 最后放一张 Farcry5 的截图。大家感受一下。<br>
![](/images/procedural-vegetation2.png)<br>

For more details see <br>
[Procedural World Generation of Ubisoft’s Far Cry 5](https://vimeo.com/273986776)<br>