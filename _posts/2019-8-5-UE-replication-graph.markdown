---
layout: post
title:  "浅析 UE ReplicationGraph"
date:   2019-8-5 10:00:00 +0800
categories: none
---
## 浅析 UE ReplicationGraph

### 底层实现原理和流程

UE dedicated server 同步的代码还是很清晰的，NetDriver、NetSerialization 等等，看起来赏心悦目尤其是 ReplicationGraph，学到了一些思路。<br>
首先是建立连接，server 和 client 三次握手（handshakes）之后，一个 Client 对应一个 Connection，NetDriver 负责创建对应平台类型的 Socket 以及数据的收发 TickFlush 和 TickDispatcher。<br>
每一个 Replicator Actor 在每一个 Client Connection 上都有一个 ActorChannel 对应，每一个 Client Connection 同时还有一个额外的 VoiceChannel。<br>
ActorChannel 负责管理 ClientConnection 和对应 Client 的同步逻辑，也就是 ServerReplicator，负责 Actor 初始化以及销毁，当然还包括 Property 的对比以及同步等；<br>
在面对大规模的 Actor 同步的时候，针对每一个 Connection 的关联计算会消耗很多 CPU 时间，如何优化呢？ReplicationGraph 应运而生。<br>

### ReplicationGraph
	
ReplicatorGraph 和 UE 默认的同步机制是互斥的，只能同时使用其中一个机制。<br>
ReplicatorGraph 除了可以处理常规的各种同步需求外，还可以根据项目情况自定义各种各样的自定义同步机制，比如 Team 组队功能、Visible只同步主角平截头体内 Actor、降低重要度低的 Actor 的同步频率等等。<br>
首先Client和Server简历连接之后，ReplicatorGraph 针对每个 Client 创建一个 UNetReplicationGraphConnection，然后初始化 GraphNode。<br>
GraphNode 分两种，一种是 GlobalNode，针对所有的 ClientConnection，另一种是针对 Client 的 ForConnectionNode。<br>
ReplicationGraphConnection（ConnectionManager）有两个核心的函数，NotifyActorChannelAdded、NotifyActorChannelRemoved，维护 GraphNode 的 Actorlist。<br>

#### ReplicationGraphConnection（ConnectionManager）	
主要是以下几个因素累加值影响 Actor 的同步顺序。<br>
1. UpdateFrameGap，Actor 设置的同步间隔；
2. Distance Scaling，Actor 和 ViewTarget（Client Player Controller）的距离; 
3. Starvation Scaling，饥饿度，上一次同步和当前帧数差值; 
4. Pending dormancy scaling，是否进入睡眠状态; 
5. Game code priority，代码的特殊处理，比如强制同步等;
	
#### UReplicationGraphNode	
UReplicationGraphNode，所有 Node 的基类，主要有三个接口，添加和删除 NetActor，同时收集、管理 ServerReplicator 时符合条件的 ActorList。<br>
UReplicationGraphNode\NotifyRemoveNetworkActor\GatherActorListsForConnection<br>	

1. UReplicationGraphNode_ActorList，基类之一，包含 FStreamingLevelActorListCollection，可以维护多个 StreamingLevel 内的 Actor 的添加和删除等，满足大地图的需求；
2. UReplicationGraphNode_ActorListFrequencyBuckets，
这个类主要处理在 Non StreamingLevel 时大规模 Actors 的负载均衡（load balance），实现思路也很清晰简单，根据所有 Actors 的数量，分了几组 Buckets，每组设置 Actor 的上限，然后把所有的 Actors 随机分配到各个等待同步 ActorList 里。<br>
动态添加和删除 Actor 时调整 Buckets 数量，然后在 Replicator 的时候根据当前帧数依次选择一组 Buckets 进行同步，不在当前帧的 Buckets，可以旋转是否使用 FastShared 进行同步，也就是使用 NetCache 缓存的Bytes（Bunch）。<br>
3. UReplicationGraphNode_DynamicSpatialFrequency.
A node intended for dynamic (moving) actors where replication frequency is based on distance to the connection's view location.
4. UReplicationGraphNode_ConnectionDormanyNode
5. UReplicationGraphNode_DormancyNode
6. UReplicationGraphNode_GridCell
7. UReplicationGraphNode_GridSpatialization2D.
AddActor_Static\AddActor_Dynamic\AddActor_Dormancy\RemoveActor_Static…
根据每一个 actor 的 cullingdistance 设置 actor 自身辐射的格子范围，然后当 viewtarget 进入辐射的格子内时，就可以获取辐射到格子的所有 Actor；
UE针对DynamicActor做了个很精妙的优化，在同步之前（GatherActorList）的 PrepareForReplication 阶段，处理 DynamicActor 辐射的 格子变化，首先判断Actor前后变化的Rect是否相交，AABB 算法，如果完全不相交，那么全部更新 Rect 代表的格子，不过相交的话，复杂一点，判断相交的格子，增量更新；<br>
简单说下矩形的检测算法，水平方向上如果矩形1的右端比矩形2的左端还靠左或者矩形1的左端比矩形2还靠右，那么两个矩形是不可能重叠的，反之则是重叠的。<br>
垂直方向上的检测一样。重叠之后，我们判断重叠的区域部分，移除多余的格子里的 Actor，增加新增的覆盖格子，部分代码在下面。<br>

8. UReplicationGraphNode_AlwaysRelevant
9. UReplicationGraphNode_AlwaysRelevant_ForConnection.针对单个 Connection 处理，比如 PlayerController，判断当前的 PC 是否在 ActorList，添加 PC 进 ActorList，然后设置 cullingdistance 等于0，强制忽略距离因素。
10. UReplicationGraphNode_TearOff_ForConnection

{% highlight c++ %}
if (PreviousCellInfo.StartX < NewCellInfo.StartX)
{
	// We lost columns on the left side
	bDirty = true;

	for (int32 X = PreviousCellInfo.StartX; X < NewCellInfo.StartX; ++X)
	{
		auto& GridX = GetGridX(X);
		for (int32 Y = PreviousCellInfo.StartY; Y <= PreviousCellInfo.EndY; ++Y)
		{
			if (auto& Node = GetCell(GridX, Y))
			{
				Node->RemoveDynamicActor(ActorInfo);
			}
		}
	}
}
else if(PreviousCellInfo.StartX > NewCellInfo.StartX)
{
	// We added columns on the left side
	bDirty = true;

	for (int32 X = NewCellInfo.StartX; X < PreviousCellInfo.StartX; ++X)
	{
		auto& GridX = GetGridX(X);
		for (int32 Y = NewCellInfo.StartY; Y <= NewCellInfo.EndY; ++Y)
		{
			GetCellNode(GetCell(GridX,Y))->AddDynamicActor(ActorInfo);
		}
	}
}
{% endhighlight %}
	
#### FGlobalActorReplicationInfo	
设置 Uclass 类型的 FClassReplicationInfo，比如 DistanceScale、StarvationScale、CullDistance等。因为 UClass 类型对象很多，我们不可能针对每一个类型设置每一种。我们的做法是设置通用、常规的几种，比如 APawn、APlayerState 等，其他的遍历所有的 Uclass，统一设置； 

#### FConnectionReplicationActorInfo

#### EClassRepNodeMapping	
同样的，我们要设置每一种 UClass 的 EClassRepNodeMapping，也就是 Actor 归属的 Node 类型，根据这个 ClassRepNodeMapping 我们才能在 Actor 添加的时候判断把 Actor 放进那个 GraphNode。

#### 基本同步流程
ReplicateSingleActor->CallPreReplication->SetChannelActor、ActorInfo.Channel->ReplicateActor、PostReplicateActor->Loop Dependent actors。<br>
	

