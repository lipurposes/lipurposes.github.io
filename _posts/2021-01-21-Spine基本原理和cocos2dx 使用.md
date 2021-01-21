---
layout: post
title:  "Spine基本原理 和 cocos2dx 使用"
---
# Spine基本原理 和 cocos2dx 使用
## 参考说明
- [spine导出json解析](https://zhuanlan.zhihu.com/p/118942376)
- [spine用户指南](http://zh.esotericsoftware.com/spine-user-guide)
- [spine运行库指南](http://zh.esotericsoftware.com/spine-applying-animations)
- cocos2dx 源码
## spine 基本原理
Spine 中通过将图片绑定到骨骼上，然后再控制骨骼实现动画。Spine 可以实现 网格，自由变形，蒙皮，皮肤，反向动力学等

世界变换可将骨骼的本地坐标变换到世界坐标。附加到一个骨骼的网格顶点被该骨骼的世界变换所变换。生成的顶点受该骨骼及其所有父骨骼影响。此机制是Spine骨架动画系统的核心。


## spine 层级结构
spine中动画是以skeleton组织的，一个spine工程可以包含多个skeleton,每个skeleton可导出一套文件供程序使用

spine中的skeleton（骨架） 层级结构是以 bone->slot->attachment的形式组织的

### bones
每个skeleton都有很多骨骼，骨骼以层级方式排列，每个骨骼受父骨骼影响，一直到根骨骼，根骨骼默认叫root,位于0，0点。

### slots
slot是骨骼的子，slot的层级结构决定附件的渲染顺序

### attactment
attactment是slot的子，是真正保存绘制对象的地方，一个slot可以有多个attactment孩子,但只能存在一个激活attactment对象（想一想，slot决定渲染顺序，假如一次有多个attactment激活，它们的渲染顺序谁决定？）
attactment 类型：
- region： 指的是一个贴了纹理的四边形。（是spine中最多，最常见的attactment）
- mesh： 是贴了纹理的mesh,蒙皮动画，自由变形需要它
- boundingbox:用于碰撞检测的多边形，比如说物理方面。
- path：一个三次方的曲线，通常被用来控制让骨头沿着一个path运动。
- point：一个点和一个旋转，通常用来发射子弹或者粒子。
- clipping：一个在绘制其他的挂件的时候用来剪裁的多边形区域。

导出的json表中的attachment 假如没有path 字段， 那么名字就是它的默认图片path

### skin皮肤
spine的组织结构是bone->slot->attactment,那skin的位置在哪？

spine的每个attactment都是slot的子，但它们每个又属于一个skin,假如spine动画中没有创建skin,它们属于自动生成的 default 名字的Skin.

每个skin都是一个map,里面保存了所有属于自己的attactment.假如一个slot有两个attactment子，一个属于skin1,一个属于skin2,假如使用skin1,那么属于skin1的attactment激活。（还记得一个slot同时只能有一个激活的attactment吗？）

假如使用一个skin,那么属于这个skin的位于各个slot的attactment相应激活，这样就实现了动态换装。

## spine animation
对于程序来说，animation就是一系列的timeline清单，每一个timeline都有一系列的关键帧清单，这些关键帧描述了一个骨头或者是一个插槽随着时间是如何运动的。下面是一份样本JSON数据：

```json
"animations": {
 "name": {
 "bones": { ... },
 "slots": { ... },
 "ik": { ... },
 "deform": { ... },
 "events": { ... },
 "draworder": { ... },
   },
   ...
}
```

## spine Cocos2dx代码结构
我们项目中dragonBones有 setBoneNode()方法。内部主要实现是
```lua
local slot = self.__armature:getCCSlot(boneName)
slot:setDisplayImage(node)
```
从上面例子中可以看出函数主要逻辑为获取slot, 设置slot的attactment.那么spine可以实现本方法吗？

cocos2dx 中 spine的入口类为 SkeletonAnimation，继承自SkeletonRenderer，
SkeletonAnimation 有一个控制动画对象AnimationState
```c
spAnimationState* _state;
```
还有一个
```c
spSkeleton* _skeleton;
```
spine 的实现机制一般是状态对象和数据对象，一般一个状态对象都有相应的数据对象，上面两个是主要的两个状态对象。

我们有游戏中经常调用的函数setAnimation
```lua
function NodeSpineAni:play(actionName)
    if actionName == nil then
        actionName = "default"
    end
    self:setAnimation(0, actionName, false)
    return self
end
```
这是我们lua代码中的用法，参数都是什么意思呢？ 特别是第一个参数0。我们看一下它的c++函数原型：
```c++
spTrackEntry* setAnimation (int trackIndex, const std::string& name, bool loop);
```
从原型中我们发现传的参数是一个trackIndex,返回值是一个 TrackEntry.Track ? spine播放动画是分层的，上代码：
```c++
void SkeletonAnimation::update (float deltaTime) {
	super::update(deltaTime);

	deltaTime *= _timeScale;
	spAnimationState_update(_state, deltaTime);
	spAnimationState_apply(_state, _skeleton);
	spSkeleton_updateWorldTransform(_skeleton);
}

void spAnimationState_update (spAnimationState* self, float delta) {
	int i, n;
	_spAnimationState* internal = SUB_CAST(_spAnimationState, self);
	delta *= self->timeScale;
	for (i = 0, n = self->tracksCount; i < n; i++) {
		float currentDelta;
		spTrackEntry* current = self->tracks[i];
		spTrackEntry* next;
		if (!current) continue;

		current->animationLast = current->nextAnimationLast;
		current->trackLast = current->nextTrackLast;

		currentDelta = delta * current->timeScale;

		if (current->delay > 0) {
			current->delay -= currentDelta;
			if (current->delay > 0) continue;
			currentDelta = -current->delay;
			current->delay = 0;
		}

		next = current->next;
		if (next) {
			/* When the next entry's delay is passed, change to the next entry, preserving leftover time. */
			float nextTime = current->trackLast - next->delay;
			if (nextTime >= 0) {
				next->delay = 0;
				next->trackTime = nextTime + delta * next->timeScale;
				current->trackTime += currentDelta;
				_spAnimationState_setCurrent(self, i, next, 1);
				while (next->mixingFrom) {
					next->mixTime += currentDelta;
					next = next->mixingFrom;
				}
				continue;
			}
		} else {
			/* Clear the track when there is no next entry, the track end time is reached, and there is no mixingFrom. */
			if (current->trackLast >= current->trackEnd && current->mixingFrom == 0) {
				self->tracks[i] = 0;
				_spEventQueue_end(internal->queue, current);
				_spAnimationState_disposeNext(self, current);
				continue;
			}
		}
		if (current->mixingFrom != 0 && _spAnimationState_updateMixingFrom(self, current, delta)) {
			/* End mixing from entries once all have completed. */
			spTrackEntry* from = current->mixingFrom;
			current->mixingFrom = 0;
			while (from != 0) {
				_spEventQueue_end(internal->queue, from);
				from = from->mixingFrom;
			}
		}

		current->trackTime += currentDelta;
	}

	_spEventQueue_drain(internal->queue);
}
```
注意看spAnimationState_update方法，可以看出AnimationState保存了一个TrackEntry列表，每次update时都会遍历这个列表里的TrackEntry,分别执行update.所以实际播放时是animateState 持有的所有Track(通道)分别执行本通道的动画，AnimateState可以通过track分层应用动画，而我们lua中一般只使用了通道0.

我们在lua中设置事件监听设置的是 animationState的事件监听，也就是对所有通道设置，我们也可以分别对每个track设置监听。

```c++
//设置所有track的回调
void setStartListener (const StartListener& listener);
void setInterruptListener (const InterruptListener& listener);
void setEndListener (const EndListener& listener);
void setDisposeListener (const DisposeListener& listener);
void setCompleteListener (const CompleteListener& listener);
void setEventListener (const EventListener& listener);

//设置单独track的回调
void setTrackStartListener (spTrackEntry* entry, const StartListener& listener);
void setTrackInterruptListener (spTrackEntry* entry, const InterruptListener& listener);
void setTrackEndListener (spTrackEntry* entry, const EndListener& listener);
void setTrackDisposeListener (spTrackEntry* entry, const DisposeListener& listener);
void setTrackCompleteListener (spTrackEntry* entry, const CompleteListener& listener);
void setTrackEventListener (spTrackEntry* entry, const EventListener& listener);
```
我们在调用setAnimation等设置动画播放的函数后可以通过它的返回值获得当前track,可以设置本track 的回调函数。

spine的animation事件，我们游戏中经常混淆的是结束事件和完成事件，spine文档上说一个动画结束事件和完成事件只能有一个发生，假如通道中的动画是通过调用立即结束等打断当前流程的函数，会执行结束事件，或者 通过 addAnimation()函数往本通道中增加了后续动画，导致本动画不是结束动画，那么也会调用结束事件。只有一个动画正常完整结束且后续没有动画才会执行结束事件。

下面来到我们前面说的主题，更换attactment。

```c++
spBone* findBone (const std::string& boneName) const;
/* Returns 0 if the slot was not found. */
spSlot* findSlot (const std::string& slotName) const;

/* Sets the skin used to look up attachments not found in the SkeletonData defaultSkin. Attachments from the new skin are
    * attached if the corresponding attachment from the old skin was attached. Returns false if the skin was not found.
    * @param skin May be empty string ("") for no skin.*/
bool setSkin (const std::string& skinName);
/** @param skin May be 0 for no skin.*/
bool setSkin (const char* skinName);

/* Returns 0 if the slot or attachment was not found. */
spAttachment* getAttachment (const std::string& slotName, const std::string& attachmentName) const;
/* Returns false if the slot or attachment was not found.
    * @param attachmentName May be empty string ("") for no attachment. */
bool setAttachment (const std::string& slotName, const std::string& attachmentName);
/* @param attachmentName May be 0 for no attachment. */
bool setAttachment (const std::string& slotName, const char* attachmentName);

bool SkeletonRenderer::setAttachment (const std::string& slotName, const std::string& attachmentName) {
	return spSkeleton_setAttachment(_skeleton, slotName.c_str(), attachmentName.empty() ? 0 : attachmentName.c_str()) ? true : false;
}
```

这是spine api 提供的一些函数，setAttachment函数可以在当前皮肤和default皮肤中查找attachment更换当前.下面是它查找attachment的函数

```c++
spAttachment* spSkeleton_getAttachmentForSlotIndex (const spSkeleton* self, int slotIndex, const char* attachmentName) {
	if (slotIndex == -1) return 0;
	if (self->skin) {
		spAttachment *attachment = spSkin_getAttachment(self->skin, slotIndex, attachmentName);
		if (attachment) return attachment;
	}
	if (self->data->defaultSkin) {
		spAttachment *attachment = spSkin_getAttachment(self->data->defaultSkin, slotIndex, attachmentName);
		if (attachment) return attachment;
	}
	return 0;
}
```

从中我们发现，它查找的是skin 中 slotIndex 为当前slot的attactment.我们还发现所有attactment都属于皮肤，没有创建也有一个默认皮肤。（它不找其它slot下的attactment,我想一般一个slot的attactment 一般不适用于其它slot吧）.

假如确实想换其它slot的attactment,可以这样

```c++
Skeleton skeleton = ...

// 按名称查找插槽。
Slot slot = skeleton.findSlot("slotName");
// 按名称从骨架皮肤或默认皮肤获取附件。
Attachment attachment = skeleton.getAttachment(slot.index, "attachmentName");
// 设置插槽的附件。
slot.attachment = attachment;

// 或者由骨架setAttachment方法来执行上述操作。
skeleton.setAttachment("slotName", "attachmentName");
```

按照这个思路，实现更换attachment方法，你会发现，更换不成功。why? 查看spine 导出的程序文件，你会发现，文件中animation动画信息中有具体的attachment名称及动画信息。更换attachment，执行动画时又使用回原来的attachment了。再仔细一想，动画中有明确的attachment名字，那更换皮肤后，不是换了attachment吗？ 那换肤怎么实现？

查看spine的 setSkin 源码， 请看这个函数：

```c++
void spSkeleton_setSkin (spSkeleton* self, spSkin* newSkin) {
	if (newSkin) {
		if (self->skin)
			spSkin_attachAll(newSkin, self, self->skin);
		else {
			/* No previous skin, attach setup pose attachments. */
			int i;
			for (i = 0; i < self->slotsCount; ++i) {
				spSlot* slot = self->slots[i];
				if (slot->data->attachmentName) {
					spAttachment* attachment = spSkin_getAttachment(newSkin, i, slot->data->attachmentName);
					if (attachment) spSlot_setAttachment(slot, attachment);
				}
			}
		}
	}
	CONST_CAST(spSkin*, self->skin) = newSkin;
}
```

发现更换的是slot下的同名attachment，也就是说不同皮肤下属于本slot的attachment 名字都相同，再查看下spine 编辑器。spine 编辑器要使用皮肤，首先要创建皮肤占位符，然后在皮肤占位符下添加attachment,它们分别属于不同皮肤，导出时会用皮肤占位符做为所有本占位符下的attachment的名字，分别存储到对应的skin下。

更换attachment 只能更换不同皮肤下同一slot的同名attachment,否则更换为其它attachment,一旦执行动画，就又回去了。

再次实现函数，更换slot的同名attachment，再执行，发现还是不成功，经测试，在执行动画时，虽然更换为其它皮肤的attachment, 但动画执行时会再次设置attachment 为本皮肤下的attachment ,又换回去了。

再次改变函数，更换attachment时，顺便对调不同skin下本slot的attachment.测试，成功。

```c++
// 用其它skin下的slot的同名attachent 替换当前皮肤下的slot attachment.
bool SkeletonAnimation::changeSkinAttachmentOfSlot (const char* skinName, const char* slotName) {
    spSkin * newSkin = spSkeletonData_findSkin(_skeleton->data, skinName);
    spSkin * skin =_skeleton->skin;
    if (newSkin && skin && newSkin != skin){
        int slotIndex = spSkeleton_findSlotIndex(_skeleton, slotName);
        if (slotIndex) {
            spSlot * slot = _skeleton->slots[slotIndex];
            if(slot->data->attachmentName){
                spAttachment* attachment = spSkin_getAttachment(newSkin, slotIndex, slot->data->attachmentName);
                if (attachment) {
                    spAttachment* attach = this->changeSkinSlotAttachment(skin, slotIndex, slot->data->attachmentName, attachment);
                    this->changeSkinSlotAttachment(newSkin, slotIndex, slot->data->attachmentName, attach);
                    spSlot_setAttachment(slot, attachment);
                    return true;
                }
            }
        }

    }
    return false;
}
//交换两个皮肤的attachment
bool SkeletonAnimation::swapSkinAttachmentOfSlot (const char* skinNameL, const char* skinNameR, const char* slotName, const char * attachName){
    spSkin * skinL = spSkeletonData_findSkin(_skeleton->data, skinNameL);
    spSkin * skinR = spSkeletonData_findSkin(_skeleton->data, skinNameR);
    if (skinL && skinR){
        int slotIndex = spSkeleton_findSlotIndex(_skeleton, slotName);
        if (slotIndex){
            spAttachment* attachmentL = spSkin_getAttachment(skinL, slotIndex, attachName);
            spAttachment* attachmentR = spSkin_getAttachment(skinR, slotIndex, attachName);
            if (attachmentL && attachmentR) {
                this->changeSkinSlotAttachment(skinL, slotIndex, attachName, attachmentR);
                this->changeSkinSlotAttachment(skinR, slotIndex, attachName, attachmentL);
                return true;
            }
        }
    }
    return false;
}
//替换皮肤下的同名attachment
spAttachment * SkeletonAnimation::changeSkinSlotAttachment(spSkin * self,int slotIndex, const char * name, spAttachment * attachment){
    spAttachment * retAttachment = nullptr;
    _Entry* newEntry = spSkin_createEntry(slotIndex, name, attachment);
    const _Entry* entry = SUB_CAST(_spSkin, self)->entries;
    _Entry * prime = nullptr;
    while (entry) {
        if (entry->slotIndex == slotIndex && strcmp(entry->name, name) == 0) {
            retAttachment = entry->attachment;
            if (prime){
                newEntry->next = entry->next;
                prime->next = newEntry;
            }else{
                newEntry->next = entry->next;
                SUB_CAST(_spSkin, self)->entries = newEntry;
            }
            FREE(entry->name);
            FREE(entry);
            break;
        }
        prime = const_cast<_Entry *>(entry);
        entry = entry->next;
    }
    return retAttachment;
}

```

那我们能不能自己创建一个attactment ,然后像我们游戏中使用dragonBones 那样更换呢！？ 查看源码，发现spine代码 是在spine 库的基础上包装实现的
```c++
typedef struct spAttachment {
	const char* const name;
	const spAttachmentType type;
	const void* const vtable;
	struct spAttachmentLoader* attachmentLoader;

#ifdef __cplusplus
	spAttachment() :
		name(0),
		type(SP_ATTACHMENT_REGION),
		vtable(0) {
	}
#endif
} spAttachment;
```
会发现创建attachment 是一件吃力不讨好的事，首先spAttachment没有继承node,不能像node那样给它增加子等，同时创建它要设置一系列属性，关联skin等，那我们为什么不让美术直接做出attachment,在代码中换attactment呢，这个复杂的功能就留待以后吧

spine 还有很多功能，欢迎更新指正

# DragonBones
DragonBones和Spine原理基本相同，这里就不做介绍了

# Cocos2dx Action系统
## 参考说明
- [cocos2dx 源码赏析之动作](https://www.jianshu.com/p/f9f550b1f0d5)
- cocos2dx 源码
## 类图结构
****![img](../../../res/Xnip2020-11-30_20-50-39.jpg)

## 基本分类

- 基本动作
	+ 各种改变属性的动作 
	+ 3d效果的动作
	+ 摄像机效果的动作
	+ 格子动作
	+ CallFunc回调函数
	+ Animate 帧动画动画
	+ ActionTween 改变 object 属性动作， 通过本动作可以实现和 各种分立的改变属性动作一样效果的动作

- Follow
- 包装动作

	在原有动作基础上封装产生新动作
    +  Speed 加速，减速动作
    +  Ease 改变动作时间曲线产生各种效果
    +  Clone 克隆产生一个动作
    +  Reverse 翻转一个动作

- 集合动作

    + Spawn 一起同时执行动作
    + Sequence 连续动作， 一个动作接一个动作执行

## 基本执行过程

所有的Action动作的管理都是由ActionManager类来管理的。ActionManager的初始化实在CCDirector类中：
```c++
bool Director::init(void)
{
    _actionManager = new (std::nothrow) ActionManager();
    _scheduler->scheduleUpdate(_actionManager, Scheduler::PRIORITY_SYSTEM, false);
}
```
这里，开启了个定时器，来不停的更新ActionManager的逻辑：
```c++
void ActionManager::update(float dt)
{
    for (tHashElement *elt = _targets; elt != nullptr; )
    {
        _currentTarget = elt;
        _currentTargetSalvaged = false;

        if (! _currentTarget->paused)
        {
            for (_currentTarget->actionIndex = 0; _currentTarget->actionIndex < _currentTarget->actions->num;
                _currentTarget->actionIndex++)
            {
                _currentTarget->currentAction = static_cast<Action*>(_currentTarget->actions->arr[_currentTarget->actionIndex]);
                if (_currentTarget->currentAction == nullptr)
                {
                    continue;
                }

                _currentTarget->currentActionSalvaged = false;
                _currentTarget->currentAction->step(dt);

                if (_currentTarget->currentActionSalvaged)
                {
                    _currentTarget->currentAction->release();
                } else
                if (_currentTarget->currentAction->isDone())
                {
                    _currentTarget->currentAction->stop();
                    Action *action = _currentTarget->currentAction;
                    _currentTarget->currentAction = nullptr;
                    removeAction(action);
                }

                _currentTarget->currentAction = nullptr;
            }
        }

        elt = (tHashElement*)(elt->hh.next);

        if (_currentTargetSalvaged && _currentTarget->actions->num == 0)
        {
            deleteHashElement(_currentTarget);
        }

        else if (_currentTarget->target->getReferenceCount() == 1)
        {
            deleteHashElement(_currentTarget);
        }
    }

    _currentTarget = nullptr;
}
```
这个函数遍历所有的Action，满足条件的会执行Action的step方法，由于，几乎所有的Action动作实例都是继承自ActionInstant和ActionInterval，所以，这里直接看这两个类的step方法的实现。
```c++
void ActionInstant::step(float /*dt*/)
{
    float updateDt = 1;
#if CC_ENABLE_SCRIPT_BINDING
    if (_scriptType == kScriptTypeJavascript)
    {
        if (ScriptEngineManager::sendActionEventToJS(this, kActionUpdate, (void *)&updateDt))
            return;
    }
#endif
    update(updateDt);
}

void ActionInterval::step(float dt)
{
    if (_firstTick)
    {
        _firstTick = false;
        _elapsed = 0;
    }
    else
    {
        _elapsed += dt;
    }
    
    
    float updateDt = MAX (0,MIN(1, _elapsed / _duration));

    if (sendUpdateEventToScript(updateDt, this)) return;
    
    this->update(updateDt);
}
```
在step方法中，会调用Action的update方法来执行具体的更新逻辑。这个update方法会交给Action的实例实现。

当要执行一个Action动作时，一般会调用Node的runAction方法：
```c++
Action * Node::runAction(Action* action)
{
    CCASSERT( action != nullptr, "Argument must be non-nil");
    _actionManager->addAction(action, this, !_running);
    return action;
}
```
这里会将实际要执行的Action实例添加到ActionManager中：
```c++
void ActionManager::addAction(Action *action, Node *target, bool paused)
{
    CCASSERT(action != nullptr, "action can't be nullptr!");
    CCASSERT(target != nullptr, "target can't be nullptr!");
    if(action == nullptr || target == nullptr)
        return;

    tHashElement *element = nullptr;
    Ref *tmp = target;
    HASH_FIND_PTR(_targets, &tmp, element);
    if (! element)
    {
        element = (tHashElement*)calloc(sizeof(*element), 1);
        element->paused = paused;
        target->retain();
        element->target = target;
        HASH_ADD_PTR(_targets, target, element);
    }

     actionAllocWithHashElement(element);
 
     CCASSERT(! ccArrayContainsObject(element->actions, action), "action already be added!");
     ccArrayAppendObject(element->actions, action);
 
     action->startWithTarget(target);
}
```

总结：
- 动作是由ActionManager管理的
- node 运行动作时调用ActionManager的addAction方法
- ActionManager 调用 action 的 startWithTarget 初始化动作运行
- ActionManager 每帧调用 所以action 的 step 方法
- action 的 step 方法将动作执行时间转化为0 - 1 之间值， 调用 action 的 update 方法

## Ease动作原理

查看源码大多数Ease动作都是通过宏批量生成的
```c++
EASE_TEMPLATE_IMPL(EaseExponentialIn, tweenfunc::expoEaseIn, EaseExponentialOut);
EASE_TEMPLATE_IMPL(EaseExponentialOut, tweenfunc::expoEaseOut, EaseExponentialIn);
EASE_TEMPLATE_IMPL(EaseExponentialInOut, tweenfunc::expoEaseInOut, EaseExponentialInOut);
//....
```
宏里面最重要的函数是这个
```c++
void CLASSNAME::update(float time) { \
    _inner->update(TWEEN_FUNC(time)); \
} \
```
ease 动作其余的函数都是调用的内部基本动作的函数，唯一的不同是update函数首先通过 TWEEN_FUNC 改变所传时间参数，内部动作再用改变的参数执行update

再来看一个tweenfunc 
```c++
float quadEaseInOut(float time)
{
    time = time*2;
    if (time < 1)
        return 0.5f * time * time;
    --time;
    return -0.5f * (time * (time - 2) - 1);
}
```
由此我们知道 ease 主要实现方式是 通过 tweenfunc 将 step 传入的 0-1 之间的时间曲线 改为各种其它曲线，然后再update ,实现各种效果

## 最后
还有很多3d 和 格子 动作，可以实现很多有趣效果， animate 动作可以播放帧动画

# 帧动画，骨骼动画，action 比较
程序action 主要通过改变节点各种属性等来实现动画，优点是不需要美术配合，但是不能所见即所得，要调一个完美的动画可能需要很多时间，不能实现炫酷的美术效果

骨骼动画和 帧动画 都是美术通过软件实现的，可以实现各种炫酷效果。 帧动画就是把效果动画导出为图片，通过每帧渲染不同图片实现动画。 骨骼动画 是 用美术图片 绑定mesh,mesh绑定骨骼，通过调整骨骼，mesh等的属性实现效果，最后导出供程序使用。

骨骼动画优点：使用图片少，效果不错， 缺点是效果复杂或比较多时，因为需要cpu 实时计算各种属性，cpu压力会很大，游戏会卡顿

帧动画 优点：效果最好，不需要cpu计算属性。缺点，资源大。











