---
layout: post
title:  "Network-Prediction"
date:   2020-11-9 22:20:00 +0800
categories: none
---
## Network Prediction
### Network Prediction

第一部分 Networked Simulation Model，网络模拟模型，包含三部分。

1. 接收输入，产生输出(The Simulation);
2. 模拟模型定义，包括用户类型定义、Tick 设置、用来真实执行模拟算法的类和函数比如插值等，也就是你想要网络模拟执行的方式；
3. 驱动（执行）定义，Finalize Frame、Produce Input、Debugging、Visual Logging 等等；
4. 四类 Buff:
   1. 输入: 通过一个客户端生成，(not the authority)。
   2. 同步数据: 需要同步的数据， 这个数据通过 Update 函数一直更新。
   3. Aux(???): 状态也是模拟系统的输入，但是不会根据 Update 函数一直更新。Changes to this state can be trapped/tracked/predicted。
   4. Debug: Replicated buffer from server->client with server-frame centered debug information。



To be continue...