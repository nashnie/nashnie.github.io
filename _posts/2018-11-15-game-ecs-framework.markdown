---
layout: post
title:  "游戏 ECS 框架设计"
date:   2018-11-15 10:00:00 +0800
categories: framework
---
## 游戏 ECS 框架设计

### ECS 简介
守望先锋团队在 GDC 上的演讲，分享了项目的框架设计，也就是 Entity–component–system (ECS)。<br>
ECS 和其他一些 MVC 框架的区别，我觉得最重要的一点是数据驱动。理解了这一点也就理解了 ECS。<br>
这是 ECS 维基百科的介绍:<br>
>Entity–component–system (ECS) is an architectural pattern that is mostly used in game development. 
>ECS follows the composition over inheritance principle that allows greater flexibility in defining entities where every object in a game's scene is an entity (e.g. enemies, bullets, vehicles, etc.).

ECS 基本实现框架不算复杂，Context 负责管理维护所有的 Entity 的生命周期，创建、添加、销毁等等，可以同时存在多个 Context，比如 InputContext、LogicContext，各种维护自己的数据。<br>
当然最上层有个全局的类复杂管理各个 Context。<br>
Context 下面就是 Entity，负责管理维护 DataComponent的生命周期，提供各种各样的接口，比如创建、添加、回收...然后就是 DataComponent，就是一个结构体，数据变化有事件发送。<br>
ECS 的优势之一就是各种各种的 Group，Group 如何实现的呢？Group 依赖 Matcher 机制来过滤，缓存符合条件的 Entity 的 List。<br>
Matcher 提供了几种规则供 Group 调用，allOfIndices、anyOfIndices、noneOfIndices，来判断这三个集合是否条件。<br>
最后就是 System，这个也很简洁，IInitiaizeSystem、IExecuteSystem、ICleanupSystem、ITearDownSystem，一个逻辑可能需要执行的四个阶段，分别注册到四个List里，然后在 MainGame 里驱动 SystemList。<br>


从守望先锋分享之后，我开始研究ECS，并在后续我们新项目搭建了基于 Entity-Component-System 的底层。<br>
#### ECS优势，<br>
1. 面向 Component(Data) 编程，数据驱动；
2. System 和 View 分离，方便多线程；
3. 方便移植；
4. 很好用、效率的底层接口比如各种 Group；（我们使用Entitas这个开源第三方的插件）
5. 更方便的 Debug；

#### 劣势嘛，<br>
1. 框架的学习成本；
2. 不能乱写带来的临时效率降低；
3. 还有，我想不到了…

同时 Unity 也在今年放出了他们的 ECS 第一个版本，<br>
我个人认为Unity ECS 比 Entitas 更先进，以下几点：<br>
1. Data和View完全脱离，View数据化。当然这个别人想做也做不了，毕竟Unity闭源引擎；<br>
2. 剥离之后，代码更简洁美观，而且方便做渲染层的优化…<br>
3. 还有一点，Unity ECS Data 部分是基于结构体的，这个性能很好，内存连续读取更效率；<br>
4. 另外一些小优化，Unity 接口封装的很多。但是因为改动实在太大，有些功能还没有完成，可惜；<br>

<br>
<br>

设计我们项目的框架，难点在如何和 ECS 完美结合。<br>
**services-logic-data-view**<br>
![](/images/ecs-framework1.png)<br> 

1. Services 提供第三方的数据和功能接口，比如 Input 和 Config、Audio 等。比如输入层，产生数据，获取主角的 Entity，改变 Entity 内 InputComponent，Entity 发送输入事件；<br>
2. System 层通过侦听数据变化事件，来处理相应逻辑，比如 Input 事件，然后执行移动、开发逻辑；<br>
3. View 层同样接受 Component 的数据变化来渲染，同时发送一些交互事件给 ECS System；<br>
4. Component（DataComponent）粒度越小越好，不要怕麻烦，这是ECS的精髓所在；<br>

ECS Entity 是各种数据 Component 的集合，所有的对象都是 Entity，玩家，怪物，花草树木，只要愿意都是数据的合体，也就是 Entity。<br>
比如 Hp，是一个 Component，可以挂在到任意的 Entity上，挂载成功之后，Entity 就天然拥有了Hp的所有逻辑和显示。非常方便和易于维护。<br>
同时一个 System 也可以是多个 component 组合在一起的逻辑。这就很方便的了，比如只有主角才关心输入，所有我们处理Input的System比如叫 InputProcessSystem，过滤 Input 和主角tag两个 Component 就可以得到我们想要的Entity，然后做输入逻辑。<br>
游戏内的每个对象都可以拥有 Id、input、hp、gun、dead、direction 等 component，这样一个游戏就可以是很多个System的合体了，比如登陆、匹配、输入、移动、换弹、开枪、瞄准、射击、死亡、复活等。<br>
基于这个设计，ECS底层可以实现很方便的数据过滤接口，比如过滤出各种符合我们条件的 Group，然后System只需要遍历感兴趣的Entity Group就可以了。这就是和传统的面向对象编程的区别之一，不是面对单个Entity。<br>

**Systems**<br>

![](/images/ecs-framework2.png)<br> 


**Services**<br>
![](/images/ecs-framework3.png)<br> 
...

>ECS 的设计就是为了管理复杂度，它提供的指导方案就是 Component 是纯数据组合，没有任何操作这个数据的方法；而 System 是纯方法组合，它自己没有内部状态。<br>
>它要么做成无副作用的纯函数，根据它所能见到的对象 Component 组合计算出某种结果；要么用来更新特定 Component 的状态。<br>
>System 之间也不需要相互调用（减少耦合），是由游戏世界（外部框架）来驱动若干 System 的。<br>
>如果满足了这些前提条件，每个 System 都可以独立开发，它只需要遍历给框架提供给它的组件集合，做出正确的处理，更新组件状态就够了。<br>
>编写 Gameplay 的人更像是在用胶水粘合这些 System ，他只要清楚每个 System 到底做了什么，操作本身对哪些 Component 造成了影响，正确的书写 System 的更新次序就可以了。<br>
>一个 System 对大多数 Component 是只读的，只对少量 Component 是会改写的，这个可以预先定义清楚，有了这个知识，一是容易管理复杂度，二是给并行处理留下了优化空间。<br>


### 注意事项

涉及多个 System 的 Entity，比如销毁某个 Entity 或者产生了某个状态，如果立即删除或者处理会对其他 Sytem 产生不可预知的后果，<br>
我们的处理办法是在 Game 驱动 System 函数末端执行统一的处理 System 比如 EntityDestroySystem，GameEventSystem 等。<br>
这样就尽量保证了多数 System 工作的时候，对大多数组件来说是无副作用的，而把严重副作用的行为集中在单点小心处理。<br>

建议对 Entity 以及 Component 做好对象池管理，增加这个引擎的厚度。<br>

在实际使用的过程中，我们还遇到了一些 Entity 生命周期管理的问题，比如销毁之后 View 层继续使用、ECS 层事件没有移除、System 执行周期等，一些小但是频繁的问题...<br>

### 结尾

ECS 是个不错的框架，设计思路也值得我们学习，对 ECS 的理解决定这个框架的上限。
