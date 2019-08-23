---
layout: post
title:  "浅析 UE ReplicationGraph"
date:   2019-8-5 10:00:00 +0800
categories: none
---
## 浅析 UE dedicated server

### 底层实现原理和流程

UE dedicated server 同步的代码还是很清晰的，NetDriver、NetSerialization 等等，看起来赏心悦目尤其是 ReplicationGraph，学到了一些思路。<br>
首先是建立连接，server 和 client 三次握手（handshakes）之后，一个 Client 对应一个 Connection，NetDriver 负责创建对应平台类型的 Socket 以及数据的收发 TickFlush 和 TickDispatcher。<br>
每一个 Replicator Actor 在每一个 Client Connection 上都有一个 ActorChannel 对应，每一个 Client Connection 同时还有一个额外的 VoiceChannel。<br>
ActorChannel 负责管理 ClientConnection 和对应 Client 的同步逻辑，也就是 ServerReplicator，负责 Actor 初始化以及销毁，当然还包括 Property 的对比以及同步等；<br>
在面对大规模的 Actor 同步的时候，针对每一个 Connection 的关联计算会消耗很多 CPU 时间，如何优化呢？ReplicationGraph 应运而生。<br>

### ReplicationGraph
