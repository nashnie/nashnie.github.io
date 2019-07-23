---
layout: post
title:  "如何优化网络多人游戏的移动同步？"
date:   2018-12-3 10:00:00 +0800
categories: gameplay
---
### 如何优化网络多人游戏的移动同步？

## 延迟是天敌
延迟是天敌，我们来打败它。<br>

## 优化策略
1. 预测和插值；
2. 客户端和服务器保证算法一致；
3. 缓存输入大法；
4. 逻辑和表现分离； 

### 预测
客户端和服务器保持一定的帧率运行各自的逻辑，客户端请求移动，发送操作到服务器端，服务器端收到客户端的移动操作开始移动，并同时通知所有的客户端开始移动；<br>
这是一个完整的移动过程(RTT)，中间客户端和服务器通信两次，网络状况较好的情况的下，网络延迟大概在50-100ms左右，如何优化这个延迟，保证手感和一致性？<br>
>What is round-trip time (rtt)?
>In telecommunications, the round-trip delay time (RTD) or round-trip time (RTT) is the length of time it takes for a signal to be sent plus the length of time it takes for an acknowledgement of that signal to be received. 
>This time delay includes the propagation times for the paths between the two communication endpoints.

客户端在UI交互之后立即发送操作给服务器端，此时，不应该等待服务器的反馈，应该立即移动，这个行为我们叫做预测，具体如何预测玩家（我们约定本地玩家为p1）的目标点呢？<br>
首先服务器收到客户端的开始移动请求，大概在半个RTT时间之后，所以我们预测HalfRTT时间之后，玩家的位置，<br>

{% highlight c# %}
p1TargetPosition = speed * direction * HalfRTT + p1Origin；
{% endhighlight %}

同时服务器收到玩家 p1 的移动请求时候，直接设置玩家 p1 的坐标到该位置，因为玩家在本地已经开始预测行走了。然后服务器广播给其他所有客户端 p1 的移动请求，又经历了HalfRtt的时间，其他玩家收到 p1 的移动请求之后，服务器此时的 p1 位置，应该是一个完整的RTT时间之后的位置。<br>
其他玩家（p2，p3...）此时应该立即移动到服务器当前的位置，<br>

{% highlight c# %}
p1CurrentPosition = speed * direction * RTT + p1Origin；
{% endhighlight %}

### 插值
预测成功之后，有两个地方需要插值，一个是P1客户端自己本地的预测之后，p1TargetPosition，以及 p2，p3 收到 p1 的移动请求的之后的预测 p1CurrentPosition。<br>
如何插值呢？<br>
最简单的插值算法，这是Unity官方的一个插值示例，<br>

{% highlight c# %}
// Transforms to act as start and end markers for the journey.
public Transform startMarker;
public Transform endMarker;
// Movement speed in units/sec.
public float speed = 1.0F;
// Time when the movement started.
private float startTime;
// Total distance between the markers.
private float journeyLength;

void Start()
{
	// Keep a note of the time the movement started.
	startTime = Time.time;
	// Calculate the journey length.
	journeyLength = Vector3.Distance(startMarker.position, endMarker.position);
}

// Follows the target position like with a spring
void Update()
{
	// Distance moved = time * speed.
	float distCovered = (Time.time - startTime) * speed;
	// Fraction of journey completed = current distance divided by total distance.
	float fracJourney = distCovered / journeyLength;
	// Set our position as a fraction of the distance between the markers.
	transform.position = Vector3.Lerp(startMarker.position, endMarker.position, fracJourney);
}
{% endhighlight %}

插值的本质 目标值等于起始值+改变值；<br>

{% highlight c# %}
target = start + change * ratio；
{% endhighlight %}

插值算法一般有二次、三次插值（Dead reckoning）等。我们大概讲一下三次插值的算法。<br>
实现这个插值需要4个坐标值。<br>

坐标1：开始位置（即本地当前位置）<br>

坐标2：坐标1经过一定时间后的位置（速度为当前速度）<br>

坐标4：最终位置（即网络协议发送的最新位置加上一定的延迟时间后的位置）<br>

坐标3：坐标4反向移动一定时间后的位置（速度为网络最新速度）<br>

插值坐标公式为：<br>

x = At3 + Bt2 + Ct + D<br>
y = Et3 + Ft3 + Gt + H<br>

其中<br>

A = x3 – 3x2 +3x1 – x0<br>
B = 3x2 – 6x1 + 3x0<br>
C = 3x1 – 3x0 D = x0<br>
E = y3 – 3y2 +3y1 – y0<br>
F = 3y2 – 6y1 + 3y0<br>
G = 3y1 – 3y0<br>
H = y0<br>

其中: x0 、x1、x2、x3 代表Pos1 ～Pos4 中的x 坐标, y0 、y1 、y2、y3 代表Pos1 ～Pos4 中的y 坐标。<br>

{% highlight c# %}
Pos1 = Posold;
Pos2 = Posold + Vold;
Pos3 = Posnew + Vnew ×t + ( 1 /2 ) ×anew ×t2 ;
Pos4 = Pos3 + [ Vnew + ( 1 /2) ×anew ×t. ] ;
{% endhighlight %}

通过上面公式计算出相应的( x, y) 值;<br>

**导航预测对于远程玩家非常有用，因为本地玩家 p1 其实不知道他们的具体移动方向等，所以很难察觉插值带来的区别。<br>
但是本地预测插值，玩家 P1 是能感受到延迟感的，也就是手感会不好。**<br>

所以针对本地玩家 p1 我们要采用不一样的策略来应对网络延迟，比如服务器只是转发玩家的位置给其他的客户端，p1 直接执行本地的操作进行移动，就像单机一样的体验，非常好，<br>
但是会导致外挂以及服务器和客户端因为RTT导致的位置不一致等，比如服务器判断打中了某人，而客户端其实是擦肩而过...这就很糟糕了，有没有更好的解放方案？<br>
那就是缓存重放机制。<br>

### 缓存重放机制
类似 MOBA 类游戏的帧同步，我们记录每一个行为（MoveInput）以及对应的时间戳（Frame），本地记录所有的行为，上传给服务器，等待服务器返回最近执行的行为，<br>
然后我们本地判断时间戳和收到的服务器行为的差，移除过去时行为，执行剩余的行为。（预测HalfRtt时间后服务器的执行的行为时间戳。）<br>
这个机制同时要求逻辑和表现层分离，表现层以稍慢的速度消化逻辑层产生的数据，等待服务器返回后才能对比帧数差。<br>
这个本质上是客户端和服务器端的不锁帧帧同步，同时以服务器为准。<br>

### Gameplay Prediction
1. 可以预测哪些行为？ 
+ 开枪或者释放火球等行为；
+ 血量、蓝量、移动速度等属性变化；
2. 需要解决哪些问题？
+ Undo，预测失败如何处理（rollback）；
+ Redo，如何避免重复执行预测行为；
+ Dependencies，如何处理预测行为之间的依赖关系；
+ Override，根据服务器数据覆盖本地预测状态等；
3. 如何解决？
我们使用 预测密钥（Prediction Key）。预测密钥是有 Local 生成，和 gameplay 的行为一起发送给服务器，服务器完成校验后，发送校验结果（失败或者成功）以及 Prediction Key 给客户端。<br>
服务器只需要把带 Key 的结果返回给上传 Key 的服务器即可，其他的客户端不需要这个 Key。<br>
客户端收到预测结果，0:CaughtUp,1:Reject，分别处理不同的行为。<br>

<br>
我们分三种类型的行为来讨论。<br>
1. **特效预测**<br>
+ CaughtUp
+ Reject，动画 Montage 等立即 Stop；

2. **动画预测**<br>
+ CaughtUp
+ Rejec

3. **属性预测**<br>
+ CaughtUp
+ Reject

### 还有哪些潜在的问题？

### 其他优化
比如一般技能释放都有起手动作，可以利用这个来消除一定的延迟，前端的起手动作假设有6帧，我们可以在3帧之后就向服务器请求发射子弹、火球等，利用这剩余的3帧，大概（0.033x3x1000）100ms，正好消耗掉一个完整的 RTT。<br>
比如子弹打中人，我们可以本地先射线检测是否命中，然后播放命中特效，等待服务器返回后再播伤害数字。<br>

**瞬间命中弹道非常快（AK等）**<br>
1. p1 开火通知 P2（一个延迟后创建子弹），P2 转发给 P3，P3 创建子弹；
2. P1 本地命中判定响应及时（P2 命中校验，反外挂）；
3. P1 命中预表现播特效声音等，但是伤害数值等 P2 下发，P1 和 P3 开始更新血量；

**投射类（手雷等）**<br>
1. P1 开火通知 P2 并开始前摇，P2 开始前摇并转发给 P3，P3 开始前摇，P1 前摇时间大于 P2，P2 前摇时间大于 P3；
2. P2 前摇结束，成功通知 P1 和 P3 创建子弹，然后开始命中判定，判定成功通知 P1 和 P3 开始表现伤害；

参考书籍<br>
守望先锋架构设计与网络同步（Overwatch Gameplay Architecture and Netcode）<br>
Dead Reckoning 技术在网络游戏中的应用<br>
网络多人游戏架构与编程<br>

