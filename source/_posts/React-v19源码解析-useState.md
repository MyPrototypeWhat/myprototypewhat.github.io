---
title: React@v19源码解析---useState
date: 2024-06-07 18:38:44
toc: true
tags: [React]
---

- [useState](#usestate)
  - [例子](#例子)
  - [mountState](#mountstate)
    - [mountStateImpl](#mountstateimpl)
      - [mountWorkInProgressHook](#mountworkinprogresshook)
    - [总结](#总结)
  - [updateState / updateReducer](#updatestate--updatereducer)
    - [updateWorkInProgressHook](#updateworkinprogresshook)
      - [总结](#总结-1)
  - [updateReducerImpl](#updatereducerimpl)
    - [总结](#总结-2)
  - [dispatchSetState](#dispatchsetstate)
    - [总结](#总结-3)
    - [requestUpdateLane](#requestupdatelane)

## 触发时机

`hooks` 都写在函数式组件内，所以肯定在执行组件的时候触发，也就是在 `render` 阶段，通过 `renderWithHooks` 函数执行函数式组件

```js
export function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes
): any {
  // ...
  renderLanes = nextRenderLanes;
  // 这个全局变量很关键，为当前执行的fiber
  currentlyRenderingFiber = workInProgress;
  // 清除新节点的三个属性
  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;
  workInProgress.lanes = NoLanes;
  // ...
  // 保存着所有hooks
  ReactSharedInternals.H =
    // 判断是 渲染还是更新，调用不同的dispatcher
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount
      : HooksDispatcherOnUpdate;
  // ...
  let children = Component(props, secondArg);
  // 清空相关变量和在dev环境下检查hooks是否正确执行
  finishRenderingHooks(current, workInProgress, Component);
  return children;
}
```

可以看到 `hooks` 在渲染阶段和更新阶段执行的不是同一个函数

```js
const HooksDispatcherOnMount: Dispatcher = {
  readContext,
  use,
  useCallback: mountCallback,
  useContext: readContext,
  useEffect: mountEffect,
  useImperativeHandle: mountImperativeHandle,
  useLayoutEffect: mountLayoutEffect,
  useInsertionEffect: mountInsertionEffect,
  useMemo: mountMemo,
  useReducer: mountReducer,
  useRef: mountRef,
  useState: mountState,
  useDebugValue: mountDebugValue,
  useDeferredValue: mountDeferredValue,
  useTransition: mountTransition,
  useSyncExternalStore: mountSyncExternalStore,
  useId: mountId,
};
const HooksDispatcherOnUpdate: Dispatcher = {
  readContext,
  use,
  useCallback: updateCallback,
  useContext: readContext,
  useEffect: updateEffect,
  useImperativeHandle: updateImperativeHandle,
  useInsertionEffect: updateInsertionEffect,
  useLayoutEffect: updateLayoutEffect,
  useMemo: updateMemo,
  useReducer: updateReducer,
  useRef: updateRef,
  useState: updateState,
  useDebugValue: updateDebugValue,
  useDeferredValue: updateDeferredValue,
  useTransition: updateTransition,
  useSyncExternalStore: updateSyncExternalStore,
  useId: updateId,
};
```

# useState

## 例子

```js
// 简单的例子

const [count, setCount] = useState(0);
// 点击事件
const handleClick = () => {
  setCount(count + 1);
};
```

## mountState

```js
function mountState<S>(
  initialState: (() => S) | S
): [S, Dispatch<BasicStateAction<S>>] {
  // 创建一个hook
  const hook = mountStateImpl(initialState);
  // 获取hook.queue
  const queue = hook.queue;
  // 创建dispatch函数
  const dispatch: Dispatch<BasicStateAction<S>> = (dispatchSetState.bind(
    null,
    currentlyRenderingFiber,
    queue
  ): any);
  queue.dispatch = dispatch;
  // 返回的就是例子中的 [count, setCount]
  return [hook.memoizedState, dispatch];
}
```

### mountStateImpl

```js
function mountStateImpl<S>(initialState: (() => S) | S): Hook {
  // 基本所有hook都要用到，用于生成hook对象
  const hook = mountWorkInProgressHook();
  if (typeof initialState === "function") {
    // 如果参数是个函数就执行一下拿到返回值
    const initialStateInitializer = initialState;
    initialState = initialStateInitializer();
  }
  // 赋值初始值
  hook.memoizedState = hook.baseState = initialState;
  // 创建hook.queue
  const queue: UpdateQueue<S, BasicStateAction<S>> = {
    pending: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  };
  hook.queue = queue;
  return hook;
}
```

#### mountWorkInProgressHook

> `mountWorkInProgressHook`函数基本每个 hook 都会调用，放在这讲一下

```js
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,

    baseState: null,
    baseQueue: null,
    queue: null,

    next: null,
  };

  if (workInProgressHook === null) {
    // currentlyRenderingFiber 为当前的函数式组件对应的fiber
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 注意不是环状链表
    workInProgressHook = workInProgressHook.next = hook;
  }
  // workInProgressHook是最新的hook
  return workInProgressHook;
}
```

### 总结

- 目前为止已经创建好了 `hook` 对象，`hook.queue` `用来存储更新队列，dispatch` 用来触发 `state` 的更改和调度更新
- 有个变量需要注意一下，容易混淆，
  - `fiber.memoizedState`存放的是`hook对象`，每个`hook语句`都是一个`hook`对象，通过执行顺序用`next`链接
  - `hook.memoizedState`存放的是`hook`相关的值，例如`useState`的`hook.memoizedState`是`state`的值,`useEffect`是`effect`对象

## updateState / updateReducer

```js
function updateState<S>(
  initialState: (() => S) | S
): [S, Dispatch<BasicStateAction<S>>] {
  // 简洁明了，调用updateReducer
  return updateReducer(basicStateReducer, initialState);
}

function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: (I) => S
): [S, Dispatch<A>] {
  // 获取当前hook，基本每个hook语句都需要用它来获取当前的hook
  const hook = updateWorkInProgressHook();
  // 执行hook.queue中挂的更新队列
  return updateReducerImpl(hook, ((currentHook: any): Hook), reducer);
}
```

### updateWorkInProgressHook

涉及到几个变量，说明一下

- `currentHook`和`workInProgressHook`全局变量，分别代表新旧节点的当前 `hook`，在`finishRenderingHooks`中都会被置为`null`
- `nextCurrentHook`和`nextWorkInProgressHook`局部变量，分别代表新旧节点的下一个 `hook`

```js
function updateWorkInProgressHook(): Hook {
  // 此函数用于 更新 和由 渲染阶段更新 触发的重新渲染。
  // 它假设有一个当前钩子可以克隆，或者有一个来自上一个渲染过程的正在进行的钩子可以用作基础。
  let nextCurrentHook: null | Hook;
  if (currentHook === null) {
    // 如果没有 例如这个hooks是组件内第一个hooks
    // 拿到current
    const current = currentlyRenderingFiber.alternate;
    if (current !== null) {
      // 更新阶段
      // 旧节点的memoizedState
      // wip.memoizedState已经被清空了，逻辑在renderWithHooks
      nextCurrentHook = current.memoizedState;
    } else {
      // 渲染阶段
      nextCurrentHook = null;
    }
  } else {
    // 说明不是第一次执行hook了，nextCurrentHook进一位
    nextCurrentHook = currentHook.next;
  }

  let nextWorkInProgressHook: null | Hook;
  if (workInProgressHook === null) {
    // workInProgress.memoizedState在函数组件每次渲染时都会被设置成null（在renderWithHooks中）
    // 同样说明这是第一次调用hook
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    // 说明hooks链表不为空，不是第一次执行hook了
    // nextWorkInProgressHook指向下一个hook
    nextWorkInProgressHook = workInProgressHook.next;
  }

  if (nextWorkInProgressHook !== null) {
    // 说明已有hook链表，复用它，并且有下一个hook
    // 进一位，保证顺序
    workInProgressHook = nextWorkInProgressHook;
    nextWorkInProgressHook = workInProgressHook.next;
    // 更新currentHook
    currentHook = nextCurrentHook;
  } else {
    // 从currentHook克隆出一个

    if (nextCurrentHook === null) {
      const currentFiber = currentlyRenderingFiber.alternate;
      if (currentFiber === null) {
        // 在初始渲染时调用了更新阶段的hook。这可能是React中的一个错误。请提交问题。
        throw new Error(
          "Update hook called on initial render. This is likely a bug in React. Please file an issue."
        );
      } else {
        // 渲染的hooks比上一次渲染时多。
        throw new Error("Rendered more hooks than during the previous render.");
      }
    }
    // 更新currentHook
    currentHook = nextCurrentHook;

    const newHook: Hook = {
      memoizedState: currentHook.memoizedState,

      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,

      next: null,
    };

    if (workInProgressHook === null) {
      // 更新当前fiber的memoizedState和当前hooks(workInProgressHook)
      currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
    } else {
      // 如果有就往后追加
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }
  return workInProgressHook;
}
```

#### 总结

函数看着比较长，其实就做了一件事情，将旧节点的 hook 拷贝出来，组成 workInProgressHook 链表

## updateReducerImpl

```js
function updateReducerImpl<S, A>(
  hook: Hook,
  current: Hook,
  reducer: (S, A) => S
): [S, Dispatch<A>] {
  const queue = hook.queue;

  if (queue === null) {
    throw new Error(
      "Should have a queue. This is likely a bug in React. Please file an issue."
    );
  }

  queue.lastRenderedReducer = reducer;
  // 拿到上一次没处理完的队列
  let baseQueue = hook.baseQueue;

  // 尚未处理的最后一个挂起的更新。
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    // 我们有新的更新尚未处理。我们将把它们添加到baseQueue中。
    if (baseQueue !== null) {
      // 环状链表，baseQueue和pendingQueue代表链表最后一位
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    // 更新baseQueue
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }

  const baseState = hook.baseState;
  if (baseQueue === null) {
    // 如果没有挂起的更新，则memoizedState应与baseState相同。
    // 目前，这些仅在useOptimistic的情况下出现分歧，因为useOptimistics在每个渲染上都接受一个新的baseState。
    hook.memoizedState = baseState;
  } else {
    // 有队列要处理
    const first = baseQueue.next;
    let newState = baseState;

    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast: Update<S, A> | null = null;
    let update = first;
    let didReadFromEntangledAsyncAction = false;
    do {
      const updateLane = removeLanes(update.lane, OffscreenLane);
      const isHiddenUpdate = updateLane !== update.lane;
      // 检查此更新是否是在隐藏树时进行的。
      // 如果是这样，那么这不是"base" update，我们应该忽略在进入屏幕外树时添加到renderLanes的额外基本车道。
      // 这下面段逻辑与processUpdateQueue一样
      const shouldSkipUpdate = isHiddenUpdate
        ? !isSubsetOfLanes(getWorkInProgressRootRenderLanes(), updateLane)
        : !isSubsetOfLanes(renderLanes, updateLane);

      if (shouldSkipUpdate) {
        // 优先级不足。跳过此更新。
        // 如果这是第一次跳过的更新，则先前的update/state为newBase update/state。
        const clone: Update<S, A> = {
          lane: updateLane,
          revertLane: update.revertLane,
          action: update.action,
          hasEagerState: update.hasEagerState,
          eagerState: update.eagerState,
          next: (null: any),
        };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }

        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane
        );
        markSkippedUpdateLanes(updateLane);
      } else {
        // 此更新具有足够的优先级。

        // 检查这是否是一个乐观的更新。
        const revertLane = update.revertLane;
        if (!enableAsyncActions || revertLane === NoLane) {
          // 这不是一个乐观的更新，我们现在就应用它。
          //
          // 但是，如果这个更新不需要被跳过但是之前的更新被跳过，
          // 我们就需要把这个更新留在队列中，以便稍后重新进行更新。所以lane:NoLane，保证下一次一定不会被跳过
          if (newBaseQueueLast !== null) {
            const clone: Update<S, A> = {
              lane: NoLane,
              revertLane: NoLane,
              action: update.action,
              hasEagerState: update.hasEagerState,
              eagerState: update.eagerState,
              next: (null: any),
            };
            newBaseQueueLast = newBaseQueueLast.next = clone;
          }

          // 检查此更新是否是挂起的异步操作的一部分。
          // 如果是这样，我们将需要挂起，直到操作完成，以便在同一操作中将其与未来的更新一起分批处理。
          if (updateLane === peekEntangledActionLane()) {
            didReadFromEntangledAsyncAction = true;
          }
        } else {
          // 这是一个乐观的更新。如果 "revert "优先级足够，就不要执行更新。
          // 否则，执行更新，但将其保留在队列中，以便在后续渲染中进行还原或重置。
          if (isSubsetOfLanes(renderLanes, revertLane)) {
            update = update.next;

            // 检查此更新是否属于待处理异步操作的一部分。
            // 如果是，我们就需要暂停，直到该操作完成，这样它就会与同一操作中的未来更新一起被批处理。
            if (revertLane === peekEntangledActionLane()) {
              didReadFromEntangledAsyncAction = true;
            }
            continue;
          } else {
            const clone: Update<S, A> = {
              lane: NoLane,
              // 重复使用相同的 revertLane，这样我们就能知道transition何时完成。
              revertLane: update.revertLane,
              action: update.action,
              hasEagerState: update.hasEagerState,
              eagerState: update.eagerState,
              next: (null: any),
            };
            if (newBaseQueueLast === null) {
              newBaseQueueFirst = newBaseQueueLast = clone;
              newBaseState = newState;
            } else {
              newBaseQueueLast = newBaseQueueLast.next = clone;
            }
            currentlyRenderingFiber.lanes = mergeLanes(
              currentlyRenderingFiber.lanes,
              revertLane
            );
            markSkippedUpdateLanes(revertLane);
          }
        }

        // Process this update.
        const action = update.action;
        if (update.hasEagerState) {
          // 是否已经算出newState了

          // 如果该更新是状态更新（而不是reducer），并且是急切处理的，我们可以使用急切计算的状态
          // 对于没有任务处理的fiber（fiber.lanes===NoLanes），
          // 会直接算出newState的值(同时赋给eagerState)，hasEagerState=true
          newState = ((update.eagerState: any): S);
        } else {
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    if (newBaseQueueLast === null) {
      // 没有更新
      newBaseState = newState;
    } else {
      // 重新形成环状链表
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }

    // 标记fiber已工作，但前提是 新状态 与 当前状态不同。
    if (!is(newState, hook.memoizedState)) {
      // 设置didReceiveUpdate=true 表示收到更新
      markWorkInProgressReceivedUpdate();

      // 检查此更新是否属于待执行的异步操作。
      // 如果是，我们就需要暂停，直到该操作完成，这样它就会与同一操作中的未来更新一起被批处理。
      if (didReadFromEntangledAsyncAction) {
        const entangledActionThenable = peekEntangledActionThenable();
        if (entangledActionThenable !== null) {
          // 将promise已错误的形式抛出去
          throw entangledActionThenable;
        }
      }
    }
    // 更新update/state
    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;

    queue.lastRenderedState = newState;
  }

  if (baseQueue === null) {
    queue.lanes = NoLanes;
  }

  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```

### 总结

函数比较长，但是主要的逻辑就是下面几步：

- 从 `hook` 中获取当前任务队列(`queue.pending`)，从 `hook` 中获取上一次更新的任务队列(`baseQueue`)
- 如果有遗留任务(`baseQueue`!==null)，将 `queue.pending` 挂在 `baseQueue`之后，
- 判断当前更新的优先级是否在`renderLanes`中来判断是否需要跳过当前更新，
  - 如果需要跳过就根据当前 `update` `clone` 出一个 update，放进`newBaseQueue`中
  - **如果不需要跳过当前更新**，且`newBaseQueue`中**没有**任务，则计算 `newState`
  - **如果不需要跳过当前更新**，**但是**`newBaseQueue`中**有**任务，也就是前面有低优先级的任务跳过了，那么这个本不该跳过的任务，将会被 `clone` 出一个 `update`，这个`update`的 lane 被赋值为`NoLane`，保证下次一定不会跳过。
    - 为什么前面有低优先级的后面的任务就要全部跳过呢？
    - 因为要保证 `update` 的顺序，保证 `state` 的正确性
- 判断 `state` 是否相等，如果不相等则`didReceiveUpdate = true`表示有更新，如果是个`Promise`则将它抛出
  - `didReceiveUpdate`会在 `updateComponents` 结尾判断，如果是`false`就会命中`bailout`策略，如果子节点也没有任务，直接终止。
- 更新任务队列和 `state`

## dispatchSetState

> setState 的实现

```js
function dispatchSetState<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A
): void {
  const lane = requestUpdateLane(fiber);

  const update: Update<S, A> = {
    lane,
    revertLane: NoLane,
    action,
    hasEagerState: false,
    eagerState: null,
    next: (null: any),
  };
  // 是否是渲染阶段的更新
  if (isRenderPhaseUpdate(fiber)) {
    enqueueRenderPhaseUpdate(queue, update);
  } else {
    const alternate = fiber.alternate;
    if (
      fiber.lanes === NoLanes &&
      (alternate === null || alternate.lanes === NoLanes)
    ) {
      // 如果当前没有任务，可以直接计算newState值，
      // 这是个优化手段，提前计算出newState值在下面进行is判断，如果相等可以提前退出

      const lastRenderedReducer = queue.lastRenderedReducer;
      if (lastRenderedReducer !== null) {
        let prevDispatcher = null;
        try {
          // 拿到上次已渲染的state，也就是旧的state
          const currentState: S = (queue.lastRenderedState: any);
          // action是函数就执行action(currentState）否则直接返回action
          const eagerState = lastRenderedReducer(currentState, action);

          update.hasEagerState = true;
          update.eagerState = eagerState;
          if (is(eagerState, currentState)) {
            // 快速路径。我们可以在不安排React重新渲染的情况下退出。
            // 如果组件由于不同的原因重新渲染，并且到那时reducer已经更改，那么我们稍后仍有可能需要重新调整此更新的基础。
            enqueueConcurrentHookUpdateAndEagerlyBailout(fiber, queue, update);
            return;
          }
        } catch (error) {
          // Suppress the error. It will throw again in the render phase.
        }
      }
      // 将update放进concurrentQueue中
      const root = enqueueConcurrentHookUpdate(fiber, queue, update, lane);
      if (root !== null) {
        // 调度一个更新
        scheduleUpdateOnFiber(root, fiber, lane);
        // transition相关
        entangleTransitionUpdate(root, queue, lane);
      }
    }
  }
}
```

### 总结

`dispatchSetState`比较好理解，就是创建一个 update，然后将 update 放进`concurrentQueue`，然后调度一个更新

- 如果两次 state 值一样，执行`enqueueConcurrentHookUpdateAndEagerlyBailout`，将 update 放进`concurrentQueue`并且将 update 排队，但是不调度更新
- 所以如果在上述例子里`setCount(0)`多次，并不会有任何重新渲染

### requestUpdateLane

> 获取更新优先级

```js
export function requestUpdateLane(fiber: Fiber): Lane {
  // Special cases
  const mode = fiber.mode;
  // 如果在并发模式下且disableLegacyMode为false，则返回SyncLane。
  if (!disableLegacyMode && (mode & ConcurrentMode) === NoMode) {
    return (SyncLane: Lane);
  } else if (
    (executionContext & RenderContext) !== NoContext &&
    workInProgressRootRenderLanes !== NoLanes
  ) {
    // 如果当前是渲染阶段的更新
    // 且workInProgressRootRenderLanes不为NoLanes，则返回当前渲染的lane中最高优先级。

    // This is a render phase update. These are not officially supported. The
    // old behavior is to give this the same "thread" (lanes) as
    // whatever is currently rendering. So if you call `setState` on a component
    // that happens later in the same render, it will flush. Ideally, we want to
    // remove the special case and treat them as if they came from an
    // interleaved event. Regardless, this pattern is not officially supported.
    // This behavior is only a fallback. The flag only exists until we can roll
    // out the setState warning, since existing code might accidentally rely on
    // the current behavior.
    return pickArbitraryLane(workInProgressRootRenderLanes);
  }
  // transition相关 useTransition再讲这部分
  const transition = requestCurrentTransition();
  if (transition !== null) {
    const actionScopeLane = peekEntangledActionLane();
    return actionScopeLane !== NoLane
      ? // 我们在一个异步操作范围内。重复使用相同的lane
        actionScopeLane
      : // 重新分配一个transition lane
        requestTransitionLane(transition);
  }
  // resolveUpdatePriority会从事件中获取优先级
  return eventPriorityToLane(resolveUpdatePriority());
}
```

`resolveUpdatePriority`首先会去`ReactDOMSharedInternals.p`中获取优先级，`React`会监听原生事件，当事件触发时会通过不同事件给`ReactDOMSharedInternals.p`赋值不同优先级
