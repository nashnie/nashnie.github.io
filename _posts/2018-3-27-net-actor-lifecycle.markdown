---
layout: post
title:  "浅析 UE Actor 生命周期管理"
date:   2019-3-26 00:00:00 +0800
categories: gameplay
---

### 浅析 UE Actor 生命周期管理
UE UActor 继承自 UObject，是 UE 世界的非常基础和重要的单位，可以在 level 里摆放以及动态 spawn。<br>
我使用UE这段时间，发现有些同事对 Actor 的管理比如生命周期非常模糊，同时我在重构 Actor 管理这块的时候也有一些心得体会，所以有了这篇文章，希望能帮到大家。<br>
如果我们没有使用UE的 DS（Dedicated Server），我们需要自己处理 actor 的 spawn以及destroy，服务器 AOI 通知客户端添加一个 actor 比如玩家，一般服务器的协议里会至少带 ID、TypeID 以及玩家名字、状态等，这个 ID 是服务器保证唯一的，<br>
我们通过这个 TypeID 来读取 excel 配置表，获取一些静态数据比如玩家的最大血量、玩家移动速度，最重要的是获取玩家的资源蓝图路径，然后UE提供给我们了一个spawnActor的接口，这个接口可以动态的在当前场景（level）上创建 actor，一些看上去都非常完美...<br>
我们遇到了什么问题呢？这就要说到actor的生命周期了。<br>
玩家可以编程的 actor 的初始化步骤大概一下几步：（摘抄自UE4.21源码）<br>
1. PostLoad/PostActorCreated - Do any setup of the actor required for construction. PostLoad for serialized actors, PostActorCreated for spawned.  
2. AActor::OnConstruction - The construction of the actor, this is where Blueprint actors have their components created and blueprint variables are initialized
3. AActor::PreInitializeComponents - Called before InitializeComponent is called on the actor's components
4. UActorComponent::InitializeComponent - Each component in the actor's components array gets an initialize call (if bWantsInitializeComponent is true for that component)
5. AActor::PostInitializeComponents - Called after the actor's components have been initialized
6. AActor::BeginPlay - Called when the level is started

说一句，我们是组件式开发的gameplay。我们的业务组件都是在BeginPlay里启动的，添加侦听，初始化状态等等。<br>
但是问题来了，我们获取了actor的蓝图路径之后，World->spawnActor，spawn成功返回spawned actor，然后我们设置这个actor给我们的全局数据结构比如TMap储存起来，key是 actor NetID，value 是 actor和服务器通知的数据的组装结构体 PlayerNetData。<br>
所以在spawnActor 成功之前，我们是拿不到这个Tmap里的数据的。<br>
spawnActor的流程是这样的。（我就不翻译了，大家自己脑翻...）<br>

1. PostSpawnInitialize
2. PostActorCreated - called for spawned Actors after its creation, constructor like behavior should go here. PostActorCreated is mutually exclusive with PostLoad.
3. ExecuteConstruction:
OnConstruction - The construction of the Actor, this is where Blueprint Actors have their components created and blueprint variables are initialized
4. PostActorConstruction:
PreInitializeComponents - Called before InitializeComponent is called on the Actor's components
InitializeComponent - Helper function for the creation of each component defined on the Actor
PostInitializeComponents - Called after the Actor's components have been initialized
5. **OnActorSpawned broadcast on UWorld**
6. BeginPlay is called.
7. Return spawnedActor.

看到这里这里大家可能已经明白了，BeginPlay 是在 spawnedActor 返回之前就执行了的，如果我们在 BeginPlay 想获取全局TMap里存储的服务器玩家状态、名字等数据拿不到的...<br>
如何解决这个问题呢？<br>
World->SpawnActor 这个接口提供的参数非常有限，只有一个Params结构体，<br>
这个结构体可以传入 Name、Temp等非常可怜的几个参数。<br>
Name大家都知道是干嘛的，spawn出来actor的资源名字嘛，temp呢，这个也简单，就是个模板（母体），如果有这个temp的话，spawn出来的actor就等于是clone了一个一样的actor出来，听起来很好，但是问题在哪呢？<br>
因为我们是没有使用 UE DS 服务器设计，我们自己的玩家 actor 里面是有很多数据不需要 clone 的，比如主角自己状态等，<br>
第二个最重要的是是我们的 actor 本身是挂载了很多 component 的，这些个 component 默认 beginplay 就启动逻辑，我们不需要场景有两个主角把，一个temp主角一个clone的主角，然后temp的主角也有很多逻辑在执行，这个虽然加一些temp flag 避免，还是觉得麻烦了。<br>
我们还有一个 Name接口，我打算用这个参数大做文章，我在之前使用的Unity为调试方便，是把场景里的玩家等object的名字设置成服务器通知的ID的。
很自然的想起来的这个方案，把 ID 当成 name 设置给 actor，但是 UE 有个限制，所有的 actor 的名字必须是唯一的，如果添加了一个名字重复的 actor，会产生很诡异的现象，首先会主动销毁上一个actor，然后当前的actor也会莫名消失，反正都是我们不想看到的。<br>
UE提供了一个接口可以帮我生成一个全局唯一的名字供我们使用，（这更加验证了我们那名字做文章的思路是对的），这个名字规则很简单，levelName_actorBPTypeName_当前level存在的count++，然后我们把玩家ID拼接在后面，**levelName_actorBPTypeName_当前level存在的count++_NetID**，完美，<br>
设计好 actor 的名字规则之后，我们在就可以 World->SpawnActor 之前就提前在TMAP里存储好我们的数据，key 是玩家唯一资源名字，value 是 PlayerNetData。这样我们spawning过程中，任意的阶段都是可以获取这个 PlayerNetData 的包括 BeginPlay。<br>
不知道大家注意到没有，在 spawning 过程中，UE World 是发了一个事件出来的，在 BeginPlay 之前，OnActorSpawned broadcast on UWorld。<br>
看到这个我也是激动了一下，和你一样，但是...<br>
在测试以及阅读源码之后，发现，这个事件应该也确实是在 BeginPlay 之后发送的，倒是在 Return spawnedActor 之前，不过已经没有更多实际价值了...UE是个伟大的引擎，这是个低级的错误。希望 UE 早日修复这个 bug 或者修改文档...<br>
<br>
<br>
关于 actor 的销毁...<br>
说到销毁，如果不使用对象池的话，销毁很简单，直接调用 Destroy 就可以了。但是一般情况下，考虑到性能问题，对象池是必须要使用的。<br>
对象池的基本功能有哪些？

1. 对象基本存取以及预加载
2. 定时销毁
3. 最大上限
4. 内存 dump debug

具体实现如下，<br>
1. 根据 actor template 建立 key、value 的映射结构；然后实例化需要预加载的对象数量，存入 Array 中；
2. Tick 里检测 Actor 存入 pool 的时间和当前时间作对比，超过设置的时间比如5分钟，那么销毁该对象；
3. 设置最大上限，防止 pool 里对象过多带来的内存压力，在存入对象的时候检测 pool 里的资源数量，大于最大值，销毁该对象；
4. 内存 dump 信息，用于 debug actor pools 占用的内存；
<br>
<br>

终上所述，感谢阅读。<br>

[ActorLifecycle](https://docs.unrealengine.com/en-us/Programming/UnrealArchitecture/Actors/ActorLifecycle)<br>

