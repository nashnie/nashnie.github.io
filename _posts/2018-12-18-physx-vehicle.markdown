---
layout: post
title:  "Physx Vehicle 笔记"
date:   2018-12-18 10:00:00 +0800
categories: none
---
## Physx Vehicle 笔记

本文只为记录使用以及学习 nvidia physx Vehicles 过程中的心得体会。<br>

### Tuning Guide

#### PxVehicleWheelData

mRadius:轮胎中心到轮胎外延的距离<br>
mWidth:轮胎的宽度，和轮胎 Mesh 匹配<br>
mMass:轮子和轮胎的总重量（20Kg-80Kg）<br>
mMOI:MOI = 0.5 * Mass * Radius * Radius，自动计算的，不需要设置<br>
mDampingRate:失去动力，自由旋转的轮子停止的速度（(0.25-2)）<br>
mMaxBrakeTorque:制动力矩，值越大越能更快地锁定车轮（1500左右）<br>
mMaxHandBrakeTorque:同上，手刹<br>
mMaxSteer:轮子最大转角（30-90度）<br>
这是方向盘完全锁定时车轮转向角（弧度）的值<br>
通常，对于四轮汽车，只有前轮响应转向，在这种情况下，后轮需要值0<br>
然而，更奇特的汽车可能希望前轮和后轮响应转向<br>
相当于30度到90度之间的弧度值似乎是一个很好的起点，但它实际上取决于被模拟的车辆<br>
较大的最大转向值将导致更严格的转弯，而较小的值将导致更宽的转弯<br>
但请注意，大转速下的大转向角可能会导致汽车失去牵引力并失控，就像真车一样<br>
避免这种情况的一个好方法是过滤在运行时传递给汽车的转向角，以便在更大的速度下产生更小的转向角。该策略将模拟在高速下实现大转向角的难度（在高速时车轮抵抗方向盘施加的转向力）<br>
mToeAngle: 脚趾角度可用于帮助汽车在转弯后伸直。 这是一个很好的数字，但最好留在0，除非需要进行详细的调整<br>

#### PxVehicleWheelsSimData
setSuspTravelDirection<br>
setSuspForceAppPointOffset<br>
setTireForceAppPointOffset<br>
setWheelCentreOffset<br>

#### PxVehicleSuspensionData
mSprungMass:
mSpringDamperRate:这描述了弹簧消散存储在弹簧中的能量的速率。可以通过调节这个值来控制载具颠簸感。<br>

4. PxVehicleAntiRollBar
5. PxVehicleTireData
6. PxVehicleEngineData
7. PxVehicleGearsData
8. PxVehicleAutoBoxData
9. PxVehicleClutchData
10. PxVehicleAckermannGeometryData
11. PxVehicleTireLoadFilterData
12. PxVehicleDifferential4WData
13. PxRigidDynamic

### Algorithm
...

For more details see <br>
[nvidia Vehicles](https://docs.nvidia.com/gameworks/content/gameworkslibrary/physx/guide/Manual/Vehicles.html)<br>


