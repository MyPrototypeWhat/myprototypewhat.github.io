---
title: React源码解析---React入口
toc: true
tags:
  - React
date: 2024-06-07 19:57:22
---

> React@v19

# 前言

这章会详细讲解：

- 渲染一个`React`应用都做了什么
- fiber 的部分属性
- 如何调度一个渲染任务

# ReactDom

常见 React 应用写法

```js
const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<App />);
```

`root.render`为入口开启了`React`应用的渲染

## createRoot

```js
export function createRoot(
  container: Element | Document | DocumentFragment,
  options?: CreateRootOptions
): RootType {
  // 是否是有效的根节点
  if (!isValidContainer(container)) {
    throw new Error("Target container is not a DOM element.");
  }

  let isStrictMode = false;
  let concurrentUpdatesByDefaultOverride = false;
  let identifierPrefix = "";
  let onUncaughtError = defaultOnUncaughtError;
  let onCaughtError = defaultOnCaughtError;
  let onRecoverableError = defaultOnRecoverableError;
  let transitionCallbacks = null;

  if (options !== null && options !== undefined) {
    // 配置项的处理
  }
  // 关键，会创建两个fiber，一个是FiberRootNode，一个是HostRootFiber
  // createContainer 调用 createFiberRoot
  const root = createContainer(
    container, // dom根节点
    ConcurrentRoot, // 1
    null,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
    identifierPrefix,
    onUncaughtError,
    onCaughtError,
    onRecoverableError,
    transitionCallbacks
  );
  // 给hostRootFiber上挂上一个属性
  // node[internalContainerInstanceKey] = hostRootFiber;
  // 方便后续某些函数从dom节点找HostRootFiber
  markContainerAsRoot(root.current, container);
  // 如果是个注释节点，取父节点
  const rootContainerElement =
    container.nodeType === COMMENT_NODE ? container.parentNode : container;
  // 给根节点绑定事件，分捕获和非捕获，还有一些无法绑定在根节点上的会绑在document上
  // 具体的后面会单独开一章
  listenToAllSupportedEvents(rootContainerElement);

  return new ReactDOMRoot(root);
}
```

### ReactDOMRoot

```js
function ReactDOMRoot(internalRoot) {
  this._internalRoot = internalRoot; // FiberRootNode
}

ReactDOMHydrationRoot.prototype.render = ReactDOMRoot.prototype.render =
  function () {}; // 渲染函数
ReactDOMHydrationRoot.prototype.unmount = ReactDOMRoot.prototype.unmount =
  function () {}; // 卸载
```

### createFiberRoot

```js
function createFiberRoot(containerInfo, tag，// 其余参数省略
) {
  // fiber相关属性太多，没必要现在了解，用到什么属性再看就行
  // 创建FiberRootNode
  var root = new FiberRootNode(containerInfo, tag, hydrate, identifierPrefix, onUncaughtError, onCaughtError, onRecoverableError, formState);
	// 创建HostRootFiber
  var uninitializedFiber = createHostRootFiber(tag, isStrictMode);
  // 互指
  // 在render阶段结束之后会进行树的切换,就是讲current指向新树
  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;

  var initialState = {
    element: initialChildren, // null
    isDehydrated: hydrate,
    cache: null
  };
  uninitializedFiber.memoizedState = initialState; // null
	// 初始化UpdateQueue，挂载在HostRootFiber
  initializeUpdateQueue(uninitializedFiber);
  // 返回FiberRootNode
  return root;
}
```

#### initializeUpdateQueue

```js
export function initializeUpdateQueue(fiber) {
  const queue: UpdateQueue = {
    // basXXX代表已处理过的更新
    baseState: fiber.memoizedState,
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: {
      pending: null,
      lanes: NoLanes,
      hiddenCallbacks: null,
    },
    callbacks: null,
  };
  fiber.updateQueue = queue;
}
```

## 总结

- 创建两个 fiber 节点(`FiberRootNode `\ `HostRootFiber`),初始化 HostRootFiber 的`updateQueue`
- 给`dom`根节点绑定事件
- 返回一个`new ReactDOMRoot`
  - `ReactDOMRoot`上挂了一个属性`_internalRoot`，指向`FiberRootNode`
  - 原型上挂载了`render` \ `unmount`

下面执行`render`函数，正式开启`react`应用的渲染

## render

```js
ReactDOMRoot.prototype.render = function (children) {
  const root = this._internalRoot;
  // ...省略参数检查
  updateContainer(children, root, null, null);
};
```

### updateContainer

```js
export function updateContainer(
  element,
  container,
  parentComponent,
  callback
): Lane {
  // HostRootFiber
  const current = container.current;
  // 获取此次更新优先级
  const lane = requestUpdateLane(current);
  updateContainerImpl(
    current,
    lane,
    element,
    container,
    parentComponent,
    callback
  );
  return lane;
}
```

#### requestUpdateLane

```js
function requestUpdateLane(fiber) {
  // 当前执行上下文为render，也就是在render阶段
  if (
    (executionContext & RenderContext) !== NoContext &&
    // 渲染任务未结束
    workInProgressRootRenderLanes !== NoLanes
  ) {
    // 从workInProgressRootRenderLanes中获取最高优先级车道
    return pickArbitraryLane(workInProgressRootRenderLanes);
  }
  // 下面就是transition相关逻辑，暂时不用看
  var transition = requestCurrentTransition();

  if (transition !== null) {
    {
      if (!transition._updatedFibers) {
        transition._updatedFibers = new Set();
      }

      transition._updatedFibers.add(fiber);
    }

    var actionScopeLane = peekEntangledActionLane();
    return actionScopeLane !== NoLane
      ? actionScopeLane
      : requestTransitionLane();
  }
  // 事件优先级转lane
  return eventPriorityToLane(resolveUpdatePriority());
}
```

##### resolveUpdatePriority

```js
function resolveUpdatePriority() {
  // 找有没有已保存的优先级
  var updatePriority = Internals.p;

  if (updatePriority !== NoEventPriority) {
    return updatePriority;
  }
  // 当前事件对象
  var currentEvent = window.event;

  if (currentEvent === undefined) {
    // 没有就取默认的
    return DefaultEventPriority;
  }

  return getEventPriority(currentEvent.type);
}
```

不同事件对应的优先级如下

```js
var NoEventPriority = NoLane; // 0
// 离散事件优先级 例如click,touchend,touchstart
var DiscreteEventPriority = SyncLane; // 2
// 连续事件优先级 例如mousemove,touchmove,drag
var ContinuousEventPriority = InputContinuousLane; // 8
// 默认优先级
var DefaultEventPriority = DefaultLane; // 32
// 空闲事件优先级 这个很少用到，跟调度器有关
var IdleEventPriority = IdleLane;
```

#### updateContainerImpl

```js
function updateContainerImpl(
  rootFiber: Fiber,
  lane: Lane,
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function
): void {
  // 获取子树context 基本都是空对象 没意义
  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }
  // 创建一个update对象
  const update = createUpdate(lane);
  // element对象 也就是jsx解析出来的
  // 例如:上述的例子中可以看出这个element是render函数的参数children
  // 只有HostRootFiber的update才有element属性
  update.payload = { element };
  // 修正callback的值
  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    update.callback = callback;
  }
  // 该函数用于将更新添加到 Fiber 对象的updateQueue中。根据不同的条件，更新可能会被立即处理或在后续的并发渲染过程中处理。
  const root = enqueueUpdate(rootFiber, update, lane);
  if (root !== null) {
    // 重点！！调度更新
    scheduleUpdateOnFiber(root, rootFiber, lane);
    // transition相关
    entangleTransitions(root, rootFiber, lane);
  }
}
```

##### enqueueUpdate

> packages/react-reconciler/src/ReactFiberClassUpdateQueue.js

```js
export function enqueueUpdate<State>(
  fiber: Fiber,
  update: Update<State>,
  lane: Lane
): FiberRoot | null {
  const updateQueue = fiber.updateQueue;
  if (updateQueue === null) {
    // 仅在fiber已卸载的情况下发生。
    return null;
  }

  const sharedQueue = updateQueue.shared;
  // 这个更新是一个不安全的render阶段的更新
  // 需要立刻将update插进队列中，这样就能立即处理他
  if (isUnsafeClassRenderPhaseUpdate(fiber)) {
    // 目前为止第一次出现环状链表
    // 不明白的可以多看几遍，后面这种操作非常多

    // pending是表尾，始终站在最新的节点上
    const pending = sharedQueue.pending;
    if (pending === null) {
      update.next = update;
    } else {
      // 新的update指向表头
      update.next = pending.next;
      // 表尾追加新的update
      pending.next = update;
    }
    // 更新pending
    sharedQueue.pending = update;
    // 这个函数就不赘述了
    // 大概就是从当前这个fiber向上找到FiberRootNode，然后向上冒泡lanes
    // unsafe_markUpdateLaneFromFiberToRoot返回值是FiberRootNode
    return unsafe_markUpdateLaneFromFiberToRoot(fiber, lane);
  } else {
    // 将更新排队，返回当前更新的fiber的FiberRootNode
    return enqueueConcurrentClassUpdate(fiber, sharedQueue, update, lane);
  }
}
```

###### enqueueConcurrentClassUpdate

```js
export function enqueueConcurrentClassUpdate<State>(
  fiber: Fiber,
  queue: ClassQueue<State>,
  update: ClassUpdate<State>,
  lane: Lane
): FiberRoot | null {
  const concurrentQueue = queue;
  const concurrentUpdate = update;
  // 排队更新
  // 跟上面函数同名但不是一个文件
  enqueueUpdate(fiber, concurrentQueue, concurrentUpdate, lane);
  // 获取更新所在的节点的根（FiberRootNode）
  return getRootForUpdatedFiber(fiber);
}
```

**enqueueUpdate**

> packages/react-reconciler/src/ReactFiberConcurrentUpdates.js

```js
function enqueueUpdate(
  fiber: Fiber,
  queue: ConcurrentQueue | null,
  update: ConcurrentUpdate | null,
  lane: Lane
) {
  // 依次放进concurrentQueues
  concurrentQueues[concurrentQueuesIndex++] = fiber; // 此次更新的fiber
  concurrentQueues[concurrentQueuesIndex++] = queue; // 更新队列(updateQueue.shared)
  concurrentQueues[concurrentQueuesIndex++] = update; // 更新对象
  concurrentQueues[concurrentQueuesIndex++] = lane; // 更新lanes
  // 更新 并发更新lanes（将update的lanes合并到concurrentlyUpdatedLanes中）
  concurrentlyUpdatedLanes = mergeLanes(concurrentlyUpdatedLanes, lane);
  // 更新 当前fiber的lanes（将update的lanes合并到当前fiber的lanes中）
  fiber.lanes = mergeLanes(fiber.lanes, lane);
  const alternate = fiber.alternate;
  // 如果有alternate也一样
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
}
```

## 总结

> 可以先暂停一下 `render`入口渲染到这就结束了，后续就是调度更新了，

- `updateContainer`中计算出本次更新的`lane`（从原生事件中获取，默认为 32），执行`updateContainerImpl`
- `updateContainerImpl`根据`lane`创建`update`对象，执行`enqueueUpdate`将`update`进行排队,执行`scheduleUpdateOnFiber`调度更新
- `enqueueUpdate`根据是否是`render`阶段中的更新来判断`update`立即插入还是先排队（`enqueueConcurrentClassUpdate`）等待后续插入
- `enqueueConcurrentClassUpdate`执行`enqueueUpdate`将更新排队
- `enqueueUpdate`将`fiber、queue、update、lane`依次插入`concurrentQueues`，更新相关`lanes`变量
