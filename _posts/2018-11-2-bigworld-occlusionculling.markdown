---
layout: post
title:  "基于大地图的遮挡剔除优化方案"
date:   2018-11-1 22:41:00 +0800
categories: none
---

### 基于大地图的遮挡剔除优化方案

CPU 上传数据到 GPU 渲染，要经过剔除（过滤），有两种剔除方案，平截头体剔除和遮挡剔除，<br>
一般这两种方案一起使用。平截头体剔除也就是相机剔除，不在相机平截头内的物件不渲染；遮挡剔除，顾名思义，就是被物体挡住的物体不渲染，<br>
这两种方案大大提升了渲染效率。

![](/images/bigworld-oc1.png)
这是没有遮挡剔除的效果；<br>

![](/images/bigworld-oc2.png)
这是平截头体剔除的效果；<br>

![](/images/bigworld-oc3.png)
这是加上遮挡剔除的效果；<br>

平接头剔除比较简单，Unity 也同时提供了剔除接口，方便大家做其他用途，比如做自己的地形草系统等等；<br>
常规的几种遮挡剔除思路如下，<br>
基于 CPU 的**software rasterization culling**，利用 Depth Buffer Rasterization，更准确以及可以方便多线程同时避开复杂的GPU部分，但是如果获取GPU深度图会有延迟问题，CPU 的剔除永远会延迟一帧，所以后续优化成，CPU生成深度图，物件利用AABB检测是否遮挡；<br>
基于 GPU 的检索，这部分我也没动手去尝试，优点是GPU越来越强大，可充分利用，缺点逻辑层不方便做基于 occlusion culling 的优化，比如降低tick等…当然可以通过八叉树等来做这个优化；<br>
为什么不用 Unity 原生的 occlusion culling？<br>
它占用太多的磁盘空间和内存，因为 Unity Umbra 在构建期间烘焙遮挡数据，Unity 需要在加载场景时将其从磁盘加载到 RAM。不支持动态加载、内存占用高、无法关闭动态物件 Culling，同时还有 CPU 消耗…这种方案不太适合对内存和包体要求较高的大地形方案，比如几公里*几公里的地图。
如何解决呢？<br>
离线的基于 **pvs** 的 occlusion culling，典型的空间换时间。<br>

1. 地图切割成 Tile（比如256*256），Tile 切割成 Portal（视口比如32*32）和 Cell（大物件（比如128*128）、中物件（比如64*64）、小物件（比如32*32）等）；<br>
2. 检测场景物件所属 Cell（区分大中小物件。基于 Cell 射线遍历检测），一个物件可以属于多个 Cell；<br>
3. 利用蒙特卡洛方法 Portal 随机一些起点，Cell 随机一些终点（根据大、中、小随机不同数量的点，数量越多越精确同时烘焙时间越长）；<br>
4. 双层遍历每个 Portal 和 Cell，射线检测是否阻挡，如果阻挡，判断阻挡物是否被在当前检测 Cell 内，如果是，Cell 可见；（注，当前设置两个高度检测，后续优化）<br>
5. 保存检测数据，运行时加载解析（当前数据结构为xml，方便调试用，后续提供二进制结构保存），根据玩家位置检测当前 Portal，然后遍历场景所有 Cell 内 Item，判断显示和隐藏；<br>

流式加载的数据以及简易的数据结构，解决了内存和磁盘暂用较大的问题，同时 CPU 消耗几乎可以忽略不计；同时方便做基于距离的加载，非常适合大地图方案。<br>
当然，缺点就是烘焙时间稍长...<br>

![](/images/bigworld-oc4.png)
这是全图的效果；<br>

![](/images/bigworld-oc5.png)
这是加上**遮挡剔除**的效果；<br>
红色箭头指向的是玩家的位置，地图其他区域已经被剔除掉了。<br>

**以下是代码实现**<br>
[PotentiallyVisibleSetPlugin](https://github.com/nashnie/PotentiallyVisibleSetPlugin)<br>

希望能帮到你。<br>

For more details see <br>
[software-occlusion-culling](https://software.intel.com/en-us/articles/software-occlusion-culling)<br>
[Potentially_visible_set](https://en.wikipedia.org/wiki/Potentially_visible_set)<br>
[OcclusionCulling](https://docs.unity3d.com/Manual/OcclusionCulling.html)<br>
