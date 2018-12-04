---
layout: post
title:  "physics character movement"
date:   2018-01-1 10:00:00 +0800
categories: gameplay
---
### physics character movement

## Movement position += motion
1.正常线性移动
## PushBack 
1.检测capsule character是否嵌入墙体内，是的话，find closest points 检测墙体比character薄的情况（子弹穿纸）
## ProbeGround  slope limiting and ground clamping
1.ground clamping，墙体拐角，normal插值
