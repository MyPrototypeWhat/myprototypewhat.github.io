---
title: React@v19源码解析---useContext
toc: true
tags:
  - React
date: 2024-06-07 19:52:42
---

# useContext

> 调用的是 `readContext`，`mount` 和 `update` 调用的一样

```js
export function readContext<T>(context: ReactContext<T>): T {
  return readContextForConsumer(currentlyRenderingFiber, context);
}
```

## createContext

> 先看下创建的过程

```js
export function createContext<T>(defaultValue: T): ReactContext<T> {
  const context: ReactContext<T> = {
    $$typeof: REACT_CONTEXT_TYPE,
    // 两个值是为了并发渲染器
    // react Dom中使用_currentValue
    // react art中使用_currentValue2
    _currentValue: defaultValue,
    _currentValue2: defaultValue,
    // 用于跟踪此上下文目前在单个渲染器中支持多少个并发渲染器。例如并行服务端渲染。
    _threadCount: 0,
    Provider: (null: any),
    Consumer: (null: any),
  };

  context.Provider = context;
  context.Consumer = {
    $$typeof: REACT_CONSUMER_TYPE,
    _context: context,
  };

  return context;
}
```

`createContext`生成一个 `context` 对象，并且有两个属性，一个是 ·，一个是 ` Consumer。``Provider `类型是`REACT_CONTEXT_TYPE` `Consumer` 类型是 `REACT_CONSUMER_TYPE`，在生成对应 fiber 的时候会用到

## readContextForConsumer

```js
function readContextForConsumer<T>(
  consumer: Fiber | null,
  context: ReactContext<T>
): T {
  // 拿到context值
  const value = isPrimaryRenderer
    ? context._currentValue
    : context._currentValue2;
  // lastFullyObservedContext只等于null
  if (lastFullyObservedContext === context) {
    // Nothing to do. We already observe everything in this context.
    // 没什么可做的
  } else {
    const contextItem = {
      context: ((context: any): ReactContext<mixed>),
      memoizedValue: value,
      next: null,
    };

    if (lastContextDependency === null) {
      // consumer就是当前组件的fiber
      if (consumer === null) {
        // 只有在 React 渲染时才能读取上下文。
        // 在类中，您可以在 render 方法或 getDerivedStateFromProps 中读取它。
        // 在函数组件中，可以直接在函数体中读取，但不能在钩子（如 useReducer() 或 useMemo()）中读取。
        throw new Error(
          "Context can only be read while React is rendering. " +
            "In classes, you can read it in the render method or getDerivedStateFromProps. " +
            "In function components, you can read it directly in the function body, but not " +
            "inside Hooks like useReducer() or useMemo()."
        );
      }

      // 这是该组件的第一个依赖项。创建新列表。
      lastContextDependency = contextItem;
      // 设置fiber.dependencies
      consumer.dependencies = {
        lanes: NoLanes,
        firstContext: contextItem,
      };
    } else {
      // 链接下一个context
      lastContextDependency = lastContextDependency.next = contextItem;
    }
  }
  // 返回value
  return value;
}
```

## 什么时候会更新 context

从上面的代码可以看出，`contex`t 的 `value` 保存在 `context._currentValue`，那什么时候会更新这个值呢？
是不是通过`<Context.Provider value={value} />`中 `value` 来更新 `context` 呢,来看下`Provider`

## Provider

> 从 `createContext` 中得知 `Provider` 只是一个对象
> `context.Provider = context;`

看下 `beginWork` 中对于 `REACT_CONTEXT_TYPE` 这个类型的处理

```jsx
// beiginWork
// ...
case ContextProvider:
      return updateContextProvider(current, workInProgress, renderLanes);
// ...
```

### updateContextProvider

```js
function updateContextProvider(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  let context: ReactContext<any>;
  if (enableRenderableContext) {
    // 拿到context对象，<Context>等同于<Context.Provider>
    // 因为createContext 中 context.Provider = context;
    context = workInProgress.type;
  } else {
    context = workInProgress.type._context;
  }
  const newProps = workInProgress.pendingProps;
  const oldProps = workInProgress.memoizedProps;
  // value就是组件中传的value值
  const newValue = newProps.value;
  // 更新context
  pushProvider(workInProgress, context, newValue);

  if (oldProps !== null) {
    const oldValue = oldProps.value;
    if (is(oldValue, newValue)) {
      // context没有更改，如果children没有更改就命中bailout
      if (oldProps.children === newProps.children) {
        return bailoutOnAlreadyFinishedWork(
          current,
          workInProgress,
          renderLanes
        );
      }
    } else {
      // context更改，安排更新
      propagateContextChange(workInProgress, context, renderLanes);
    }
  }

  const newChildren = newProps.children;
  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  return workInProgress.child;
}
```

#### pushProvider

> 删除`__dev__`相关代码

```js
export function pushProvider<T>(
  providerFiber: Fiber,
  context: ReactContext<T>,
  nextValue: T
): void {
  push(valueCursor, context._currentValue, providerFiber);
  // 更新
  context._currentValue = nextValue;
}
```

#### propagateContextChange

> context 更新了，如何通知到使用的组件

```js
export function propagateContextChange<T>(
  workInProgress: Fiber,
  context: ReactContext<T>,
  renderLanes: Lanes
): void {
  propagateContextChange_eager(workInProgress, context, renderLanes);
}

function propagateContextChange_eager<T>(
  workInProgress: Fiber,
  context: ReactContext<T>,
  renderLanes: Lanes
): void {
  // Only used by eager implementation
  if (enableLazyContextPropagation) {
    return;
  }
  let fiber = workInProgress.child;
  if (fiber !== null) {
    // 更改指向，更新阶段的workInProgress是从旧节点的current.alternate赋值过来的，
    // child.return 的指向是旧节点，所以要更正
    fiber.return = workInProgress;
  }
  // 从上到下遍历子节点
  while (fiber !== null) {
    let nextFiber;

    // Visit this fiber.
    const list = fiber.dependencies;
    if (list !== null) {
      nextFiber = fiber.child;

      let dependency = list.firstContext;
      while (dependency !== null) {
        if (dependency.context === context) {
          // 如果子节点有使用这个context，就在这个fiber上调度一个更新
          // 如果是class组件，作者基本不用类组件了，直接忽略了
          // 大概流程就是创建update，然后挂载fiber.updateQueue上
          if (fiber.tag === ClassComponent) {
            const lane = pickArbitraryLane(renderLanes);
            const update = createUpdate(lane);
            update.tag = ForceUpdate;
            const updateQueue = fiber.updateQueue;
            if (updateQueue === null) {
              // 组件已被卸载
            } else {
              const sharedQueue: SharedQueue<any> = (updateQueue: any).shared;
              const pending = sharedQueue.pending;
              if (pending === null) {
                // This is the first update. Create a circular list.
                update.next = update;
              } else {
                update.next = pending.next;
                pending.next = update;
              }
              sharedQueue.pending = update;
            }
          }
          // 更新lanes
          fiber.lanes = mergeLanes(fiber.lanes, renderLanes);
          // 同步更新 alternate
          const alternate = fiber.alternate;
          if (alternate !== null) {
            alternate.lanes = mergeLanes(alternate.lanes, renderLanes);
          }
          // 向上冒泡更新父组件的 childLanes
          scheduleContextWorkOnParentPath(
            fiber.return,
            renderLanes,
            workInProgress
          );

          // 更新fiber的dependencies.lanes
          list.lanes = mergeLanes(list.lanes, renderLanes);

          // 已经找到了依赖，停止遍历依赖链表
          break;
        }
        // 没找到就继续;
        dependency = dependency.next;
      }
    } else if (fiber.tag === ContextProvider) {
      // 如果子节点是 Provider
      // 如果使用的是同一个context，就不需要向下遍历了，因为子节点的Provider也会走这部分逻辑
      nextFiber = fiber.type === workInProgress.type ? null : fiber.child;
    } else {
      nextFiber = fiber.child;
    }

    if (nextFiber !== null) {
      // 跟上面相同操作
      nextFiber.return = fiber;
    } else {
      // 没子节点了就跳到兄弟节点
      nextFiber = fiber;
      while (nextFiber !== null) {
        if (nextFiber === workInProgress) {
          // 到父节点结束
          nextFiber = null;
          break;
        }
        const sibling = nextFiber.sibling;
        if (sibling !== null) {
          // 跟上面相同操作
          sibling.return = nextFiber.return;
          nextFiber = sibling;
          break;
        }
        // 没兄弟节点，返回到父节点
        nextFiber = nextFiber.return;
      }
    }
    fiber = nextFiber;
  }
}
```

## 总结

看完上述代码可能会有几个问题:

1. `context` 更改导致重新渲染的整个流程是什么

   1. 首先肯定是某个 `state` 更改调度一个更新
   2. 然后在 `beginWork` 中，走到`updateContextProvider`函数中
   3. 用 `props` 中的`value`更新 `context._currentValue`
   4. 判断新旧 `value` 是否相同
      1. 相同则继续判断 `children` 是否有更改，如果没有则命中 `bailout` 提前结束
      2. 不同则执行 `propagateContextChange`=>`propagateContextChange_eager`更新子节点的 `lanes`

2. `dependencies.lanes` 有什么用?

```js
// updateFunctionComponents函数中
// ⬇️
prepareToReadContext(workInProgress, renderLanes);
// ⬇️
if (includesSomeLane(dependencies.lanes, renderLanes)) {
  // 如果当前渲染任务中包含dependencies.lanes,就标记为有更新
  didReceiveUpdate = true;
}
// 重置，在useContext中重新生成
dependencies.firstContext = null;
```

3. `context` 是以什么形式存在的？

   1. 首先通过 `React.createContext` 创建一个 `context` 对象
   2. `useContext` 中创建`contextItem`通过 `next` 链接下一个 `context`，保存在 `fiber.dependencies.firstContext` 中

4. `contextItem` 是通过 `lastContextDependency` 穿起来的，如何保证这个变量的生命周期只在当前 `fiber` 节点内？
   `lastContextDependency`变量会在`prepareToReadContext`函数内重置

```js
currentlyRenderingFiber = workInProgress;
lastContextDependency = null;
lastFullyObservedContext = null;
```

5. `<Context>`等同于`<Context.Provider>`，具体看[issue](https://github.com/facebook/react/pull/28226)
