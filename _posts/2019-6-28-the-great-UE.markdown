---
layout: post
title:  "我也来夸一夸 Unreal Engine"
date:   2019-6-28 13:25:00 +0800
categories: none
---
## 我也来夸一夸 Unreal Engine

使用 UE 至今，还不足一年，深深的感到这个引擎的魅力，正好有还在使用 Unity 开发游戏的几个同事，问我关于使用 UE 的感受，我在微信和 QQ 聊了几句，感觉不足以表达我的全部感受，就有了这篇文章。<br>
我在2012就已经开始使用 Unity 开发页游了，后又在2014年使用 Unity 开发页游，新古龙群侠传、怪物军团、丛林大作战还有其他一些的没能上线的游戏比如代码Moba和代号吃鸡的游戏，2018年至今，使用UE。<br>
我认为 Unity 是个很厉害的引擎，本文也不是贬低 Unity 的意思，就是想从 UE 的角度给游戏开发者一些关于这两款引擎的对比。所以本文只讲我认为 UE 比 Unity 做的好的地方。<br>

### 动画
UE 的动画系统比 Unity 更加丰富和强大，Animation Aim Offset（瞄准）、Animation Blend Spaces、Animation Composite、Anim Montage 以及各种各样的动画 IK 节点。<br>
Anim Montage 实现了哪些功能呢？<br>
我们举个例子，我在用Unity做 Moba 的时候遇到这么个需求，类似安其拉的英雄有个技能，举起手来把魔法书打开前推，然后魔法书持续的发射光柱，一段时间后，安其拉把书收起来，<br>
这个过程有三个动作，起手举起书（once），持续阶段手和书保持前推（loop），结束阶段手落下把书收起来（once）。<br>
这在 Unity 里其实要控制起来就稍微复杂了，三段控制，UE 自带的 Anim Montage 完美的解决了这个问题，可以任意组合动画片段，然后设置各自的播放逻辑和事件。<br>
比如起跳和在空持续跳然后落地等等，很方便。<br>
因为 Unity 本身没有对 IK 很好的支持，IK 的例子就更多了，比如我们想控制 Mesh 里的一根骨骼，比如吉普车的方向盘的转向功能，在 Anim BP 里加一个对这根骨骼的 rotator 节点就可以了。<br>
策划同时又要求手部跟着方向盘一起动，这个在 UE 里也不难，动画同事做三个动画，左转90，中间、右转90，设置 Animation Blend Spaces，根据方向盘旋转的角度，混合就好了。<br>
还有比如脚贴地，左右手靠墙收枪等等。<br>
还有 Animation Sharing、Master Skeleton 等等...<br>

### 物理
使用下来我个人认为 UE 对 PhysX 的封装更易用更强大一些。
比如 Physic response 的设置，UE 使用了 Preset，Unity 的类似二维数组设置。
1. 新加一个 channel 设置默认 block，overlap 或者 ignore，preset 直接使用，不需要持续维护，<br>
2. 同时同类型（ObjectType）的物件可以使用不同的 preset，满足不同的需求。<br>

比如 UE 的 visibility channel，太强大了，比如玻璃，或者草丛这种，玻璃是可视的，但是草丛是不可视的。<br>
还有 UE physics asset 机制，可以针对头、胸、腿等设置不同的 collider type 和 physics material，以满足不一样的伤害需求，也可以满足比如载具天线刹车时细腻的物理效果表现。<br>
还有 UE 封装的更强大的 PhysX vehicle 的物理效果...

### 渲染
### Gameplay Framework
1. Actor、DefaultPawn、Pawn、Character，
Actor：场景中所有对象，类似 Unity gameobject；
Pawn：有行动能力，有动画和骨骼，接受 Input，可以被 Controller 控制的单位；
Character：人形，行走能力（CharacterMovement）和人相似的 pawn；
2. FloatingPawnMovement、CharacterMovement、NavMovement、ProjectileMovement，
ProjectileMovement::弹道移动控制
CharacterMovement::人形移动控制，比如走路、游泳、跳跃等
3. DamageType，
伤害系统，point、range...
4. GamdeMode
游戏规则玩法，得分、计时等等
5. GameState、PlayerState，
GameState::游戏数据
PlayerState::角色数据
6. PlayerController、PlayerCameraController、AIController
7. PlayerInput，
输入控制
8. SpringArmComponent、PlayerStart，
SpringArmComponent::弹簧臂
PlayerStart::出生点
...

看了上面这些引擎自带的 Class，你也应该大概明白 UE Gameplay Framework 有多完善了吧。这还只是一部分。<br>
### 地图和关卡
### Delicated Server
### 工具链
比如 UE ability、blueprint、非常丰富的 Profiler Debug 指令
### 粒子系统
### 其他细节
比如出生角色叠在一起怎么办，UE 自带 adjust location 算法，骨骼自带 muzzle 机制，比如弹簧臂，比如特效对象池，比如 GC，同时 UE 是开源的，这点简直太好了。

感谢阅读。<br>


