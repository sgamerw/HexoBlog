---
layout:     post
title:      "CCDelayTime和CCRepeatForever"
tags:
- cocos2d-x
---

最近碰到一个需求，一个精灵需要呈现出这样的效果：

> 图片跳动一下，然后左右摇晃...大概重复摇晃了10秒钟，再跳动一下。

整套动作拆开来都可以用引擎的 Action 实现，并且是重复动作，很容易想到用 `CCRepeatForever` + `CCSequence` 来做。

大概最简单的方式是：

```lua
CCJumpBy:create(0.2, ccp(0, 0), 20, 1)
// shake action
CCRotateTo:create(0.07, 20)
CCRotateTo:create(0.12, -15)
CCRotateTo:create(0.07, 10)
CCRotateTo:create(0.02, -5)
CCRotateTo:create(0.01, 0)
CCDelayTime:create(0.2)
// repeat shake action 10 times
CCRotateTo...
```
以上大约几十个 Action 包装在一起交给 `CCRepeatForever` + `CCSequence` 来执行。

这样的做法，写了很多的重复代码，而且非常冗长。

然后想到用2个 `CCRepeatForever` 动作，repeatForeverJump 重复 `CCJumpBy` 和 `CCDelayTime` 动作，repeatForeverShake 重复上面的 shake action。

在跳起的动作时，repeatForeverShake 执行 `CCDelayTime`，跳完了之后，repeatForeverJump 开始等待20轮 repeatForeverShake 的序列动作完成。

最后看起来代码像这样：

```lua
local actions = Transition.sequence{ -- 拼接成 CCSequence
	CCDelayTime:create(0.2),
	CCRotateTo:create(0.07, 20),
	CCRotateTo:create(0.12, -15),
	CCRotateTo:create(0.07, 10),
	CCRotateTo:create(0.02, -5),
	CCRotateTo:create(0.01, 0),
	CCDelayTime:create(0.2),
}
node:runAction(CCRepeatForever:create(actions))

local jumpAction = Transition.sequence{
	CCJumpBy:create(0.2, ccp(0, 0), 20, 1),
	CCDelayTime:create(0.49+0.69*20),
}
node:runAction(CCRepeatForever:create(jumpAction))
```

公司有的同学不太习惯一个 node 分两次调用 runAction，可能感觉会有冲突。公司的引擎是 2.x 版本的，看代码可以了解到，CCNode 在 runAction 的时候，是把自己交给了这个 action 去处理，由这个 action 的 update 函数去改变节点的属性。每个 action 都是独立的，只要不同时改变节点的相同属性就不会有冲突。所以上面的 `CCDelayTime` 和 `CCRepeatForever` 互相之间并不会冲突。
