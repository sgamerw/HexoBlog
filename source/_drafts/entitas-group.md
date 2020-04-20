---
title: Entitas 之 Group
toc: false
comments: true
date: 2020-04-19 18:04:14
tags: Entitas
---
Group 是一组符合筛选条件的 Entity 集合。例如：

``` c#
// 所有添加了 PositionComponent 的 Entity 的 group
context.GetGroup(GameMatcher.Position)

// 所有同时添加了 PositionComponent 和 VisibleComponent 的 Entity 的 group
context.GetGroup(GameMatcher.AllOf(GameMatcher.Position, GameMatcher.Visible))
```

ECS 称为面向数据的设计思想，给实体（Entity）添加数据（Component）之后，业务逻辑可能需要处理的对象具有某些共性，即都具有某些的 Component，这样把 Entity 分组，对应的 system 只处理自己需要处理的 Entity。

``` c#
// Context.cs

/// Returns a group for the specified matcher.
///             Calling context.GetGroup(matcher) with the same matcher will always
///             return the same instance of the group.
public IGroup<TEntity> GetGroup(IMatcher<TEntity> matcher)
{
    IGroup<TEntity> group;
    if (!this._groups.TryGetValue(matcher, out group))
    {
        group = (IGroup<TEntity>) new Group<TEntity>(matcher);
        foreach (TEntity entity in this.GetEntities()) // 遍历 context 下所有 entity，把符合条件的 entity 添加到 group 中
            group.HandleEntitySilently(entity);
        this._groups.Add(matcher, group); // context 持有所有 group
        for (int index1 = 0; index1 < matcher.indices.Length; ++index1)
        {
            int index2 = matcher.indices[index1];
            if (this._groupsForIndex[index2] == null)
                this._groupsForIndex[index2] = new List<IGroup<TEntity>>();
            this._groupsForIndex[index2].Add(group);
        }
        if (this.OnGroupCreated != null)
            this.OnGroupCreated((IContext) this, (IGroup) group);
    }
    return group;
}
```