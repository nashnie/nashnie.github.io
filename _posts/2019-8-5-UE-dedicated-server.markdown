---
layout: post
title:  "浅析 UE ReplicationGraph"
date:   2019-8-5 10:00:00 +0800
categories: none
---
## 浅析 UE dedicated server

### 底层实现原理和流程
UE dedicated server 同步的代码还是很清晰的，NetDriver、NetSerialization 等等，看起来赏心悦目尤其是最后 ReplicationGraph，学到了很多。<br>
首先是建立连接，server 和 client 三次握手（handshakes）之后，一个 Client 对应一个 Connection，NetDriver 负责创建对应平台类型的 Socket 以及数据的收发 TickFlush 和 TickDispatcher。<br>

### ReplicationGraph
