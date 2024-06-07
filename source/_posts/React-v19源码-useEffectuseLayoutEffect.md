---
title: React@v19源码---useEffectuseLayoutEffect
date: 2024-06-07 19:22:31
toc: true
tags: [React]
---

# useEffect

- [useEffect](#useeffect)
  - [mountEffect / updateEffect](#mounteffect--updateeffect)
    - [mountEffectImpl / updateEffectImpl](#mounteffectimpl--updateeffectimpl)
      - [总结](#总结)
      - [pushEffect](#pusheffect)
        - [总结](#总结-1)
    - [useEffect 回调什么时候触发呢？](#useeffect-回调什么时候触发呢)
    - [flushPassiveEffects](#flushpassiveeffects)
      - [flushPassiveEffectsImpl](#flushpassiveeffectsimpl)
      - [commitHookEffectListUnmount](#commithookeffectlistunmount)
      - [commitPassiveMountEffects](#commitpassivemounteffects)
        - [总结](#总结-2)
- [useLayoutEffect](#uselayouteffect)
  - [mountEffectImpl / updateEffectImpl](#mounteffectimpl--updateeffectimpl-1)
    - [触发时机在哪呢](#触发时机在哪呢)
      - [总结](#总结-3)

## mountEffect / updateEffect

```js
// PassiveStaticEffect 静态副作用，用于标记组件只在挂载和卸载时运行
mountEffectImpl(PassiveEffect | PassiveStaticEffect, HookPassive, create, deps);

updateEffectImpl(PassiveEffect, HookPassive, create, deps);
```

除了 `mount` 的时候多了一个静态标记以外没什么别的区别

### mountEffectImpl / updateEffectImpl

```js
function mountEffectImpl(
  fiberFlags: Flags,
  hookFlags: HookFlags,
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null
): void {
  // 创建hook对象在上文已经讲过
  const hook = mountWorkInProgressHook();
  // useEffect的第二个参数
  const nextDeps = deps === undefined ? null : deps;
  // 给当前组件的fiber打上副作用标记
  currentlyRenderingFiber.flags |= fiberFlags;

  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    // 创建一个对象 {destroy:undefined}
    createEffectInstance(),
    nextDeps
  );
}

function updateEffectImpl(
  fiberFlags: Flags,
  hookFlags: HookFlags,
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null
): void {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const effect: Effect = hook.memoizedState;
  const inst = effect.inst;
  // 在渲染阶段state更新后或严格模式下重新渲染时，初始加载时 currentHook 为空
  if (currentHook !== null) {
    if (nextDeps !== null) {
      const prevEffect: Effect = currentHook.memoizedState;
      const prevDeps = prevEffect.deps;
      // 判断前后两个deps是否相等
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        hook.memoizedState = pushEffect(hookFlags, create, inst, nextDeps);
        return;
      }
    }
  }
  // 依赖发生变化时才会打上副作用标记
  currentlyRenderingFiber.flags |= fiberFlags;
  // HookHasEffect 代表需要触发effect回调
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    inst,
    nextDeps
  );
}
```

#### 总结

其实`mount`和 `update` 执行的函数没什么太大区别。`pushEffect`才是核心

- **创建/获取** `hook` 对象，
- 处理`deps`
- 执行`pushEffect`

`hook.memoizedState`里保存的是 `effect` 对象

需要注意的点：

- `PassiveEffect、PassiveStaticEffect` 是 `fiber.flags` 上的标记，用来判断是否有副作用相关的逻辑，跟`HookPassive、HookHasEffect`不一样，`HookPassive`是 `hooks` 上的`tag`标记，别搞混了
- 依赖对比是通过`Object.is`进行比较的，跟 `props` 和 `state` 是否更改的判断方法一样
- `inst` 是个对象，有个`destory`函数，这个就是 `useEffect` 第一个参数回调函数的返回值

#### pushEffect

```js
function pushEffect(
  tag: HookFlags,
  create: () => (() => void) | void,
  inst: EffectInstance,
  deps: Array<mixed> | null
): Effect {
  const effect: Effect = {
    tag,
    // create 为 useEffect第一个参数
    create,
    // inst
    inst,
    // deps 为 useEffect第二个参数
    deps,
    // Circular
    next: (null: any),
  };
  let componentUpdateQueue: null | FunctionComponentUpdateQueue =
    (currentlyRenderingFiber.updateQueue: any);
  if (componentUpdateQueue === null) {
    // 函数式组件的updateQueue结构为
    // {
    //   lastEffect: null,
    //   events: null,
    //   stores: null
    // }
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);
    // 串成环状链表
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      // 说明链表是空的
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      // 取出第一位
      const firstEffect = lastEffect.next;
      // 将最新的effect串在表尾
      lastEffect.next = effect;
      // 闭环
      effect.next = firstEffect;
      // 更新表尾
      componentUpdateQueue.lastEffect = effect;
    }
  }
  return effect;
}
```

##### 总结

`pushEffect`函数主要做了两件事情：

- 创建 `effect`
- 将 effect 串成环状链表，与 fiber.updateQueue.lastEffect 关联

`fiber.updateQueue` 保存的是 `updateQueue` 对象，其中`lastEffect`保存的是 `effect` 对象，是环状链表的表尾，`effect` 对象通过 `next` 连接当前函数组件的所有 `effect`组成环状链表，当前 `useEffect`语句 对应的 `hook` 对象中`memoizedState`保存的是 `effect` 对象，在 `updateEffectImpl` 中会获取，比较前后两个`hook`的依赖是否更改

### useEffect 回调什么时候触发呢？

给 `fiber.flags` 合并上副作用标记之后，会在 `compeleteWork` 阶段将 `flags` 向上冒泡到 `HostRootFiber.subtreeFlags`，在 `commit` 阶段的`commitRootImpl`函数内，通过

```js
if (
  // 判断子节点是否包含PassiveMask
  (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
  // 判断当前节点是否包含PassiveMask
  (finishedWork.flags & PassiveMask) !== NoFlags
) {
  // 是否已经有副作用被调度
  if (!rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = true;
    pendingPassiveEffectsRemainingLanes = remainingLanes;

    pendingPassiveTransitions = transitions;
    // 调度flushPassiveEffects
    scheduleCallback(NormalSchedulerPriority, function () {
      // 这个函数是核心
      flushPassiveEffects();
      return null;
    });
  }
}
// ...
// commitBeforeMutationEffects
// commitMutationEffects
// commitLayoutEffects
// ...
if (rootDoesHavePassiveEffects) {
  // 有有副作用，赋值这些变量
  rootDoesHavePassiveEffects = false;
  rootWithPendingPassiveEffects = root;
  pendingPassiveEffectsLanes = lanes;
}
// ...
// 如果当前渲染是离散渲染（click，touch等事件）,则直接执行flushPassiveEffects
if (
  includesSyncLane(pendingPassiveEffectsLanes) &&
  (disableLegacyMode || root.tag !== LegacyRoot)
) {
  flushPassiveEffects();
}
// ...
```

因为只是个调度，是个异步，并不会马上触发，所以**触发时机在所有渲染流程结束之后触发**

- 触发时机一定是在 `commit` 三大阶段之后触发的，也就是真实 dom 渲染到页面之后触发
- 是不是异步？因为整个渲染流程是 `MessageChannel` 的 `postMessage` 触发的，所以整个渲染流程都是异步，`flushPassiveEffects` 在异步任务中执行，自然也是异步，**但是相对于渲染流程来说不一定是异步**，有两点原因：
  - 通过 `scheduleCallback` 调度的`flushPassiveEffects`，`scheduleCallback`的逻辑是如果当前的任务不超过 5ms，则继续同步执行下一个任务，也就是说如果整个渲染流程不超过 5ms，那么会同步`flushPassiveEffects`
    - 如果超过了 5ms，则通过`MessageChannel`的 `postMessage` 触发`flushPassiveEffects`，就会被放进下一个事件循环中执行
  - 如果当前的 lane 是同步的，则同步执行`flushPassiveEffects`

### flushPassiveEffects

```js
export function flushPassiveEffects(): boolean {
  if (rootWithPendingPassiveEffects !== null) {
    const root = rootWithPendingPassiveEffects;
    const remainingLanes = pendingPassiveEffectsRemainingLanes;
    pendingPassiveEffectsRemainingLanes = NoLanes;
    // 转换成事件优先级
    const renderPriority = lanesToEventPriority(pendingPassiveEffectsLanes);
    // 找到最低事件优先级
    const priority = lowerEventPriority(DefaultEventPriority, renderPriority);
    const prevTransition = ReactSharedInternals.T;
    const previousPriority = getCurrentUpdatePriority();

    try {
      // 设置更新优先级，原生事件触发时也是调用的这个
      // 设置ReactDOMSharedInternals.p
      setCurrentUpdatePriority(priority);
      ReactSharedInternals.T = null;
      return flushPassiveEffectsImpl();
    } finally {
      setCurrentUpdatePriority(previousPriority);
      ReactSharedInternals.T = prevTransition;
      releaseRootPooledCache(root, remainingLanes);
    }
  }
  return false;
}
```

#### flushPassiveEffectsImpl

```js
function flushPassiveEffectsImpl() {
  if (rootWithPendingPassiveEffects === null) {
    return false;
  }

  const transitions = pendingPassiveTransitions;
  pendingPassiveTransitions = null;
  // 保存有副作用的fiber的fiberRootNode和待处理的副作用优先级，然后清空全局变量
  const root = rootWithPendingPassiveEffects;
  const lanes = pendingPassiveEffectsLanes;
  rootWithPendingPassiveEffects = null;

  pendingPassiveEffectsLanes = NoLanes;

  if ((executionContext & (RenderContext | CommitContext)) !== NoContext) {
    throw new Error("Cannot flush passive effects while already rendering.");
  }

  // 保存当前执行上下文，处理完之后恢复
  const prevExecutionContext = executionContext;
  // 进入commit阶段
  // 这个操作除了这个地方之外 还有在commit阶段（commitRootImpl）
  executionContext |= CommitContext;
  // 先执行unMount 后执行mount
  // 两个操作逻辑基本一致 从root开始递归检查subtreeFlags是否有对应flag，
  // 一直继续向下遍历，到最底
  // 深度优先的后续遍历，跟commit阶段一个逻辑
  commitPassiveUnmountEffects(root.current);
  commitPassiveMountEffects(root, root.current, lanes, transitions);

  // 恢复处理effect之前的执行上下文
  executionContext = prevExecutionContext;
  // 所有异步结束之前都要刷新一下同步任务
  flushSyncWorkOnAllRoots();

  return true;
}
```

#### commitHookEffectListUnmount

> unmount 先触发

```js
function commitHookEffectListUnmount(
  flags: HookFlags,
  finishedWork: Fiber,
  nearestMountedAncestor: Fiber | null
) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    // 遍历所有effect
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      //
      if ((effect.tag & flags) === flags) {
        // 是否包含flags
        // Unmount
        const inst = effect.inst;
        const destroy = inst.destroy;
        if (destroy !== undefined) {
          // 清空destroy，以便重新生成
          inst.destroy = undefined;
          // 执行destroy
          safelyCallDestroy(finishedWork, nearestMountedAncestor, destroy);
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

#### commitPassiveMountEffects

```js
function commitHookEffectListMount(flags: HookFlags, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & flags) === flags) {
        // Mount
        const create = effect.create;
        const inst = effect.inst;
        // 执行useEffect第一个参数
        const destroy = create();
        // 更新destroy
        inst.destroy = destroy;
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

##### 总结

可以看到总体的 `useEffect` 执行流程，`destroy`先执行（第一次没有，因为还没有执行回调哪来的返回值），顺序是由**子->父**，然后再执行回调，顺序是**子->父**

# useLayoutEffect

## mountEffectImpl / updateEffectImpl

```js
mountEffectImpl(UpdateEffect | LayoutStaticEffect, HookLayout, create, deps);

updateEffectImpl(UpdateEffect, HookLayout, create, deps);
```

可以看到跟 `useEffect` 调用的是同一个函数，但是 `fiberFlags` 传的是 `Update`,`effect` 的 `tag` 是 `HookLayout`

### 触发时机在哪呢

在 `commit` 阶段的 `layout` 阶段

```js
// commitRoot中设置优先级为离散事件优先级，也就是SyncLane
setCurrentUpdatePriority(DiscreteEventPriority);
// ⬇️ `layout` 阶段
commitLayoutEffects(finishedWork, root, lanes);
// ⬇️
commitLayoutEffectOnFiber(root, current, finishedWork, committedLanes);
// ⬇️
if (flags & Update) {
  // 同步去执行useLayoutEffect的回调
  // 这里传入的是HookLayout | HookHasEffect 刚好跟useLayoutEffect的effect.tag对应
  commitHookLayoutEffects(finishedWork, HookLayout | HookHasEffect);
}
// ⬇️
// 上文讲过就不重复了
commitHookEffectListMount(hookFlags, finishedWork);
```

#### 总结

可以看到回调是在 `commitLayoutEffects` 阶段执行的，而且通过在 `commitRoot` 函数内设置了同步 `lane。`
如果回调内调用 `setState`，那么调度出的任务也是个同步任务，在 `commit` 三大阶段完成后会刷新同步任务，所以可以看到：如果在回调内更改 `state`，页面并不会闪烁
`useEffect` 打上的 fiber.flags 是 `passive`，`useLayoutEffect` 打上的是 `update`
