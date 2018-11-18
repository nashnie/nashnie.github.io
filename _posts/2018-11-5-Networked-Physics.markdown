---
layout: post
title:  "网络物理同步方案"
date:   2018-11-05 10:00:00 +0800
categories: gameplay
---
## 网络物理同步

随着游戏品质要求越来越高，手游比如吃鸡等游戏上开始出现了物理交互的需求，如何实现是个难题，我给大家分享下我设计的一个物理同步方案。<br>
针对不同的游戏类型，使用的物理同步方案是不一样的，比如MOBA类游戏，大部分使用帧同步，那场景物理同步方案就比较清晰了，客户端计算，保证浮点数随机数一致性就好了，<br>
比如使用定点数修改一些开源的物理引擎，bullet、VelcroPhysics等等，MOBA类游戏的物理需求一般都不会太高，所以自己修改的难度不大，我主导的一个MOBA游戏就是这么做的。
状态同步就相对复杂了，物理计算部分可以在服务器，也可以在客户端。<br>
选择客户端的优点
1. 开发相对容易很多，服务器搭建物理刚体系统很麻烦；
2. 物理刚体系统消耗很大，放在客户端计算，不会影响服务器承载人数；

选择客户端的缺点
1. 外挂问题；
2. 多个客户端不一致问题等；

选择服务器计算物理，优缺点正好相反。<br>
**还有非常重要的一个优点**，是我们选择客户端来计算物理的原因，客户端计算物理的话，物理效果会好很多，p1(玩家自己)几乎不用考虑网络的问题，就可以做各种物理效果。

### 具体实现细节
类似于快照（snapshot），记录下每个单位的速度、角速度、坐标、旋转等，服务器转发给其他客户端进行模拟。<br>

{% highlight c# %}
public struct CubeState
{
	//是否激活
    public bool active;

    public ushort state;
	//坐标
    public int position_x;
    public int position_y;
    public int position_z;
	
	//旋转
    public uint rotation_largest;
    public uint rotation_a;
    public uint rotation_b;
    public uint rotation_c;

	//线速度
    public int linear_velocity_x;
    public int linear_velocity_y;
    public int linear_velocity_z;

	//角速度
    public int angular_velocity_x;
    public int angular_velocity_y;
    public int angular_velocity_z;
}
{% endhighlight %}

固定频率快照，然后上传。收到的客户端根据 Id 找到对应刚体，设置坐标和旋转同时根据线速度和角速度进行预测模拟。<br>

场景单位很多的话，包体大小是个大问题，所以数据结构的优化很重要<br>
1. 四元数旋转，只记录 xyz，w 通过 xyz 计算得出；
2. 定义数据的上下限以及精度，压缩数据；

{% highlight c# %}
public static int CompressFloat(float value)
{
	float range = (value - min) / (max - min);
	float maxBits = 1 << defaultRangeBits;
	range = range * (maxBits - 1);
	int intRange = (int)(range + 0.5f);

	return intRange;
}
{% endhighlight %}

min 和 max 越准确，越能准确表示压缩的小数。<br>
使用位数越多，越能准确表示压缩的小数，位数越少则压缩效果越好...<br>
 
3. 没有变化的单位不发；出于效果考虑，我们是最近五帧没有变化的话才不发；
4. 增量更新，针对单位本身增量更新，基线数据和Delta数据分开；
5. 降低同步频率；
6. 逻辑对象和表现对象分离，弹性插值跟随，以及单位根据速度和方向的预测；
7. 优化一切可以优化的字段，比如帧数使用两字节，最大65535，1-65535 循环使用；
8. ...

针对包体大小的优化其实有好多纠结点…我们也是用了**JitterBuffer**、**RingBuffer**等来优化。<br>
以上不管是帧同步还是状态同步(fixed time)都建议使用udp。<br> 
选择udp，而不是tcp，基于udp的实时性，高频协议那怕关闭tcp的negla算法，也无法和udp比，没有完美解决方案，udp的缺点在于大家知道的丢包以及同步的流量会加大(...)，所以要开发冗余包机制以及 ack  system…<br>

补充一点，<br>
网络物理同步如何选择 host ?<br>
如果一场游戏过程中，需要固定的几个客户端来完成本地计算，那么可以采用这个方案：<br>
服务器根据连接的客户端数量决定host上限，比如20%，因为我们游戏类型是玩家可以动态加入，防止游戏开始因为玩家数量过少导致host过少，而掉线导致游戏体验太差，所以开始时要快速添加host到一个最小值，比如5，然后按比例慢慢添加，同时可参考玩家硬件和网络情况。<br>
如果有类似载具的系统，车辆是场景的资源。玩家可以上车以及下车，那么谁来计算这个车辆的物理呢？<br>
1. 谁开车谁开计算物理；
2. 如果中途下车，那么在其他玩家上车之前，还是有上一个玩家来模拟物理；
3. 如果有场景九屏机制，那么该车离开了上一个玩家的范围也要继续模拟物理，不执行移除九屏处理，知道有其他玩家上车；
4. 车辆的模拟计算客户端切换问题；因为当前客户端模拟的状态和切换后的客户端模拟的状态理论上有偏差，如果处理这个偏差保证用户体验？


to be continue...<br>

