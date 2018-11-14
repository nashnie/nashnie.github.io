---
layout: post
title:  "MOBA类游戏技能系统设计"
date:   2018-11-13 10:00:00 +0800
categories: gameplay
---

## MOBA类游戏技能系统设计

### 技能数据结构设计
 
单个技能数据结构，分几个部分，<br>
技能基础帧行为，技能表现，技能事件，技能状态...<br>
好的技能数据结构设计是系统成功的一半。<br>

{% highlight c# %}
	//技能帧率
	int fps = 30;
	string description;

	//关键帧触发技能速度
	AnimSpeedKeyFrame[] animSpeedKeyFrame;

	bool fixedSpeed = true;
	float animationSpeed = 1;

	//帧过程 startUp-active-recovery
	int totalFrames = 15;
	int startUpFrames = 0;
	int activeFrames = 1;
	int recoveryFrames = 2;
	int currentFrame
{% endhighlight %}
	
{% highlight c# %}	
	//技能表现部分，动作以及融合
	AnimationClip animationClip;
	bool overrideBlendingIn = true;
	bool overrideBlendingOut = false;
	float blendingIn = 0;
	float blendingOut = 0;
	bool applyRootMotion = false;
{% endhighlight %}

{% highlight c# %}
	//输入
	ButtonPress[] buttonExecution;
	//帧事件-音效
	SoundEffect[] soundEffects;
	//帧事件-buff
	Buff[] buffs;
	//帧事件-位移
	Flash[] flashs;
	//帧事件-特效
	MoveParticleEffect[] particleEffects;
	//帧事件-攻击
	//activeFramesBegin;
	//activeFramesEnds;
	Hit[] hits;
	//帧事件-触发弹道
	Projectile[] projectiles;
	...
{% endhighlight %}

{% highlight c# %}	
	//帧状态
	MoveInfo[] previousMoves;
	//技能关联-下一个技能
	FrameLink[] frameLinks;
	//技能条件
	PlayerConditions opponentConditions;
	PlayerConditions selfConditions;

	bool cancelable 
	...
{% endhighlight %}

### 技能数据驱动逻辑
底层固定帧驱动技能帧按每个技能自身的频率移动，然后触发技能帧事件、状态等。<br>
如果当前帧大于技能最大帧，则技能结束。<br>

{% highlight c# %}	
//固定帧驱动
void DoFixedUpdate()
{
	//技能当前Tick递增
	skill.currentFix64Tick += skill.fixedDeltaTimeOfFps;
	skill.currentTick = (float)skill.currentFix64Tick;

	while (skill.currentTick > skill.currentFrame)
	{	
		//技能当前Frame递增
		skill.currentFrame++;
	}

	//设置技能当前所处阶段StartupFrames - ActiveFrames - RecoveryFrames
	if (skill.currentFrame <= skill.startUpFrames)
	{
		skill.currentFrameData = CurrentFrameData.StartupFrames;
	}
	else if (skill.currentFrame > skill.startUpFrames && skill.currentFrame <= skill.startUpFrames + skill.activeFrames)
	{
		skill.currentFrameData = CurrentFrameData.ActiveFrames;
	}
	else
	{
		skill.currentFrameData = CurrentFrameData.RecoveryFrames;
	}
}
{% endhighlight %}

触发弹道、音效、buff等事件。<br>
{% highlight c# %}	
//帧事件-触发弹道
for (int i = 0; i < skill.projectiles.Length; i++)
{
	Projectile projectile = skill.projectiles[i];
	projectile.castingFrame = (int)Mathf.Floor(projectile.castingFrame);
	projectile.casted = false;
}
//帧事件-音效
for (int i = 0; i < skill.soundEffects.Length; i++)
{
	SoundEffect soundEffect = skill.soundEffects[i];
	soundEffect.castingFrame = (int)Mathf.Floor(soundEffect.castingFrame);
	soundEffect.casted = false;
}
//帧事件-buff
for (int i = 0; i < skill.buffs.Length; i++)
{
	Buff buffEffect = skill.buffs[i];
	buffEffect.castingFrame = (int)Mathf.Floor(buffEffect.castingFrame);
	buffEffect.casted = false;
}
{% endhighlight %}

技能的碰撞检测稍微复杂一些，首先检测技能是否在攻击有效帧范围内，然后获取伤害碰撞盒，根据碰撞部位进行伤害计算。<br>

{% highlight c# %}	
for (int i = 0; i < skill.hits.Length; i++)
{
	Hit hit = skill.hits[i];
	HurtBox[] activeHurtBoxes = null;
	//是否在技能攻击有效帧范围内
	if (skill.currentFrame >= hit.activeFramesBegin &&
	    skill.currentFrame < hit.activeFramesBegin)
	{
		if (hit.hurtBoxes.Length > 0)
		{
			if (hit.disabled == false)
			{
				activeHurtBoxes = hit.hurtBoxes;
				//HurtBox hurtBox = hit.hurtBoxes[0];
				//BodyPart bodyPart = hurtBox.bodyPart;
				//process hit...			
				hit.disabled = true;
			}
		}
	}
}
{% endhighlight %}

补充一下，弹道触发之后，会生成弹道逻辑和显示两个实例，显示实例依赖逻辑产生的数据进行渲染，大家一定要把逻辑和显示分开，以方便执行一个预测、性能优化等，技能本身也是一样的。<br>

### 技能编辑器
一个好的数据结构成功的一半，一个好的编辑器是成功的另一半。^^<br>
技能编辑器一般有两种，基于时间轴以及基于节点，前者可读性好，后者扩展性好。<br>
我的技能编辑器是基于**时间轴**的，每一帧都可以编辑，添加事件，触发弹道、音效、buff、特效等，同时也可编辑的技能的状态，<br>
当然，最重要的是技能可以**预览**，这个是技能编辑器的灵魂也是难点。<br>
最后输出技能文件只包含数据没有任何显示层对象。或者输出两份，一份**纯数据**，一份带显示对象。<br>
