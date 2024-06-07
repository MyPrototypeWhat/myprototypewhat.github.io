---
title: React@v19源码解析---Suspense原理
toc: true
tags:
  - React
date: 2024-06-07 19:53:42
---

# 前言

**看之前应该对 `React` 的完整渲染流程有一个清晰的认知，否则很容易迷茫。**

# Suspense

## 例子

```js
// Test.js
const promsie = new Promise((resolve) => {
  setTimeout(resolve, 1000, "state");
});

const Test = () => {
  const state = use(promsie);
  return <div>{state}</div>;
};

// App.js
function App() {
  return (
    <Suspense fallback={<div>loading...</div>}>
      <Test />
    </Suspense>
  );
}
```

**现象就是现`loading...`然后`state`**

## `<Suspense>`

> 只是一个类型

```js
export const REACT_SUSPENSE_TYPE: symbol = Symbol.for("react.suspense");
```

## updateSuspenseComponent

分为三大部分：

- 变量判断
- 渲染阶段
- 更新阶段

### 变量判断

```js
// 拿到props
const nextProps = workInProgress.pendingProps;

let showFallback = false;
// 是否挂起
const didSuspend = (workInProgress.flags & DidCapture) !== NoFlags;
if (
  didSuspend ||
  shouldRemainOnFallback(current, workInProgress, renderLanes)
) {
  // 已经挂起，渲染 fallback
  showFallback = true;
  // 去掉DidCapture
  workInProgress.flags &= ~DidCapture;
}

// useDeferredValue相关
const didPrimaryChildrenDefer = (workInProgress.flags & DidDefer) !== NoFlags;
workInProgress.flags &= ~DidDefer;
```

### 渲染阶段

```js
if (current === null) {
  // Initial mount
  // 分别拿到children和fallback
  const nextPrimaryChildren = nextProps.children;
  const nextFallbackChildren = nextProps.fallback;
  // 展示fallback，可以先不看
  if (showFallback) {
    // 渲染重新生成children对应的fiber和fallback对应的fiber
    const fallbackFragment = mountSuspenseFallbackChildren(
      workInProgress,
      nextPrimaryChildren,
      nextFallbackChildren,
      renderLanes
    );
    const primaryChildFragment: Fiber = (workInProgress.child: any);
    // 设置memoizedState
    primaryChildFragment.memoizedState =
      mountSuspenseOffscreenState(renderLanes);
    // 设置childLanes
    primaryChildFragment.childLanes = getRemainingWorkInPrimaryTree(
      current,
      didPrimaryChildrenDefer,
      renderLanes
    );
    // suspense fiber.memoizedState
    // SUSPENDED_MARKER = {
    //   dehydrated: null,
    //   treeContext: null,
    //   retryLane: NoLane,
    // };
    workInProgress.memoizedState = SUSPENDED_MARKER;
    // 返回fallback fiber
    return fallbackFragment;
  } else {
    // 更新suspenseHandlerStackCursor.current = workInProgress
    pushPrimaryTreeSuspenseHandler(workInProgress);
    return mountSuspensePrimaryChildren(
      workInProgress,
      nextPrimaryChildren,
      renderLanes
    );
  }
}
```

#### mountSuspenseFallbackChildren

```js
function mountSuspenseFallbackChildren(
  workInProgress: Fiber,
  primaryChildren: $FlowFixMe,
  fallbackChildren: $FlowFixMe,
  renderLanes: Lanes
) {
  const mode = workInProgress.mode;

  const primaryChildProps: OffscreenProps = {
    mode: "hidden",
    children: primaryChildren,
  };

  let primaryChildFragment;
  let fallbackChildFragment;
  // 重新生成children对应的fiber
  primaryChildFragment = mountWorkInProgressOffscreenFiber(
    primaryChildProps,
    mode,
    NoLanes
  );
  // 生成fallback对应的fiber
  fallbackChildFragment = createFiberFromFragment(
    fallbackChildren,
    mode,
    renderLanes,
    null
  );
  // 指向共同的父节点
  primaryChildFragment.return = workInProgress;
  fallbackChildFragment.return = workInProgress;
  // chidren 指向fallback
  primaryChildFragment.sibling = fallbackChildFragment;
  // children作为firstChild
  workInProgress.child = primaryChildFragment;
  // 将fallback fiber 返回出去
  return fallbackChildFragment;
}
```

#### mountSuspensePrimaryChildren

```js
function mountSuspensePrimaryChildren(
  workInProgress: Fiber,
  primaryChildren: $FlowFixMe,
  renderLanes: Lanes
) {
  const mode = workInProgress.mode;
  // 创建子节点props
  const primaryChildProps: OffscreenProps = {
    mode: "visible",
    children: primaryChildren,
  };
  // 创建子节点
  const primaryChildFragment = mountWorkInProgressOffscreenFiber(
    primaryChildProps,
    mode,
    renderLanes
  );
  // 链接父节点和子节点
  primaryChildFragment.return = workInProgress;
  workInProgress.child = primaryChildFragment;
  return primaryChildFragment;
}
```

##### mountWorkInProgressOffscreenFiber / createFiberFromOffscreen

```js
function mountWorkInProgressOffscreenFiber(
  offscreenProps: OffscreenProps,
  mode: TypeOfMode,
  renderLanes: Lanes
) {
  return createFiberFromOffscreen(offscreenProps, mode, NoLanes, null);
}

export function createFiberFromOffscreen(
  pendingProps: OffscreenProps,
  mode: TypeOfMode,
  lanes: Lanes,
  key: null | string
): Fiber {
  // 创建Offscreenfiber
  const fiber = createFiber(OffscreenComponent, pendingProps, key, mode);
  // REACT_OFFSCREEN_TYPE = Symbol.for('react.offscreen')
  fiber.elementType = REACT_OFFSCREEN_TYPE;
  fiber.lanes = lanes;
  const primaryChildInstance: OffscreenInstance = {
    _visibility: OffscreenVisible,
    _pendingVisibility: OffscreenVisible,
    _pendingMarkers: null,
    _retryCache: null,
    _transitions: null,
    _current: null,
    detach: () => detachOffscreenInstance(primaryChildInstance),
    attach: () => attachOffscreenInstance(primaryChildInstance),
  };
  fiber.stateNode = primaryChildInstance;
  return fiber;
}
```

##### 总结

上面代码可以看到：在首次渲染阶段，会优先展示 `fallback`，然后把子节点当做 `offscreen` 来生成 `fiber`，然后挂载到父节点上。继续渲染流程。
下一步 `beginWork` 走到了 `OffscreenComponent` 这个判断中

```js
case OffscreenComponent:
    return updateOffscreenComponent(current, workInProgress, renderLanes);
```

#### updateOffscreenComponent

```js
function updateOffscreenComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  const nextProps: OffscreenProps = workInProgress.pendingProps;
  const nextChildren = nextProps.children;

  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

没什么特别的，正常的渲染流程，执行到函数组件中的 `use`，`use` 会将 `promise` 放在全局变量中，然后抛出一个异常，被 `catch` 住之后走到 `handleThrow` 中。

<!-- ### 更新阶段 -->

## handleThrow

```js
function handleThrow(root: FiberRoot, thrownValue: any): void {
  // 这些内容应立即重置，因为它们只应在 React 执行用户代码时设置。
  // 重置currentlyRenderingFiber和触发hooks的dispatcher
  resetHooksAfterThrow();
  // 例子中use语法会抛出这个错误
  if (thrownValue === SuspenseException) {
    // 这是一种用于 Suspense 的特殊异常类型。
    // 由于历史原因，Suspense 实现的其他部分希望抛出的值是 thenable，
    // 因为在 `use` 存在之前，这是用于suspending 的（不稳定的） API。
    // 一旦我们弃用旧的 API 而改用 `use`，这个实现细节就会随之改变。

    // thrownValue原本是一个error，将它更改为promise
    // 从suspendedThenable变量中获取，之后将它置为null
    thrownValue = getSuspendedThenable();
    // 这个变量会在renderRootXXX中switch里做不同的处理
    workInProgressSuspendedReason =
      // 判断是否停留在上一个画面
      shouldRemainOnPreviousScreen() &&
      // 检查是否有其他待处理的更新可能解除对此组件的suspend。
      !includesNonIdleWork(workInProgressRootSkippedLanes) &&
      !includesNonIdleWork(workInProgressRootInterleavedUpdatedLanes)
        ? // 挂起任务直到resolve
          SuspendedOnData
        : // 除了检查数据是否已经resolve（即在微任务中），不要暂停工作循环。
          // 否则，触发最近的 Suspense fallback。
          SuspendedOnImmediate;
  } else if (thrownValue === SuspenseyCommitException) {
    thrownValue = getSuspendedThenable();
    workInProgressSuspendedReason = SuspendedOnInstance;
  } else if (thrownValue === SelectiveHydrationException) {
    workInProgressSuspendedReason = SuspendedOnHydration;
  } else {
    // 常规错误
    const isWakeable =
      thrownValue !== null &&
      typeof thrownValue === "object" &&
      typeof thrownValue.then === "function";

    workInProgressSuspendedReason = isWakeable
      ? SuspendedOnDeprecatedThrowPromise
      : SuspendedOnError;
  }

  workInProgressThrownValue = thrownValue;

  const erroredWork = workInProgress;
  if (erroredWork === null) {
    // This is a fatal error
    // 这是一个致命错误
    // 说明还没开始渲染就报错了
    workInProgressRootExitStatus = RootFatalErrored;
    logUncaughtError(
      root,
      createCapturedValueAtFiber(thrownValue, root.current)
    );
    return;
  }
}
```

```js
export function shouldRemainOnPreviousScreen(): boolean {
  // 这是在问，挂起transition并停留在上一屏幕，与尽快显示 fallback 是否更好。
  // 它既要考虑render的优先级，也要考虑显示 fallback 是否会带来理想的用户体验。
  // 获取最近的suspense fiber
  // suspenseHandlerStackCursor.current
  const handler = getSuspenseHandler();
  if (handler === null) {
    // root into a "detached" mode.
    // 没有可以提供 fallback 的 Suspense 结界。
    // 我们别无选择，只能停留在前一个屏幕上。
    // 注意：即使是同步更新，我们也会这样做，因为没有更好的选择。
    // 将来，我们可能会改变处理方式，比如将整个根目录设置为 "分离 "模式。
    return true;
  }
  // 判断当前渲染任务是否只有transition
  if (includesOnlyTransitions(workInProgressRootRenderLanes)) {
    if (getShellBoundary() === null) {
      return true;
    } else {
      return false;
    }
  }
  // 判断当前渲染任务是否只有retry 或者 包含OffscreenLane
  if (
    includesOnlyRetries(workInProgressRootRenderLanes) ||
    includesSomeLane(workInProgressRootRenderLanes, OffscreenLane)
  ) {
    return handler === getShellBoundary();
  }

  // 除transition和retry之外的所有lanes了，我们不应等待数据加载。
  return false;
}
```

`getShellBoundary`函数获取`shellBoundary`，`shellBoundary`代表最外层的 `sususpense` 组件.

## renderRootSync

当`handleThrow`执行完之后，由于 `while(true)`所以继续执行下面的判断逻辑

```js
// ...
if (
    workInProgressSuspendedReason !== NotSuspended &&
    workInProgress !== null
  ) {
    // 工作循环暂停 在同步渲染过程中，我们不会让位于主线程。
    // 而是立即释放堆栈。这将触发 fallback 或错误边界。
    const unitOfWork = workInProgress;
    const thrownValue = workInProgressThrownValue;
    switch (workInProgressSuspendedReason) {
      case SuspendedOnHydration: {
        // Selective hydration. An update flowed into a dehydrated tree.
        // Interrupt the current render so the work loop can switch to the
        // hydration lane.
        resetWorkInProgressStack();
        workInProgressRootExitStatus = RootDidNotComplete;
        break outer;
      }
      case SuspendedOnImmediate:
      case SuspendedOnData: {
        // getSuspenseHandler() === null 表示没有fallback
        // didSuspendInShell为true时会检测是否因为没有缓存promise而导致无限ping循环
        if (!didSuspendInShell && getSuspenseHandler() === null) {
          didSuspendInShell = true;
        }
        // Intentional fallthrough
      }
      default: {
        workInProgressSuspendedReason = NotSuspended;
        workInProgressThrownValue = null;
        // 核心
        throwAndUnwindWorkLoop(root, unitOfWork, thrownValue);
        // 终止循环
        break;
      }
    }
  }
// ...
```

### throwAndUnwindWorkLoop

```js
function throwAndUnwindWorkLoop(
  root: FiberRoot,
  unitOfWork: Fiber,
  thrownValue: mixed
) {
  // 这是 performUnitOfWork 的一个分叉，专门用于解除抛出异常的 fiber。

  // 重置hook相关的所有全局变量
  resetSuspendedWorkLoopOnUnwind(unitOfWork);
  // 获取父节点，基本没用到这个变量
  const returnFiber = unitOfWork.return;
  try {
    // 找到并标记最近的可以处理此 "异常 "的 Suspense 或错误边界。
    const didFatal = throwException(
      root,
      returnFiber,
      unitOfWork,
      thrownValue,
      workInProgressRootRenderLanes
    );
    if (didFatal) {
      // 结束了 致命错误无法恢复
      panicOnRootError(root, thrownValue);
      return;
    }
  } catch (error) {
    // 我们在处理错误时遇到了麻烦。
    // 发生这种情况的一个例子是访问错误边界的 `componentDidCatch` 属性时抛出了错误。
    // 这是一个奇怪的边缘情况。
    // 对此有一个回归测试。为防止出现无限循环，可将错误上升到下一个父级
    if (returnFiber !== null) {
      // 提升到父级并且抛出错误继续处理
      workInProgress = returnFiber;
      throw error;
    } else {
      // 结束了 致命错误无法恢复
      panicOnRootError(root, thrownValue);
      return;
    }
  }

  if (unitOfWork.flags & Incomplete) {
    // 向上找到最近的错误边界
    unwindUnitOfWork(unitOfWork);
  } else {
    completeUnitOfWork(unitOfWork);
  }
}
```

#### throwException

> 删除部分 hydrate 和 dev 以及实验功能代码

```js
function throwException(
  root: FiberRoot,
  returnFiber: Fiber | null,
  sourceFiber: Fiber,
  value: mixed,
  rootRenderLanes: Lanes
): boolean {
  // The source fiber did not complete.
  // 给这个fiber打上未完成的标记
  sourceFiber.flags |= Incomplete;

  if (value !== null && typeof value === "object") {
    if (typeof value.then === "function") {
      const wakeable: Wakeable = (value: any);
      // 标记最近的 Suspense 边界，切换到渲染 fallback。
      // 找到最近的suspense
      const suspenseBoundary = getSuspenseHandler();
      if (suspenseBoundary !== null) {
        switch (suspenseBoundary.tag) {
          case SuspenseComponent: {
            const current = suspenseBoundary.alternate;
            if (current === null) {
              // 渲染阶段，标记暂停渲染
              // workInProgressRootExitStatus = RootSuspended;
              renderDidSuspend();
            }
            suspenseBoundary.flags &= ~ForceClientRender;
            // 给suspense节点flags打上ShouldCapture
            markSuspenseBoundaryShouldCapture(
              suspenseBoundary,
              returnFiber,
              sourceFiber,
              root,
              rootRenderLanes
            );
            const isSuspenseyResource =
              wakeable === noopSuspenseyCommitThenable;
            if (isSuspenseyResource) {
              suspenseBoundary.flags |= ScheduleRetry;
            } else {
              const retryQueue: RetryQueue | null =
                (suspenseBoundary.updateQueue: any);
              if (retryQueue === null) {
                // 更新updateQueue，Set[promise]
                suspenseBoundary.updateQueue = new Set([wakeable]);
              } else {
                retryQueue.add(wakeable);
              }
              // 监听promise状态更改
              attachPingListener(root, wakeable, rootRenderLanes);
            }
            // 到这就结束了
            return false;
          }
        }
        throw new Error(
          `Unexpected Suspense handler tag (${suspenseBoundary.tag}). This ` +
            "is a bug in React."
        );
      } else {
        // 在并发根中，允许在没有 Suspense 的情况下暂停。它将无限期地暂停而不提交

        // 监听promise状态更改
        attachPingListener(root, wakeable, rootRenderLanes);
        renderDidSuspendDelayIfPossible();
        return false;
      }
    }
  }
}
```

##### markSuspenseBoundaryShouldCapture

```js
function markSuspenseBoundaryShouldCapture(
  suspenseBoundary: Fiber,
  returnFiber: Fiber | null,
  sourceFiber: Fiber,
  root: FiberRoot,
  rootRenderLanes: Lanes
): Fiber | null {
  // 给当前suspense打上标记
  suspenseBoundary.flags |= ShouldCapture;
  // 更新lanes
  suspenseBoundary.lanes = rootRenderLanes;
  return suspenseBoundary;
}
```

##### attachPingListener

```js
export function attachPingListener(
  root: FiberRoot,
  wakeable: Wakeable,
  lanes: Lanes,
) {
  // 数据可能会在我们提交 fallback 之前resolved。或者，在刷新的情况下，我们永远不会提交 fallback。
  // 因此，我们现在需要附加一个监听器。当它解析（"ping"）后，我们就可以决定是否再次尝试render树。
  //
  // 只有当我们当前正在渲染的lane还不存在监听器时，才会附加监听器（在这里监听器就像 "thread ID"）。
  //
  // 我们只在并发模式下这样做。
  let pingCache = root.pingCache;
  let threadIDs;
  if (pingCache === null) {
    // 无缓存
    // PossiblyWeakMap = WeakMap
    pingCache = root.pingCache = new PossiblyWeakMap();
    threadIDs = new Set<mixed>();
    // WeakMap {
    //   promise:Set()
    // }
    pingCache.set(wakeable, threadIDs);
  } else {
    threadIDs = pingCache.get(wakeable);
    if (threadIDs === undefined) {
      threadIDs = new Set();
      pingCache.set(wakeable, threadIDs);
    }
  }
  if (!threadIDs.has(lanes)) {
    workInProgressRootDidAttachPingListener = true;

    // 一个lanes对应一个listeners
    threadIDs.add(lanes);
    const ping = pingSuspendedRoot.bind(null, root, wakeable, lanes);
    // 给promise添加then
    wakeable.then(ping, ping);
  }
}
```

#### unwindUnitOfWork

```js
function unwindUnitOfWork(unitOfWork: Fiber): void {
  let incompleteWork: Fiber = unitOfWork;
  do {
    const current = incompleteWork.alternate;
    // 向上查找找到suspense
    const next = unwindWork(current, incompleteWork, entangledRenderLanes);

    if (next !== null) {
      // 找到了可以处理此异常的边界。重新进入beigin阶段。该分支将返回正常工作循环。
      //
      // 既然我们要重启，那就从flags中删除任何不是host effect的flag。
      // HostEffectMask是所有commit相关的flags集合
      next.flags &= HostEffectMask;
      workInProgress = next;
      return;
    }

    const returnFiber = incompleteWork.return;
    if (returnFiber !== null) {
      // 既然要重新开始那么，
      // 将父节点flags打上Incomplete，subtreeFlags，deletions置空
      returnFiber.flags |= Incomplete;
      returnFiber.subtreeFlags = NoFlags;
      returnFiber.deletions = null;
    }

    // 返回父节点
    incompleteWork = returnFiber;
    // 更新workInProgress
    workInProgress = incompleteWork;
  } while (incompleteWork !== null);

  // 更新workInProgressRootExitStatus
  workInProgressRootExitStatus = RootDidNotComplete;
  workInProgress = null;
}
```

##### unwindWork

> 找到 suspense

```js
// ...
case SuspenseComponent: {
  if (flags & ShouldCapture) {
    // 如果有ShouldCapture标记，去掉并打上DidCapture
    workInProgress.flags = (flags & ~ShouldCapture) | DidCapture;
    return workInProgress;
  }
}
// ...
```

##### 总结

- 正常渲染，渲染到 `suspense` 节点，赋值全局变量`suspenseHandlerStackCursor.current`，将子节点当成 `offscreen` 生成 `fiber`，然后继续渲染。
- 执行到函数组件中使用了`use`,`use`会将`promise`保存在全局变量中，然后`throw SuspenseException`
- 命中 `catch`，执行 `handleThrow`，通过各种条件判断设置`workInProgressSuspendedReason`值。
- 在`renderRootSync`中，通过判断`workInProgressSuspendedReason`执行`throwAndUnwindWorkLoop`
- `throwAndUnwindWorkLoop`中做了两件事
  - 执行`throwException`
    - `throwException`中找到最近的 `suspense` 给它打上 `ShouldCapture`，`updateQueue` 赋值为 `Set[promise]`，
    - 执行`attachPingListener`，缓存 `promise`,并且监听 `promise` 状态更改，
  - 执行 `unwindUnitOfWork`
    - 从当前的 `fiber`(函数组件)向上找到最近的 `suspense`，去掉`ShouldCapture`，打上`DidCapture`，将沿途的`fiber`都打上`Incomplete`，置空`subtreeFlags，deletions`，
    - 将`workInProgress`设置为 `suspense`，将 `workInProgressRootExitStatus` 设置为 `RootDidNotComplete`
- 结束，进入 `workLoop`，从`suspense`节点重新开始正常渲染阶段
- 从`beginWork`中再次回到`updateSuspenseComponent`，重新生成子节点 `fiber` 和 `fallback fiber`,将 `fallback fiber` 返回出去作为下一次 `beginWork` 处理的节点
- 经过一连串的 `render` 和 `commit` 阶段，渲染结束，此时 `fallback` 已经在页面上了，然后下一个 `eventLoop` 中`promise.then` 触发，也就是`pingSuspendedRoot`

##### pingSuspendedRoot

> 如何将 fallback 替换成正常子节点，很简单，就是调度一个更新

```js
function pingSuspendedRoot(
  root: FiberRoot,
  wakeable: Wakeable,
  pingedLanes: Lanes
) {
  // 拿到缓存，并删除它
  const pingCache = root.pingCache;
  if (pingCache !== null) {
    pingCache.delete(wakeable);
  }
  // 将lane合并到root.pingedLanes
  // 是将
  markRootPinged(root, pingedLanes);

  if (
    // 正常renderRootXxx结束之后workInProgressRoot会被清空,commit阶段开始前也会清空
    workInProgressRoot === root &&
    isSubsetOfLanes(workInProgressRootRenderLanes, pingedLanes)
  ) {
    // 收到与当前渲染优先级相同的 ping。
    // 我们可能需要重新启动渲染。

    // 如果是延迟暂停，或者是重试，我们会一直暂停，以便随时重启。
    if (
      workInProgressRootExitStatus === RootSuspendedWithDelay ||
      (workInProgressRootExitStatus === RootSuspended &&
        includesOnlyRetries(workInProgressRootRenderLanes) &&
        now() - globalMostRecentFallbackTime < FALLBACK_THROTTLE_MS)
    ) {
      // 通过释放堆栈，强制从根目录重新启动。除非这是从渲染阶段调用的，因为那样会导致崩溃。

      // 不在渲染阶段
      if ((executionContext & RenderContext) === NoContext) {
        // 放弃之前的渲染，重新生成wip
        prepareFreshStack(root, NoLanes);
      }
    } else {
      // 现在不能马上重启，合并pingedLanes进workInProgressRootPingedLanes，在之后可能有机会
      workInProgressRootPingedLanes = mergeLanes(
        workInProgressRootPingedLanes,
        pingedLanes
      );
    }
  }
  // 调度更新
  ensureRootIsScheduled(root);
}
```

### 更新阶段

> updateSuspenseComponent

```js
if (showFallback) {
  // 展示fallback
  pushFallbackTreeSuspenseHandler(workInProgress);

  const nextFallbackChildren = nextProps.fallback;
  const nextPrimaryChildren = nextProps.children;
  // 生成fallback fiber
  const fallbackChildFragment = updateSuspenseFallbackChildren(
    current,
    workInProgress,
    nextPrimaryChildren,
    nextFallbackChildren,
    renderLanes
  );
  const primaryChildFragment: Fiber = (workInProgress.child: any);
  const prevOffscreenState: OffscreenState | null = (current.child: any)
    .memoizedState;
  // 更新子节点memoizedState
  primaryChildFragment.memoizedState =
    prevOffscreenState === null
      ? mountSuspenseOffscreenState(renderLanes)
      : updateSuspenseOffscreenState(prevOffscreenState, renderLanes);
  primaryChildFragment.childLanes = getRemainingWorkInPrimaryTree(
    current,
    didPrimaryChildrenDefer,
    renderLanes
  );
  // 更新当前节点memoizedState
  workInProgress.memoizedState = SUSPENDED_MARKER;
  // 返回fallback fiber
  return fallbackChildFragment;
} else {
  // 展示子节点

  // 更新`suspenseHandlerStackCursor`，suspenseHandlerStackCursor会在completeWork阶段被清空
  pushPrimaryTreeSuspenseHandler(workInProgress);

  const nextPrimaryChildren = nextProps.children;
  // 生成子节点 fiber
  const primaryChildFragment = updateSuspensePrimaryChildren(
    current,
    workInProgress,
    nextPrimaryChildren,
    renderLanes
  );
  // 代表不再显示fallback
  workInProgress.memoizedState = null;
  // 返回子节点 fiber
  return primaryChildFragment;
}
```
