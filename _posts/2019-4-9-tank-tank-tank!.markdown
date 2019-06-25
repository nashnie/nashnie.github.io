---
layout: post
title:  "Tank!"
date:   2019-4-9 10:00:00 +0800
categories: none
---
## Tank!

坦克暂时告一段落，做坦克以及吉普等其他物理载具的过程中，遇到了很多问题，也积累了一些经验，记录下来，算是一个交代。<br>

### 坦克需要实现业务功能
1. 载人
2. 撞人、撞树
3. 开火，炮筒、炮台转动
4. 相机，Free Camera 
5. 遥感移动，非常复杂的移动逻辑...
6. 除了这些业务逻辑，我们要做的真实感的坦克，坦克底层的物理（PhysXVehicle）是最困难的部分
7. 网络同步
8. AI

### 载人
在车门附近配置 volumes，车门挂载socket等，触发上车逻辑，这个很常规，<br>
玩家上车之后，玩家的物理和载具的物理穿透的话，会出现很诡异的物理现象，坦克飞在天上，等等，我们处理的办法就是在上车之后关闭掉玩家和载具的物理交互，<br>
下车通过射线等检测玩家落脚点，这一步没有难点；<br>

### 撞人、撞树
撞人的难点是载具移动速度快的时候，车在撞人的一刹那有撞到墙的感觉，然后人死，整个过程是不流程的，<br>
我们的解决办法是在坦克身上挂载一个 Box，根据载具的速度动态控制 Box size，然后根据 Overlap 事件来判断撞人，这个 Box 是比坦克本省大一些的，这样的话就避免了我们刚才说的问题。<br>
撞树等障碍物的时候，有撞到墙的感觉是正常的，我们没有特殊处理，但是会动态的移除树木、篱笆等障碍物，保证坦克开起来比较顺畅。<br>
需要注意的是，我们在处理撞人的时候，使用 RingBuffer 记录坦克坐标，来检测坦克是否 AtRest，作为撞人的一个条件之一。<br>

### 开火，炮筒、炮台转动
这一步核心点是 IK 动画以及开火后坐力效果，IK 动画控制炮筒和炮台的旋转，<br>
注意如果想单纯的由 IK 动画来控制，记得移除（关闭）炮台和炮筒身上的Physics Asset，只保留底座的 Assset，因为 IK 和物理不能同时控制一个骨骼节点。开火后坐力实现比较简单，给坦克底座添加一个反方向的力即可。

### 相机，Free Camera 等
这一步难点是，我们的相机会根据炮台和炮筒来控制旋转和坐标，据我所知，是没办法直接使用 UE 的CameraController 的，我自己写了一个独立的 Camera Component，处理坦克的相机需求，<br>
核心点就是平滑输入，UI 层屏幕滑动输入是不平滑的，断断续续，我们首先插值玩家输入，然后根据炮筒和炮台的旋转（Yaw&Pitch）计算 Camera 的 位置和朝向，这里要注意的就是0-360边界的问题，我是用 Quaternion 计算差值来避免这个问题。

### 遥感移动，复杂的移动逻辑...
第一步，坦克炮筒本身分离，有两个控制维度，一个是镜头、一个AI根据底座的转向控制；<br>
玩家输入方向，镜头方向映射移动方向，判断坦克转向逻辑，计算坦克是车头还是车尾、顺时针还是逆时针、给油还是给刹车，然后对坦克履带施加力，希望坦克精准移动到目标方向；这是第二步<br>
最难是第三步，<br>
因为坦克本身惯性，比如坦克在高速前进时候玩家输入转向，策划希望如果方向大致一致的情况下，坦克不减速，惯性就很大、以及上坡坡度很大的时候滑坡这个是最难的等，坦克在坡上的时候，AI 很难精准控制目标方向，需要写很多逻辑驱动AI，然后暴露很多参数，调节参数；<br>
因为影响坦克的参数（包括物理参数）特别多，所以这个优化起来就很费劲；<br>
最后优化来优化去，加了误差之后比如10度，我们勉强可以达到策划的要求，在坡上也可以自由转向移动到我们的目标点。<br>

### 坦克底层的物理（PhysXVehicle）

我们先大致讲一下 vehicle 的底层结构
1. Vehicle Manager
2. Vehicle Data Struct
3. Vehicle Raw Input
4. Vehicle Wheels Animation

#### Vehicle Manager
VehicleManager，负责管理 Vehicle，管理 vechile 的 PreTick、Tick、PostTick，包括更新 Wheels 的 WheelState，比如 LocalPose、SteerAngle等等。
1. PxVehicleSuspensionRaycasts
2. PxVehicleUpdates

#### Vehicle Data Struct
1. VehicleEngineData MaxRPM、MOI、TorqueCurve...
2. VehicleGearboxData UpRatio、DownRatio、SwitchTime...
3. VehicleWheelData Radius、Width、DampingRate、Mass、MaxSteer...
4. VehicleSuspensionData MaxDroop、MaxCompression、SpringStrength...
5. VehicleCluthData??
6. VehicleWheelAnimData BoneName、LocalPose...

#### Vehicle Raw Input
Throttle、Brake、Steer、Clutch...

#### Vehicle Wheels Animation
Wheel 在初始化的时候通过Vehicle Manager 绑定，保证一一对应，然后每一帧根据轮胎的 Index 获取 Wheel LocalPose，在 AnimNode 里渲染。

#### 参数调整
我们在做坦克的过程中，遇到了很多奇怪的问题，比如坦克在撞击有些障碍物的时候会突然有奇怪的表现，坦克被反弹甚至击飞，后来我们发现这可能是UE的一个bug。<br>
如果障碍物设置为 BlockAll，但是障碍物的碰撞盒没有设置，载具撞击上障碍物的时候就会表现很奇怪，我们的解决办法是要求美术同事把所有需要设置 Block 的物件都一定要设置对应的 CollisionBox。<br>
我们还遇到坦克在撞击到时候会原地旋转，速度很快，经过无数次测试发现是因为我们修改 physx 数学库导致的...血的教训。<br>
关于坦克的爬坡能力，最重要的是坦克自身的 Physics Asset，一定要注意和地面的碰撞，以及轮胎的半径。<br>
然后就是轮胎力矩和坦克马力比例，保证爬坡时力矩的数值。<br>
第三点要注意档位的变换，在爬坡时调节 GearboxData 的 SwitchTime 尽量保证档位不要快速切换，让坦克在低档爬坡。<br>
其他的比如摩擦力等，但是不要一味的调大，过大的摩擦力会导致坦克启动的时候抖动，因为摩擦力太大了，启动困难。<br>
还有其他的小细节，比如坦克的 SpringStrength，保证坦克在坡面的抖动效果，轮胎履带通过设置 UV 滚动来实现，轮胎压过地面的痕迹通过 Decal 实现等。<br>

### 网络同步
这个比较常规，预测和插值，预测的时间根据坦克的速度以及 Ping 调节。<br>
炮台和炮筒的旋转和移动包一起，每0.1s同步一次，插值保证平滑。<br>

### AI
坦克在复杂地形上移动很复杂，关于AI部分，我后续更新...<br>

感谢阅读。<br>

更多细节<br>
[physx Vehicles](https://docs.nvidia.com/gameworks/content/gameworkslibrary/physx/guide/Manual/Vehicles.html)<br>