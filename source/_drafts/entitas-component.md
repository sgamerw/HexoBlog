---
title: Entitas 之 Component
toc: false
comments: true
date: 2020-04-18 22:12:32
tags: Entitas
---
Component 就是数据。

``` c#
/// Implement this interface if you want to create a component which
///             you can add to an entity.
///             Optionally, you can add these attributes:
///             [Unique]: the code generator will generate additional methods for
///             the context to ensure that only one entity with this component exists.
///             E.g. context.isAnimating = true or context.SetResources();
///             [MyContextName, MyOtherContextName]: You can make this component to be
///             available only in the specified contexts.
///             The code generator can generate these attributes for you.
///             More available Attributes can be found in Entitas.CodeGeneration.Attributes/Attributes.
public interface IComponent
{
}
```
我们自定义的 Component 实现这个接口就可以了，可以给 Component 添加一些标签，让 Component 具备唯一性或者指定所属 context 等其他属性。例如：
``` c#
[Game]
Public class IdComponent : IComponent 
{
    [PrimaryIndex] // 各种属性，后续看代码
    public int value;
}
```
按照 ECS 原则，我们不给 Component 添加方法。
