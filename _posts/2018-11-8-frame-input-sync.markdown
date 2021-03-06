---
layout: post
title:  "帧同步手游客户端架构设计以及优化"
date:   2018-11-8 10:00:00 +0800
categories: architecture
---
## 帧同步手游客户端架构设计以及优化

今天分享一下之前实现过的 MOBA 帧同步的方案。<br>
先定义一下什么是帧同步，所有状态存储在服务器，然后下发给客户端的我们都不认为是帧同步方案，哪怕服务器是定时下发的，<br>
我们今天要讲的帧同步方案，特指客户端仅仅上传输入，服务器不做任何逻辑（防外挂不算）仅仅下发输入，这种方案我们叫做帧同步方案。<br>

那么帧同步到底有什么难点么？或者说 Moba类游戏的难点在哪里？同时帧同步为什么又是**MOBA类**游戏首先的同步方案呢？<br>

**难点：**
1. 对客户端架构要求高，<br>
	a. 逻辑表现必须分离，方便性能优化以及手感优化；<br>
	b. 保证数据一致性，必须底层做好所有数据计算的隔离，使用定点数等方案替代浮点数、随机数种子同步；<br>
2. 对客户端性能优化要求高，客户端承载了大量的逻辑计算的同时还在保证同屏多人的对战流畅；
3. 对客户端逻辑要求高，大量复杂的逻辑需要在客户端实现，比如复杂的一整套技能系统，伤害计算、Buff&Debuff等；
4. 对服务器反外挂要求高;

**优点：**
1. 优化做的好的话，容易实现很好的手感；
2. 客户端设计的好的话，可以快速迭代开发，几乎不用考虑网络层；

### 如何实现？<br>

### 架构设计
基本流程，输入 > 数据 > 帧 > 逻辑 > 表现。<br>
输入层产生帧输入数据，上传给服务器，服务器合并当前帧收到的所有输入数据下发给我所有客户端，客户端基于收到的输入数据模拟逻辑计算，然后输出逻辑数据给表现层，表现层渲染。<br>
值得注意的是，表现层的粒度是要大于逻辑层的，时间片的粒度更小，那么误差的范围可能更小，设定一个容差范围，通过表现层的平滑（影子跟随等）将误差逐步修正，允许表现层位置和逻辑层位置在一定范围内不一致，这是关键，而“这个范围”就是用来“包容”网络延迟的。

我们把所有对象（比如Player、Monster、Tower等）的 FixedUpdate 统一放在我们的一个管理器里面处理，严格控制逻辑的执行顺序，同时表现层Unit根据各自的Update获取数据进行渲染；<br>
同时我们写了基于 FixedUpdate 的帧事件系统来保证事件以及Timer等行为的一致性，最后我们把所有的浮点数计算都修改为定点数，浮点数只在最初转换一次，不要来回转换，最后在渲染层转换层浮点数就可以了。<br>
另外，我们的技能系统也是帧驱动的，同样是逻辑和表现分离；一个技能动作的触发行为，完全是数据化的，和 Animation Clip 本身没有关系，保证了触发的时机。<br>
后面我会详细分享一下我写的这个复杂强大的技能系统，^^<br>
其他需要注意事项，比如避免使用不确定性的接口，比如 Update、Time 等来驱动帧逻辑。<br>


### 性能优化
我主要讲一下帧同步类游戏特有的性能热点以及优化思路。<br>
如何优化同屏大量 unit 计算？
1. 场景管理部分，写了基于四叉树的 unit 管理，大大降低的计算量，比如一个小兵需要计算它的攻击对象的时候，只需要找到四叉树邻居进行判断就好了，不需要遍历所有;
2. 同时开发了不同的帧率控制不同的逻辑单位，比如，小兵的逻辑帧频20，玩家的30，还可以动态调整;
3. 大量小兵寻路优化，我采用的方案是缓存部分 A* 计算结果，因为大部分小兵的寻路是固定点的，因为路线固定;
4. 避免频繁的不必要的计算，比如浮点数定点数转换，尽量缓存运算结果;
5. 使用 GPU 来计算小兵的动作蒙皮，大幅降低了大量小兵同屏的CPU压力;
6. 草丛检测逻辑使用最简单的 BoxCollider 替换 MeshCollider，优化物理计算压力;
7. ...

### 网络部分
通常得建议是，网络延时，100毫秒以内，用户体验流畅，100-200毫秒，用户体验可以接收，200-400毫秒，勉强可玩，400毫秒以上，保证稳定和一致性的前提下，尽量能玩。200毫秒能覆盖很大部分的玩家了。<br>
根据这个基线来设计网络的策略以及优化。<br>
客户端不固定频率根据各自性能(update)和ping 值(截稿前^^固定三帧)上传数据。<br>
服务器每帧收集数据，固定频率发送数据给客户端。<br>
客户端根据收到数据来存储 inputbuffer，如果该数据包无某个 player 帧数据，则认为该player 网卡或者掉线等，如无缓存 input，用空白数据填充该 player 的 inputbuffer，伪锁运行逻辑。<br>
做法大概是这样，有个需要注意的点是包合并的问题，<br>

解析服务器下发的帧同步消息。<br>
{% highlight C# %}
void ParseFrameSyncMergeMsg()
{
	if (frameSyncMergeMsgList.Count > 0)
	{
		MergeRoomMsg mergeRoomMsg = frameSyncMergeMsgList[0];
		//收到的包小于当前帧才会解析
		//大于的话说明当前逻辑还没有执行到对应的帧，需要等逻辑执行完毕，逻辑部分会有对应的加速等逻辑
		if (mergeRoomMsg.frame <= FightManager.currentNetworkFrame)
		{
			playerNumList.Clear();
			frameSyncMergeMsgList.RemoveAt(0);
			parseFrameSyncMsgCount++;
			currentServerNetworkFrame = mergeRoomMsg.frame;

			for (int i = 0; i < mergeRoomMsg.msgs.Count; i++)
			{
				MsgRoomMsg msgRoomMsg = mergeRoomMsg.msgs[i];
				int playerNum = msgRoomMsg.data[0];
				if (playerNumList.IndexOf(playerNum) < 0)
				{
					playerNumList.Add(playerNum);
				}
				RemotePlayerController remotePlayerController;
				remotePlayerController = GetController(playerNum) as RemotePlayerController;
				if (remotePlayerController != null)
				{
					//解析收到的FrameInput ProcessInputBufferMessage
					remotePlayerController.OnMessageReceived(msgRoomMsg.data);
				}
			}

			//检查是否有player没有完整的FrameInput
			if (currentNetworkFrame == mergeRoomMsg.frame)
			{
				for (int i = 0; i < maxPlayerInMatch; i++)
				{
					int playerNum = i + 1;
					RemotePlayerController remotePlayerController;
					remotePlayerController = GetController(playerNum) as RemotePlayerController;
					remotePlayerController.AddLostMessage();
				}
			}

			//循环解析，网络状况下不好的情况解析多个收到的包
			ParseFrameSyncMergeMsg();
		}
	}
}
{% endhighlight %}

如何解析FrameInput？<br>


{% highlight C# %}
void ProcessInputBufferMessage(InputBufferMessage msg)
{
	if (msg.Data != null)
	{
		int offset = (int)(msg.CurrentFrame - this.currentFrame);
		FrameInput frame;
		int index;
		for (int i = 0; i < msg.Data.Length; ++i)
		{
			frame = msg.Data[i];
			//计算偏移，小于0说明是冗余包，不要处理，丢弃即可
			//framedelay（冗余）是客户端决定的，我们默认1-3帧，根据网络状态可以动态调整
			index = i + offset;

			if (index >= 0)
			{
				if (index >= this.inputBuffer.Count)
				{
					Dictionary<InputReferences, InputEvents> dict = SpawnerInputEventsList();
					this.inputBuffer.Add(dict);
				}
				
				foreach (InputReferences input in this.inputReferences)
				{
					InputEvents inputEvents = this.inputBuffer[index][input];
					if (input.inputType == InputType.LeftAxis)
					{
						inputEvents.axis = frame.horizontalAxis;
						inputEvents.axisRaw = frame.verticalAxis;
					}
					else
					{
						inputEvents.button = (frame.buttons & input.engineRelatedButton.ToNetworkButtonPress()) != 0;
						if (inputEvents.button)
						{
							inputEvents.axis = frame.horizontalRightAxis;
							inputEvents.axisRaw = frame.verticalRightAxis;
							inputEvents.percent = frame.percent;
						}                          
					}

					inputEvents.currentFrame = this.currentFrame + index;
				}
			}
		}
	}
}
{% endhighlight %}

如何处理FrameDelay？<br>

使用 JitterBuffer。<br>
缓存1-3帧Input，使抖动尽量均匀化。<br>
这个 JitterBuffer 可以在发送端也可以在接收端处理，发送端处理的话，设计比较清晰，但是发送的包量会大一些，可以通过一些策略优化比如相邻 FrameInput 如果没变优化成一个字节的 Flag 等。<br>
注意点就是一定要保证对应逻辑帧执行对应的输入，保证不同客户端逻辑一致性。<br>


### 总结<br>

帧同步手游客户端架构有三座大山，客户端一致性、性能瓶颈、手感（也就是网络同步优化），越过这三座大山就可以抵达彼岸。
