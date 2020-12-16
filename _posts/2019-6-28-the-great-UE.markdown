---
layout: post
title:  "我也来夸一夸 Unreal Engine"
date:   2019-6-28 13:25:00 +0800
categories: none
---
## 我也来夸一夸 Unreal Engine

使用 UE 至今，深深的感到这个引擎的魅力，正好有还在使用 Unity 开发游戏的几个同事，问我关于使用 UE 的感受，我在微信和 QQ 聊了几句，感觉不足以表达我的全部感受，就有了这篇文章。<br>
我在2012就已经开始使用 Unity 开发页游了，后又在2014年使用 Unity 开发手游，新古龙群侠传、怪物军团、丛林大作战还有其他一些没能上线的游戏比如代码 Moba 和代号吃鸡的游戏，2018年至今，使用 UE。<br>
我认为 Unity 是个很厉害的引擎，本文也不是贬低 Unity 的意思，就是想从 UE 的角度给游戏开发者一些关于这两款引擎的对比。所以本文只讲我认为 UE 比 Unity 做的好的地方。<br>

### 动画
UE 的动画系统比 Unity 更加丰富和强大，Animation Aim Offset（瞄准）、Animation Blend Spaces、Animation Composite、Anim Montage 以及各种各样的动画 IK 节点。<br>
Anim Montage 实现了哪些功能呢？<br>
我们举个例子，我在用 Unity 做 Moba 的时候遇到这么个需求，类似安其拉的英雄有个技能，举起手来把魔法书打开前推，然后魔法书持续的发射光柱，一段时间后，安其拉把书收起来，<br>
这个过程有三个动作，起手举起书（once），持续阶段手和书保持前推（loop），结束阶段手落下把书收起来（once）。<br>
这在 Unity 里其实要控制起来就稍微复杂了，三段控制，UE 自带的 Anim Montage 完美的解决了这个问题，可以任意组合动画片段，然后设置各自的播放逻辑和事件。<br>
比如起跳和在空持续跳然后落地等等，很方便。<br>
因为 Unity 本身没有对 IK 很好的支持，IK 的例子就更多了，比如我们想控制 Mesh 里的一根骨骼，比如吉普车的方向盘的转向功能，在 Anim BP 里加一个对这根骨骼的 rotator 节点就可以了。<br>
策划同时又要求手部跟着方向盘一起动，这个在 UE 里也不难，动画同事做三个动画，左转90，中间、右转90，设置 Animation Blend Spaces，根据方向盘旋转的角度，混合就好了。<br>
还有比如脚贴地，左右手靠墙收枪等等。<br>
还有 Animation Sharing、Master Skeleton 等等...<br>

### 物理
使用下来我个人认为 UE 对 PhysX 的封装更易用更强大一些。<br>
比如 physics response 的设置，UE 使用了 Preset 编辑同一类资源的 Overlap、Block 设定，而 Unity 的类似二维数组设置。<br>
1. 新加一个 channel 设置默认 block，overlap 或者 ignore，preset 直接使用，不需要持续维护，<br>
2. 同时同类型（ObjectType）的物件可以使用不同的 preset，满足不同的需求。<br>

比如 UE 的 visibility channel，太强大了，比如玻璃，或者草丛这种，玻璃是可视的，但是草丛是不可视的。<br>
还有 UE physics asset 机制（Physics Asset Editor），可以针对头、胸、腿等设置不同的 collider type 和 physics material，以满足不一样的伤害需求，也可以满足比如载具天线刹车时细腻的物理效果表现。<br>
Physics Asset Editor 也可以很方便的编辑 mesh 的 collider 等等，box、sphere、capsule、各种精度的凸包等等。<br>
还有 UE 封装的更强大的 PhysX vehicle 的物理效果...
还有 PhysX 为 UE 封装的 Blast 插件等...

### 渲染
1. 材质编辑器；
2. 强大的预览工具；

### Gameplay Framework
1. Actor、DefaultPawn、Pawn、Character，<br>
Actor:场景中所有对象，类似 Unity gameobject；<br>
Pawn:有行动能力，有动画和骨骼，接受 Input，可以被 Controller 控制的单位；<br>
Character:人形，行走能力（CharacterMovement）和人相似的 pawn；<br>
2. FloatingPawnMovement、CharacterMovement、NavMovement、ProjectileMovement，<br>
ProjectileMovement:弹道移动控制<br>
CharacterMovement:人形移动控制，比如走路、游泳、跳跃等<br>
3. DamageType，<br>
伤害系统，point、range...<br>
4. GamdeMode<br>
游戏规则玩法，得分、计时等等
5. GameState、PlayerState<br>
GameState:游戏数据<br>
PlayerState:角色数据<br>
6. PlayerController、PlayerCameraController、AIController<br>
7. PlayerInput<br>
输入控制
8. SpringArmComponent、PlayerStart<br>
SpringArmComponent:弹簧臂<br>
PlayerStart:出生点<br>
...

![ue-game-gramework.h](/images/ue-game-gramework.png)<br>

看了上面这些引擎自带的 Class，你也应该大概明白 UE Gameplay Framework 有多完善了吧。这还只是一部分。<br>
### 地图和关卡
1. 强大的大世界支持，以及 LOD（HLOD） 机制；
2. 程序化生成的植被系统，高斯分布、区域划分、QuadTree、成长系统以及等等，当然还有 Foliage Instance...，同时植被系统自带剔除和 LOD 机制；
3. Layer 和 Level 机制组合，更灵活；
4. World composition，根据距离加载的 Layer 和 Level，满足同一块区域不同细节的 LOD 需求；

### Delicated Server
UE 继承的服务器功能异常完善和稳定，这点是 unity 完全不能比的，虽然 unity 也推出了自家的 network，估计没人商用吧。国内的吃鸡手游就是用的 UE 的服务器。<br>

1. 客户端服务器共用一套代码；
2. 服务器为游戏逻辑服务器，单个服务器为核心，多个客户端连接；
3. 服务器拥有引擎的所有功能，大地形、物理、动画、AI 等等，可以很轻松的实现所有功能；
4. 房间机制很完善，和 gameplay 完美融合；
5. 简单的标签，Replicating、RPC，就可以实现数据同步，非常容易学习；
6. 针对 charactermovement 做了非常多的网络同步优化；
7. 同步频率和优先级等可以设置，针对不同的 actor 设置不同的刷新频率；
8. 防外挂机制；
9. 简单的几步设置就可以启动服务器；
10. 针对大规模 Actor 的同步优化，也就是 RepGraph，通过把 World 切分成 Grid，把 Actor 分组，利用 Distance、Starvation、GamdeCode 比如 ForceUpdateNet 等等计算 AccumulatedPriority；
11. float、vector 的压缩算法以及增量同步等，优化带宽；
12. RepGraph，满足各种各样的的 AOI 需求，比如每个 Actor 不同的 NetCullingDistance等，以及 AOI Grids 增量更新等；
13. 所有代码都是**开源**的...等等；

but，因为 UE Delicated Server 确实实现了大多的功能，性能瓶颈确实存在的，需要花力气优化。<br>

### 工具链
比如 UE ability、blueprint、非常丰富的 Profiler Debug 指令、DeviceProfileSelector、GConfig...<br>
1. UE GConfig 类似 unity 的 ScriptObject，自动序列化数据，UE 优势在统一的加载和管理（FConfigCacheIni）以及 运行时的保存机制
2. UE ability，非常强大的技能系统
3. UE blueprint，蓝图对策划以及动画、TA 同事非常友好，快速实现效果，快速检验，蓝图的试错成本远低于 C++
4. UE profiler debug，比如 UE cycle counter stat，方便监控任意代码块的数据，比如 Stat Unit、Stat Game、Stat GPU 等等，监控项目模块的性能
5. UE device profile 和 device selector，根据不同的设备优化定制不同的效果等
6. UE AI，behavitor tree、RVO 等

### 其他细节
UE 有非常非常多的技术细节，<br>
比如 UE 自带 adjust location 算法<br>
比如骨骼自带 muzzle 机制<br>
比如弹簧臂<br>
比如特效对象池<br>
比如多线程 GC<br>
比如多线程渲染<br>
比如材质编辑器（ShaderForge）<br>
比如 pvs、soft occlusion culling<br>
再比如可以很方便的设置 Tick 的频率、执行依赖、执行顺序等等<br>
最后，UE 是**开源**的，这点简直太好了。

感谢阅读。<br>


