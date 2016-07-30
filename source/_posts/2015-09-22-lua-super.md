---
title: Lua面向对象的super关键字
layout: post
toc: false
comments: true
date: 2015-09-22 16:39:24
tags:
- lua
---
在Cocos2d-x-lua中，用以下方法实现了lua的继承机制来模拟面向对象。

```
function clone(object)
    local lookup_table = {}
    local function _copy(object)
        if type(object) ~= "table" then
            return object
        elseif lookup_table[object] then
            return lookup_table[object]
        end
        local new_table = {}
        lookup_table[object] = new_table
        for key, value in pairs(object) do
            new_table[_copy(key)] = _copy(value)
        end
        return setmetatable(new_table, getmetatable(object))
    end
    return _copy(object)
end

--Create an class.
function class(classname, super)
    local superType = type(super)
    local cls

    if superType ~= "function" and superType ~= "table" then
        superType = nil
        super = nil
    end

    if superType == "function" or (super and super.__ctype == 1) then
        -- inherited from native C++ Object
        cls = {}

        if superType == "table" then
            -- copy fields from super
            for k,v in pairs(super) do cls[k] = v end
            cls.__create = super.__create
            cls.super    = super
        else
            cls.__create = super
        end

        cls.ctor    = function() end
        cls.__cname = classname
        cls.__ctype = 1

        function cls.new(...)
            local instance = cls.__create(...)			
            -- copy fields from class to native object
            for k,v in pairs(cls) do instance[k] = v end
            instance.class = cls
            instance:ctor(...)
            return instance
        end

    else
        -- inherited from Lua Object
        if super then
            cls = clone(super)
            cls.super = super
        else
            cls = {ctor = function() end}
        end

        cls.__cname = classname
        cls.__ctype = 2 -- lua
        cls.__index = cls

        function cls.new(...)
            local instance = setmetatable({}, cls)
            instance.class = cls
            instance:ctor(...)
            return instance
        end
    end

    return cls
end
```

我们可以这样用：

**Base.lua**

```
local Base = class("Base")

function Base:ctor()
	print("Base:ctor()")
end

return Base
```

**Derived.lua**

```
lcoal Base = require("Base")
local Derived = class("Derived", Base)
function Derived:ctor()
	Derived.super.ctor(self) 	-- 方式1
	-- self.super.ctor(self)	-- 方式2
	-- Base.ctor(self)		-- 方式3
	self.super = 0
end

return Derived
```

我在派生类中见过以上3种方式调用父类的方法。

正确的使用方法应该是`方式1`。

### 方式2的弊端：
> 对于不知道class实现方式的用户来说，super这个变量应该也作为类的成员变量使用，所以如果修改了super的值，那么这种方式会调用不到父类方法。

### 方式3的弊端：
> 1. 我认为这里应该是为了使类的创建者关注的信息更少，不用去关心父类的名字。
> 2. 如果父类有变更，不用在类子类实现中逐个修改有调用父类方法的地方。
