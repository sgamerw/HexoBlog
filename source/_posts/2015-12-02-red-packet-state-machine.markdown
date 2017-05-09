---
layout:     post
title:      "简单的状态机应用"
tags:
- lua
- 设计模式
---
公司项目需要实现一个红包功能。

有如下几个状态：
<!--more-->
|状态|没有红包|可以领|查看上局结果|自己已领|被别人领完|
|---|:-----:|:---:|:--------:|:-----:|:------:|
|没有红包|O|O|O|X|X|
|可以领|X|O|X|O|O|
|查看上局结果|O|O|O|X|O|
|自己已领|X|X|X|O|X|
|被别人领完|X|X|X|X|X|

`O` 是能转换，`X` 是不能转换。

思考可以用状态机来实现。

quick-cocos2d-x 中有一个强大的状态机，我没有用那个，只是很简单的实现一下状态转换的逻辑。

----------
在红包对象中加入了初始化和改变状态的两个函数。

```lua
LCRedPacketView = class("LCRedPacketView")
function LCRedPacketView:ctor()
	-- code ...
	self._state = nil
	-- code ...
end

function LCRedPacketView:initWithState(state)
	if state == nil then return end
	state.update(self)
	self._state = state
end

function LCRedPacketView:changeState(state)
	if state == nil then GError("no state input.") return end
	if self._state then
		if self._state == state then -- 如果是相同状态，则刷新数据
			self._state.update(self)
			return
		end
		self._state.onExit(self)
	end
	self._state = state
	self._state.onEnter(self)
	self._state.update(self)
end
```

该状态机的每一个状态都有 `onEnter`,`update`,`onExit`三个函数，并传入状态机对象自身。

```lua
local state = {}
function state.onEnter(obj)
	-- 处理进入该状态的相关初始化工作。
	-- 或者播放过渡动画。
end

function state.update(obj)
	-- 按照逻辑更新数据。
	-- 可以在内部根据逻辑跳转状态。
end

function state.onExit(obj)
	-- 退出该状态时清理相关数据。
	-- 或者播放退出动画，如弹框缩小，渐出。
end
```
类似上面 state 可以是一个类，但是我只需要提供逻辑处理过程，不用在状态类中存储状态相关信息，所以直接写成一张表，更简单。

### 在 `onEnter` 函数中
可以处理过渡动画，我在开红包的时候，封面会上移，就是从 「**可以领**」 到 「**自己已领**」 的状态时，这时可以在 `onEnter` 函数中开始动画，这里需要注意用一个变量把在 `onEnter` 和 `update` 中对封面的操作区分开，把行为区分开。

```lua
local state = {}
function state.onEnter()
	state.enter = true
	-- set cover action code
end

function state.update()
	if not state.enter then
		state.enter = nil
		-- set cover position
		-- set cover ...
	end
end
```

### 在 `update` 函数中
可以直接处理状态的跳转，这种比较直接，适合简单的逻辑。在 quick-cocos2d-x 中，它的状态机会要求传入一张状态转换的表，把状态之间的转换关系和转换时调用的函数配置进去。这样的配置灵活性更高。

### 在 `onExit` 函数中
可以做一些退出状态的清理工作，比如主角进入狂暴状态，攻击力提升，在退出该状态时，需要恢复之前的数值。

我这里的状态是根据服务器发来的数据进行更新，根据数据来判断现在应该进入哪个状态，然后直接 `changeState` ，如果已经是打开红包的状态，数据又有更新，但还是同样的状态，这时候在 `changeState` 中就已经有会有处理，相同状态直接 `update` 刷新数据，不是重新进入状态了。
