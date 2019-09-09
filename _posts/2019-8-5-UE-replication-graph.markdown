---
layout: post
title:  "详解 UE ReplicationGraph 方案"
date:   2019-8-5 10:00:00 +0800
categories: none
---
## 详解 UE ReplicationGraph 方案

### 底层同步浅析

UE dedicated server 同步的代码还是很清晰的，NetDriver、NetSerialization 等等，看起来赏心悦目尤其是 ReplicationGraph，学到了一些思路。<br>
首先是建立连接，server 和 client 三次握手（handshakes）之后，一个 Client 对应一个 Connection，NetDriver 负责创建对应平台类型的 Socket 以及数据的收发 TickFlush 和 TickDispatcher。<br>
每一个 Replicator Actor 在每一个 Client Connection 上都有一个 ActorChannel 对应，每一个 Client Connection 同时还有一个额外的 VoiceChannel。<br>
ActorChannel 负责管理 ClientConnection 和对应 Client 的同步逻辑，也就是 ServerReplicator，负责 Actor 初始化以及销毁，当然还包括 Property 的对比以及同步等；<br>
每一个 Actor 会通过 Class.cpp 注册的 ClasssReps 初始化需要同步的 Property(clsReps 这里包含了所有需要同步的 Property)，然后根据 Property 内存记录 Property 的 Offset、Size 等也就是 Replayout 布局，这个 PropertyList Replayout 用于对比 Actor 数据是否发生变化以及序列化数据 Bunch。<br>
但是 在面对大规模的 Actor 同步的时候，针对每一个 Connection 的关联计算会消耗很多 CPU 时间，同时我们也会有一些同步的定制需求，如何处理呢？ReplicationGraph 应运而生。<br>

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
3. UReplicationGraphNode_DynamicSpatialFrequency
这个 GraphNode 和 ActorListFrequencyBuckets 很像，尽量保证同步效果的时候，降低需要同步的 Actor 数量，达到优化同步性能的目的。利用 Connection view location 计算距离裁切，然后在根据点积计算 Actor 和 Connection view target 的夹角。
根据夹角和距离在预测设置好的Buckets（Zones）里计算同步周期。
4. UReplicationGraphNode_ConnectionDormanyNode
5. UReplicationGraphNode_DormancyNode
6. UReplicationGraphNode_GridCell
7. UReplicationGraphNode_GridSpatialization2D.
AddActor_Static\AddActor_Dynamic\AddActor_Dormancy\RemoveActor_Static…
根据每一个 actor 的 cullingdistance 设置 actor 自身辐射的格子范围，然后当 viewtarget 进入辐射的格子内时，就可以获取辐射到格子的所有 Actor；
UE针对DynamicActor做了个很精妙的优化，在同步之前（GatherActorList）的 PrepareForReplication 阶段，处理 DynamicActor 辐射的 格子变化，首先判断Actor前后变化的Rect是否相交，AABB 算法，如果完全不相交，那么全部更新 Rect 代表的格子，不过相交的话，复杂一点，判断相交的格子，增量更新；<br>
简单说下矩形的检测算法，水平方向上如果矩形1的右端比矩形2的左端还靠左或者矩形1的左端比矩形2还靠右，那么两个矩形是不可能重叠的，反之则是重叠的。<br>
垂直方向上的检测一样。重叠之后，我们判断重叠的区域部分，移除多余的格子里的 Actor，把 Actor 添加到新增的格子列表里，我们根据 PreviousCellInfo 和 NewCellInfo 的 StartX、StartY、EndX、EndY 分四种情况，我们其中一种情况的代码在下面。<br>
8. UReplicationGraphNode_AlwaysRelevant
这个类处理需要同步给所有 Connection 的 Actor，需要注意的点就是如何过滤出这种类型的 Actor，一个入门是在我们后面会提到的EClassRepNodeMapping里设置，另一个入口是在该Node里添加动态入口，满足我们一些特殊需求。
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
设置 Uclass 类型的 FClassReplicationInfo，区别是这个设置是针对 Connection 的，不同的 Connection 可以不一样。

#### EClassRepNodeMapping	
同样的，我们要设置每一种 UClass 的 EClassRepNodeMapping，也就是 Actor 归属的 Node 类型，根据这个 ClassRepNodeMapping 我们才能在 Actor 添加的时候判断把 Actor 放进那个 GraphNode。

#### SingleActor 同步流程
ReplicateSingleActor 入口函数，执行 Actor 内部的 CallPreReplication，创建或者获取 ActorChannel，执行 Channel ReplicateActor，然后执行 Actor 内部的 PostReplicateActor，最后同步所有依赖该 Actor 的 Actor List。<br>
<br>
细节代码如下<br>

{% highlight c++ %}
int64 UReplicationGraph::ReplicateSingleActor(AActor* Actor, FConnectionReplicationActorInfo& ActorInfo, FGlobalActorReplicationInfo& GlobalActorInfo, FPerConnectionActorInfoMap& ConnectionActorInfoMap, UNetConnection* NetConnection, const uint32 FrameNum)
{
	RG_QUICK_SCOPE_CYCLE_COUNTER(NET_ReplicateActors_ReplicateSingleActor);

	if (RepGraphConditionalActorBreakpoint(Actor, NetConnection))
	{
		UE_LOG(LogReplicationGraph, Display, TEXT("UReplicationGraph::ReplicateSingleActor: %s. NetConnection: %s"), *Actor->GetName(), *NetConnection->Describe());
	}

	if (ActorInfo.Channel && ActorInfo.Channel->Closing)
	{
		// We are waiting for the client to ack this actor channel's close bunch.
		return 0;
	}

	ActorInfo.LastRepFrameNum = FrameNum;
	ActorInfo.NextReplicationFrameNum = FrameNum + ActorInfo.ReplicationPeriodFrame;

	UClass* const ActorClass = Actor->GetClass();

	/** Call PreReplication if necessary. */
	if (GlobalActorInfo.LastPreReplicationFrame != FrameNum)
	{
		RG_QUICK_SCOPE_CYCLE_COUNTER(NET_ReplicateActors_CallPreReplication);
		GlobalActorInfo.LastPreReplicationFrame = FrameNum;

		Actor->CallPreReplication(NetDriver);
	}

	const bool bWantsToGoDormant = GlobalActorInfo.bWantsToBeDormant;
	TActorRepListViewBase<FActorRepList*> DependentActorList(GlobalActorInfo.DependentActorList.RepList.GetReference());

	if (ActorInfo.Channel == nullptr)
	{
		// Create a new channel for this actor.
		INC_DWORD_STAT_BY( STAT_NetActorChannelsOpened, 1 );
		ActorInfo.Channel = (UActorChannel*)NetConnection->CreateChannelByName( NAME_Actor, EChannelCreateFlags::OpenedLocally );
		if ( !ActorInfo.Channel )
		{
			return 0;
		}

		CSVTracker.PostActorChannelCreated(ActorClass);

		//UE_LOG(LogReplicationGraph, Display, TEXT("Created Actor Channel:0x%x 0x%X0x%X, %d"), ActorInfo.Channel, Actor, NetConnection, FrameNum);
					
		// This will unfortunately cause a callback to this  UNetReplicationGraphConnection and will relook up the ActorInfoMap and set the channel that we already have set.
		// This is currently unavoidable because channels are created from different code paths (some outside of this loop)
		ActorInfo.Channel->SetChannelActor( Actor );
	}

	if (UNLIKELY(bWantsToGoDormant))
	{
		ActorInfo.Channel->StartBecomingDormant();
	}

	int64 BitsWritten = 0;
	const double StartingReplicateActorTimeSeconds = GReplicateActorTimeSeconds;
					
	if (UNLIKELY(ActorInfo.bTearOff))
	{
		// Replicate and immediately close in tear off case
		BitsWritten = ActorInfo.Channel->ReplicateActor();
		BitsWritten += ActorInfo.Channel->Close(EChannelCloseReason::TearOff);
	}
	else
	{
		// Just replicate normally
		BitsWritten = ActorInfo.Channel->ReplicateActor();
	}

	const double DeltaReplicateActorTimeSeconds = GReplicateActorTimeSeconds - StartingReplicateActorTimeSeconds;

	if (bTrackClassReplication)
	{
		if (BitsWritten > 0)
		{
			ChangeClassAccumulator.Increment(ActorClass);
		}
		else
		{
			NoChangeClassAccumulator.Increment(ActorClass);
		}
	}

	CSVTracker.PostReplicateActor(ActorClass, DeltaReplicateActorTimeSeconds, BitsWritten);

	// ----------------------------
	//	Dependent actors
	// ----------------------------
	if (DependentActorList.IsValid())
	{
		RG_QUICK_SCOPE_CYCLE_COUNTER(NET_ReplicateActors_DependentActors);

		const int32 CloseFrameNum = ActorInfo.ActorChannelCloseFrameNum;

		for (AActor* DependentActor : DependentActorList)
		{
			repCheck(DependentActor);

			FConnectionReplicationActorInfo& DependentActorConnectionInfo = ConnectionActorInfoMap.FindOrAdd(DependentActor);
			FGlobalActorReplicationInfo& DependentActorGlobalData = GlobalActorReplicationInfoMap.Get(DependentActor);

			UpdateActorChannelCloseFrameNum(DependentActor, DependentActorConnectionInfo, DependentActorGlobalData, FrameNum, NetConnection);

			// Dependent actor channel will stay open as long as the owning actor channel is open
			DependentActorConnectionInfo.ActorChannelCloseFrameNum = FMath::Max<uint32>(CloseFrameNum, DependentActorConnectionInfo.ActorChannelCloseFrameNum);

			if (!ReadyForNextReplication(DependentActorConnectionInfo, DependentActorGlobalData, FrameNum))
			{
				continue;
			}

			//UE_LOG(LogReplicationGraph, Display, TEXT("DependentActor %s %s. NextReplicationFrameNum: %d. FrameNum: %d. ForceNetUpdateFrame: %d. LastRepFrameNum: %d."), *DependentActor->GetPathName(), *NetConnection->GetName(), DependentActorConnectionInfo.NextReplicationFrameNum, FrameNum, DependentActorGlobalData.ForceNetUpdateFrame, DependentActorConnectionInfo.LastRepFrameNum);
			BitsWritten += ReplicateSingleActor(DependentActor, DependentActorConnectionInfo, DependentActorGlobalData, ConnectionActorInfoMap, NetConnection, FrameNum);
		}					
	}

	return BitsWritten;
}
{% endhighlight %}

### 总结
ReplicationGraph 旨在解决大规模 Actors 同步时服务器 CPU 性能压力以及流量压力，ReplicationGraph 作为 UE 的插件，设计思路特别清晰，代码非常清晰易懂，比如 增量 Grid 、角色 FOV 视野、负载平衡等等，还可以很方便的定制项目里的各种自定义 Group 需求，强烈推荐使用。 
	

