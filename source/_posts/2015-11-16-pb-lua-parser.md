---
title: protobuf协议文件格式化翻译
layout: post
toc: false
comments: true
date: 2015-11-16 17:23:24
tags:
- lua
---
公司开发有这么一个固定的流程，定下服务器和客户端的协议之后，需要把 protobuf 的 message 结构转变成固定格式的 lua 代码。
<!--more-->

protobuf 结构：

```
message Pair{
    required int32 key   = 1;
    required int64 value = 2;
}

message BJChange {
    required int32 seatPos  = 1;
    required int32 status   = 2; // 0 没参与 1输了 2 赢了 3平局
    required int64 winCount = 3; //>=0
    repeated Pair  pairList = 4;
}
```

翻译为：

```lua
LCPB_Pair = class("LCPB_Pair", LCProtoBase)
function LCPB_Pair:ctor(msgInput)
	local msg = msgInput or {}
	LCPB_Pair.super.ctor(self, protoPack, "Pair", msg)
	self.print = function()
		self:printItem("key")
		self:printItem("value")
	end
end

LCPB_BJChange = class("LCPB_BJChange", LCProtoBase)
function LCPB_BJChange:ctor(msgInput)
	local msg = msgInput or {}
	LCPB_BJChange.super.ctor(self, protoPack, "BJChange", msg)
	self.print = function()
		self:printItem("seatPos")
		self:printItem("status")
		self:printItem("winCount")
		for _, data in ipairs(self._msg.pairList or {}) do
			LCPB_Pair.new(data):printTitle():print()
		end
	end
end
```

规则大概是：

1. 内部类型（int32,...）直接输出，非内部类型（Pair）则调用该类型的输出来打印数据。
2. require 和 optional 输出一次，repeated 修饰的变量循环输出。

剩下的其他行规则更加明显。    
所以想着花些时间来自动化实现这个过程。

在[这本书](http://pan.baidu.com/s/1qWvgmPu)的帮助下，以及参考了一下 lua 的源代码实现了这个需求。

![自制编程语言](zzbcyy.png)

代码在[这里](https://github.com/sgamerw/pb2lua)，词法分析先拆分出每一个 token ，因为翻译规则简单，语法分析是 hard coding。后面再碰到类似需求，有这次的实践做支撑。

现在的版本还很初级，后续还可以

 1. 添加行号，遇到错误可以报出是第几行；
 2. 在变量名和保留字的解析处可以参考 lua 源代码，查表判断字符。
