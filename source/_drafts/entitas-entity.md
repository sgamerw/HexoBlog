---
title: Entitas 之 Entity
toc: false
comments: true
date: 2020-04-18 17:51:33
tags: Entitas
---
我们用 Entity 来表示程序中各种对象，可以理解为它是数据的容器，创建出一个 Entity 后，它是一个空容器，再添加相关数据（Component）后，它就能表示各种不同的对象了。

我们必须通过 `context.CreateEntity()` 来创建一个 Entity，这样创建出来的 Entity 才会被回收和复用。

``` c#
// Contexts.cs

/// Creates a new entity or gets a reusable entity from the
///             internal ObjectPool for entities.
public TEntity CreateEntity()
{
    TEntity entity;
    if (this._reusableEntities.Count > 0) // 如果回收池里有可用的 Entity，则出栈一个激活复用
    {
        entity = this._reusableEntities.Pop();
        // 更新 Entity 的唯一 ID
        entity.Reactivate(this._creationIndex++);
    }
    else // 如果回收池是空的，则新创建一个
    {
        entity = this._entityFactory();
        // 初始化 Entity 的唯一 ID 和存储 Component 的空间
        entity.Initialize(this._creationIndex++, this._totalComponents, this._componentPools, this._contextInfo, this._aercFactory((IEntity) entity));
    }
    this._entities.Add(entity); // context 把当前 Entity 加入到激活的 Entity 集合中
    entity.Retain((object) this);
    this._entitiesCache = (TEntity[]) null;
    entity.OnComponentAdded += this._cachedEntityChanged;  // context 会响应 entity 的各种事件， context 依赖这些事件来更新 group
    entity.OnComponentRemoved += this._cachedEntityChanged;
    entity.OnComponentReplaced += this._cachedComponentReplaced;
    entity.OnEntityReleased += this._cachedEntityReleased;
    entity.OnDestroyEntity += this._cachedDestroyEntity;
    if (this.OnEntityCreated != null)
    this.OnEntityCreated((IContext) this, (IEntity) entity); // 触发 OnEntityCreated 的监听 
    return entity;
}
```

`context.CreateEntity` 会调用 Entity 的以下方法进行初始化或者状态更新操作。
``` c#
// Entity.cs
public void Initialize(
    int creationIndex,
    int totalComponents,
    Stack<IComponent>[] componentPools,
    ContextInfo contextInfo = null,
    IAERC aerc = null)
{
    this.Reactivate(creationIndex);
    this._totalComponents = totalComponents; // 总 Component 个数
    this._components = new IComponent[totalComponents]; // 依据总 Component 个数，创建 Component 的缓存数组
    this._componentPools = componentPools; // Component 回收缓存池，remove 的 Component 将会放回这个池，这是从 context 中传入的参数，所以 context 中持有所有回收的 Component，这些 Component 可给任意 Entity 复用
    this._contextInfo = contextInfo ?? this.createDefaultContextInfo(); // 记录 Entity 所属 context 信息，主要用于完善异常信息的展示
    this._aerc = aerc ?? (IAERC) new SafeAERC((IEntity) this); // Automatic Entity Reference Counting (AERC) 实体自动引用计数
}

public void Reactivate(int creationIndex)
{
    this._creationIndex = creationIndex; // 更新 Entity 的唯一 ID，根据创建顺序加 1
    this._isEnabled = true; // 设置为激活状态
}
```

从这里可以看到，`_creationIndex` 可以唯一索引到一个 Entity，如果我们需要定位到某个 Entity，不要直接持有这个 Entity 对象，而是持有这个 Entity 的唯一 ID，可以考虑以下做法：

``` c#
// 创建一个 ID Component
[Game]
public class IdComponent : IComponent
{
    [PrimaryIndex]
    public int value;
}

// 在 context 创建 Entity 之前添加 OnEntityCreated 的监听
context.OnEntityCreated += (context1, entity) =>
{
    entity.AddId(entity.creationIndex);
};

// 接下来，我们可以通过 context.GetEntityWithId(int value) 找到对应的 Entity
```

另外，`Initialize` 函数中，Entity 根据 totalComponents 创建了一个 Component 数组，用来存放可以添加的 Component，这也是为什么我们需要把尽量相关的数据放到一个 context 下，举个例子：
> GameContext 下，1000 个 Entity 添加了 A、B、C 三个 Component，另外 1000 个 Entity 添加了 D、E、F 三个 Component，这种情况下，totalComponents 是 6，这 2000 个 Entity 都会 `this._components = new IComponent[6]`，如果我们把这 2000 个 Entity 分别放在两个 context1(A,B,C)， context2(D,E,F) 里，可以让数组填充率更高。

接下来看看添加 Component 时的处理过程：
``` c#
// Entity.cs

/// Adds a component at the specified index.
///             You can only have one component at an index.
///             Each component type must have its own constant index.
///             The prefered way is to use the
///             generated methods from the code generator.
public void AddComponent(int index, IComponent component)
{
    if (!this._isEnabled)
        throw new EntityIsNotEnabledException("Cannot add component '" + this._contextInfo.componentNames[index] + "' to " + (object) this + "!");
    if (this.HasComponent(index))
        throw new EntityAlreadyHasComponentException(index, "Cannot add component '" + this._contextInfo.componentNames[index] + "' to " + (object) this + "!", "You should check if an entity already has the component before adding it or use entity.ReplaceComponent().");
    this._components[index] = component;
    this._componentsCache = (IComponent[]) null;
    this._componentIndicesCache = (int[]) null;
    this._toStringCache = (string) null;
    if (this.OnComponentAdded == null)
        return;
    this.OnComponentAdded((IEntity) this, index, component); // 触发对应事件
}
```
一目了然，每个 Component 都有一个 index 对应，在生成代码时会自动创建这个索引，这里依此把 Component 放入数组对应的地方。

那么我们也能猜到 HasComponent[s] 这个方法的实现了。
``` c#
// Entity.cs

/// Determines whether this entity has a component
///             at the specified index.
public bool HasComponent(int index)
{
    return this._components[index] != null;
}

/// Determines whether this entity has components
///             at all the specified indices.
public bool HasComponents(int[] indices)
{
    for (int index = 0; index < indices.Length; ++index)
    {
        if (this._components[indices[index]] == null)
            return false;
    }
    return true;
}
```

ReplaceComponent 也是更新同一个数据，但是会分情况触发不同的事件。
``` c#
// Entity.cs

/// Replaces an existing component at the specified index
///             or adds it if it doesn't exist yet.
///             The prefered way is to use the
///             generated methods from the code generator.
public void ReplaceComponent(int index, IComponent component)
{
    if (!this._isEnabled)
        throw new EntityIsNotEnabledException("Cannot replace component '" + this._contextInfo.componentNames[index] + "' on " + (object) this + "!");
    if (this.HasComponent(index))
    {
        this.replaceComponent(index, component); // Component 存在，走 replace 流程
    }
    else
    {
        if (component == null)
            return;
        this.AddComponent(index, component); // Component 不存在，走 Add 流程
    }
}

private void replaceComponent(int index, IComponent replacement)
{
    IComponent component = this._components[index];
    if (replacement != component)
    {
        this._components[index] = replacement;
        this._componentsCache = (IComponent[]) null;
        if (replacement != null) // 如果新值不为 null，触发更新事件
        {
            if (this.OnComponentReplaced != null)
                this.OnComponentReplaced((IEntity) this, index, component, replacement);
        }
        else // 如果新值为 null，触发移除事件
        {
            this._componentIndicesCache = (int[]) null;
            this._toStringCache = (string) null;
            if (this.OnComponentRemoved != null)
                this.OnComponentRemoved((IEntity) this, index, component);
        }
        this.GetComponentPool(index).Push(component); // 回收 Component 
    }
    else // 即使更新的值和原来一样，还是会触发事件，被相应的 Collector 捕获
    {
        if (this.OnComponentReplaced == null)
            return;
        this.OnComponentReplaced((IEntity) this, index, component, replacement);
    }
}
```
这里我们其实可以用 ReplaceComponent 来代替 AddComponent, 开发过程中，可能出现创建 Entity 并添加 Component 的地方，改为了获取某个 Entity 并更新 Component，这时 AddComponent 可能因为已经存在这个 Component 而报错，举个例子：
``` c#
// 原代码
var e = context.CreateEntity();
e.AddA(A);
e.AddB(B);
e.AddC(C);

// 开发过程中更新
var e = context.GetEntityWithX(X);
/*
// 这里 e 可能已经有 A|B|C 组件，则会报错,
e.AddA(A); 
e.AddB(B);
e.AddC(C);
*/
e.ReplaceA(A); // 写 Replace 方法，更新或者添加 Component 都适用
e.ReplaceB(B);
e.ReplaceC(C);
```

replaceComponent 方法触发移除事件，那么我们也可以看下 RemoveComponent 的实现。
``` c#
// Entity.cs 

/// Removes a component at the specified index.
///             You can only remove a component at an index if it exists.
///             The prefered way is to use the
///             generated methods from the code generator.
public void RemoveComponent(int index)
{
    if (!this._isEnabled)
        throw new EntityIsNotEnabledException("Cannot remove component '" + this._contextInfo.componentNames[index] + "' from " + (object) this + "!");
    if (!this.HasComponent(index))
        throw new EntityDoesNotHaveComponentException(index, "Cannot remove component '" + this._contextInfo.componentNames[index] + "' from " + (object) this + "!", "You should check if an entity has the component before removing it.");
    this.replaceComponent(index, (IComponent) null);
}
```
这里也走到 replaceComponent 流程，所以我们了解到 `ReplaceX(null)` 和 `RemoveX()` 的异同，如果 XComponent 存在，那么这两个方法是一样的，如果 XComponent 不存在，`RemoveX()` 会抛出异常，`ReplaceX(null)` 则什么都不处理。

这些方法看下来，Entity 如何存储和更新 Component 就清楚了，也了解在对应的地方触发对应的事件。后续我们在 context 中再看对应这些事件做什么处理。