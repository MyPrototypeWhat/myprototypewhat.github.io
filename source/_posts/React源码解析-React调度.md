---
title: React源码解析---React调度
toc: true
tags:
  - React
date: 2024-06-07 19:56:38
---

> React@v19

# 前言

这章会详细讲解：

- 如何调度一个渲染任务
- 不包含`Scheduler`相关代码

# scheduleUpdateOnFiber

```js
export function scheduleUpdateOnFiber(
  root: FiberRoot,
  fiber: Fiber,
  lane: Lane
) {
  if (
    // 当前调度的更新在当前正在处理的根节点上
    (root === workInProgressRoot &&
      // 并且正在等待数据返回
      workInProgressSuspendedReason === SuspendedOnData) ||
    root.cancelPendingCommit !== null
  ) {
    // 传入的更新可能会解除当前渲染的阻塞，中断当前尝试，从头开始。
    // 重新生成workInProgress树
    prepareFreshStack(root, NoLanes);
    // 会将当前lanes合并到root.suspendedLanes中
    markRootSuspended(
      root,
      workInProgressRootRenderLanes,
      workInProgressDeferredLane
    );
  }

  // 标记根具有挂起的更新。
  // fiberRootNode.pendingLanes |= lane
  // 只存在FiberRootNode上，记录的是所有（因为可能有多根的情况，这里指所有） 待更新的lanes
  markRootUpdated(root, lane);
  // 如果更新是在渲染阶段调度的，则给出警告，并跟踪在渲染阶段更新的lanes。
  if (
    (executionContext & RenderContext) !== NoLanes &&
    root === workInProgressRoot
  ) {
    warnAboutRenderPhaseUpdatesInDEV(fiber);
    workInProgressRootRenderPhaseUpdatedLanes = mergeLanes(
      workInProgressRootRenderPhaseUpdatedLanes,
      lane
    );
  }
  // 否则，这是一个正常的更新。
  else {
    if (root === workInProgressRoot) {
      // 不在render阶段
      if ((executionContext & RenderContext) === NoContext) {
        workInProgressRootInterleavedUpdatedLanes = mergeLanes(
          workInProgressRootInterleavedUpdatedLanes,
          lane
        );
      }
      if (workInProgressRootExitStatus === RootSuspendedWithDelay) {
        markRootSuspended(
          root,
          workInProgressRootRenderLanes,
          workInProgressDeferredLane
        );
      }
    }
    // 调度一个更新
    ensureRootIsScheduled(root);
  }
}
```

## 总结

> 这个函数有点长，总结一下

核心目标是协调 React 组件的更新流程，**确保**它们在正确的时机被调度和处理，同时提供必要的调试信息和错误检查。
如果是只**从首次渲染角度来看**，所有的判断都没什么用，**就执行了两个函数**`markRootUpdated`和`ensureRootIsScheduled`

## ensureRootIsScheduled

```js
export function ensureRootIsScheduled(root: FiberRoot): void {
  // 每当根收到更新时，都会调用此函数。它做两件事：
  // 1）确保根在根计划中，
  // 2）确保有一个挂起的微任务来处理根计划。
  if (root === lastScheduledRoot || root.next !== null) {
    // Fast path.
    // 这个root已经被调度
  } else {
    // 老套路了 将需要调度的根串成环状链表
    // lastScheduledRoot firstScheduledRoot 链表 lastScheduledRoot为最新的root
    // 一般情况只有一个根，firstScheduledRoot = lastScheduledRoot = root;
    // 多根时为链表
    if (lastScheduledRoot === null) {
      firstScheduledRoot = lastScheduledRoot = root;
    } else {
      lastScheduledRoot.next = root;
      lastScheduledRoot = root;
    }
  }

  // 每当根收到更新时，我们都会将其设置为true，直到下一次处理调度为止。
  // 如果它是false，那么我们可以快速退出flushSync
  mightHavePendingSyncWork = true;
  // 判断是否当前已经有一个微任务在调度了
  if (!didScheduleMicrotask) {
    didScheduleMicrotask = true;
    // 在微任务中执行根节点的调度
    scheduleImmediateTask(processRootScheduleInMicrotask);
  }
}
```

### scheduleImmediateTask

```js
function scheduleImmediateTask(cb: () => mixed) {
  // 将回调放入微任务中
  scheduleMicrotask(() => {
    // 在 Safari 中，添加 iframe 会强制将目前的微任务同步执行（早已修复）
    // https://github.com/facebook/react/issues/22459
    // 我们不支持在render或commit过程中运行回调，因此我们需要检查这一点。
    const executionContext = getExecutionContext();
    if ((executionContext & (RenderContext | CommitContext)) !== NoContext) {
      // 在render或commit阶段中
      // 使用宏任务调度
      Scheduler_scheduleCallback(ImmediateSchedulerPriority, cb);
      return;
    }
    // 直接在微任务中执行
    // 从上文来看cb就是processRootScheduleInMicrotask
    cb();
  });
}
```

### processRootScheduleInMicrotask

```js
function processRootScheduleInMicrotask() {
  didScheduleMicrotask = false;
  mightHavePendingSyncWork = false;

  const currentTime = now();

  let prev = null;
  let root = firstScheduledRoot;
  while (root !== null) {
    const next = root.next;
    // transition相关
    if (
      currentEventTransitionLane !== NoLane &&
      shouldAttemptEagerTransition()
    ) {
      upgradePendingLaneToSync(root, currentEventTransitionLane);
    }
    // 给root调度任务
    const nextLanes = scheduleTaskForRootDuringMicrotask(root, currentTime);
    if (nextLanes === NoLane) {
      // 表示这个根已经没有待处理的任务，将它从链表中移除
      root.next = null;
      if (prev === null) {
        // 新的表头
        firstScheduledRoot = next;
      } else {
        prev.next = next;
      }
      if (next === null) {
        // 新的表尾
        lastScheduledRoot = prev;
      }
    } else {
      // This root still has work. Keep it in the list.
      prev = root;
      if (includesSyncLane(nextLanes)) {
        // 如果有同步任务，则设置标志
        mightHavePendingSyncWork = true;
      }
    }
    root = next;
  }

  currentEventTransitionLane = NoLane;

  // 在微任务结束时，清除所有待处理的同步工作。
  // 这必须在任务结束时进行，因为实际的渲染工作可能会中断。
  flushSyncWorkOnAllRoots();
}
```

#### scheduleTaskForRootDuringMicrotask

```js
export function flushSyncWorkOnAllRoots() {
  flushSyncWorkAcrossRoots_impl(false);
}

function scheduleTaskForRootDuringMicrotask(
  root: FiberRoot,
  currentTime: number
): Lane {
  // 遍历 pendingLanes，检查是否已到过期时间。
  // 如果是，我们就会假设更新正处于饥饿状态，并将其标记为过期，以强制其完成更新（后续）。
  markStarvedLanesAsExpired(root, currentTime);

  // 获取workInProgressRoot全局变量
  const workInProgressRoot = getWorkInProgressRoot();
  // 首次渲染 workInProgressRootRenderLanes === 0
  const workInProgressRootRenderLanes = getWorkInProgressRootRenderLanes();
  // 重点！！！
  // 计算这个渲染周期应该执行什么任务
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
  );

  const existingCallbackNode = root.callbackNode;
  if (
    // 没任务要做
    nextLanes === NoLanes ||
    // 如果此根当前处于suspended状态，正在等待数据解析，
    // 请不要安排任务来render。我们将等待ping，或者等待接收更新。
    // Suspended render phase
    (root === workInProgressRoot && isWorkLoopSuspendedOnData()) ||
    // Suspended commit phase
    root.cancelPendingCommit !== null
  ) {
    // 如果没有任务或者处于suspended
    // 取消这次任务调度
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
    }
    root.callbackNode = null;
    root.callbackPriority = NoLane;
    return NoLane;
  }

  if (includesSyncLane(nextLanes)) {
    // 同步工作总是在微任务结束时刷新，因此我们不需要安排额外的任务。
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
    }
    root.callbackPriority = SyncLane;
    root.callbackNode = null;
    return SyncLane;
  } else {
    // 我们使用优先级最高的车道来表示回调的优先级。
    // callbackPriority表示当前任务优先级
    const existingCallbackPriority = root.callbackPriority;
    // 找到最高优先级
    const newCallbackPriority = getHighestPriorityLane(nextLanes);

    if (newCallbackPriority === existingCallbackPriority) {
      // 优先级没改变 复用现有任务
      return newCallbackPriority;
    } else {
      // 取消现有的任务，在下面重新调度一个
      cancelCallback(existingCallbackNode);
    }
    // lane转scheduler优先级
    let schedulerPriorityLevel;
    switch (lanesToEventPriority(nextLanes)) {
      case DiscreteEventPriority:
        schedulerPriorityLevel = ImmediateSchedulerPriority;
        break;
      case ContinuousEventPriority:
        schedulerPriorityLevel = UserBlockingSchedulerPriority;
        break;
      case DefaultEventPriority:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
      case IdleEventPriority:
        schedulerPriorityLevel = IdleSchedulerPriority;
        break;
      default:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
    }
    // 将performConcurrentWorkOnRoot放入调度队列，messageChannel触发
    const newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      // 进入render阶段
      performConcurrentWorkOnRoot.bind(null, root)
    );
    // 更新任务优先级
    root.callbackPriority = newCallbackPriority;
    // 存放 scheduleCallback返回的task对象
    root.callbackNode = newCallbackNode;
    return newCallbackPriority;
  }
}
```

#### flushSyncWorkOnAllRoots

```js
function flushSyncWorkAcrossRoots_impl(onlyLegacy: boolean) {
  if (isFlushingWork) {
    // 防止重复进入
    return;
  }
  if (!mightHavePendingSyncWork) {
    // 代表是否有待处理的同步任务
    return;
  }

  // 可能有同步任务也可能没有，需要检查一下
  let didPerformSomeWork;
  // 表示同步工作已经开始。
  isFlushingWork = true;
  do {
    // 表示当前循环尚未执行任何同步工作。
    didPerformSomeWork = false;
    let root = firstScheduledRoot;
    // 有可能多根，所以要循环
    while (root !== null) {
      if (onlyLegacy && (disableLegacyMode || root.tag !== LegacyRoot)) {
        // Skip non-legacy roots.
      } else {
        // workInProgressRoot为null时代表没有渲染的根
        const workInProgressRoot = getWorkInProgressRoot();
        const workInProgressRootRenderLanes =
          getWorkInProgressRootRenderLanes();
        const nextLanes = getNextLanes(
          root,
          root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
        );
        if (includesSyncLane(nextLanes)) {
          // 还有同步的任务，立刻执行
          didPerformSomeWork = true;
          performSyncWorkOnRoot(root, nextLanes);
        }
      }
      root = root.next;
    }
    // 有同步任务就继续执行
  } while (didPerformSomeWork);
  isFlushingWork = false;
}
```

##### markStarvedLanesAsExpired

> 函数内有些 lanes 会在后面解释，先往下看

```js
export function markStarvedLanesAsExpired(
  root: FiberRoot,
  currentTime: number
): void {
  // 待处理车道的总和
  const pendingLanes = root.pendingLanes;
  // 挂起的车道总和
  const suspendedLanes = root.suspendedLanes;
  // 被ping的车道总和
  const pingedLanes = root.pingedLanes;
  // expirationTimes 初始值为Array(32).fill(NoTimestamp)
  const expirationTimes = root.expirationTimes;

  // 排除重试车道
  let lanes = pendingLanes & ~RetryLanes;
  while (lanes > 0) {
    // 获得有多少个有效位
    const index = pickArbitraryLaneIndex(lanes);
    // 1向左移动index位，也就是1*(2的index次方)
    // 这个操作也就是取lanes中最低优先级
    const lane = 1 << index;

    const expirationTime = expirationTimes[index];
    if (expirationTime === NoTimestamp) {
      // 如果没有过期
      if (
        (lane & suspendedLanes) === NoLanes ||
        (lane & pingedLanes) !== NoLanes
      ) {
        // 如果这个lane不是suspendedLanes又不是pingedLanes
        // 则使用当前时间重新计算新的过期时间。
        expirationTimes[index] = computeExpirationTime(lane, currentTime);
      }
    } else if (expirationTime <= currentTime) {
      // 过期就将这个lane合并到expiredLanes
      root.expiredLanes |= lane;
    }
    // 移除这个优先级继续遍历
    lanes &= ~lane;
  }
}
```

###### pendingLanes

> pendingLanes 是 root 的属性，也就是等待处理的车道总和。

###### suspendedLanes

> 挂起车道总和

大概分为两种情况

- 一种是遇到了报错，或者某些异常情况，会执行`markRootSuspended`将当前 lanes 合并到`root.suspendedLanes`，以便后面重试
- 另一种是用`Suspense`组件

###### pingedLanes

> 被 pinged 的车道总和与`suspendedLanes`相关

- 被`Suspense`组件包裹的子组件一般是会产生一个`Promise` 例如`React.lazy`会将`Promise`以错误的形式抛出，通过`handleThrow`处理将`workInProgressSuspendedReason`设置为`SuspendedOnData`然后通过`throwException`中执行`attachPingListener`监听`Promise`的状态更改，如果`resolve`了就把这次的 `lanes` 合并进`pingedLanes`

###### pickArbitraryLaneIndex

> 这个函数后面经常用到

```js
function pickArbitraryLaneIndex(lanes: Lanes) {
  // clz32是Math.clz32()
  // 给定数字的32位二进制表示中前导零位的数量。
  // 例如：
  // Math.clz32(2) 值为 30 2转二进制为 0010，前面有30个0
  return 31 - clz32(lanes);
}
```

##### getNextLanes

> 重点！！！！，它负责如何从一堆任务中挑出一批任务在这个周期执行

```js
export function getNextLanes(root: FiberRoot, wipLanes: Lanes): Lanes {
  // pendingLanes代表当前待处理的任务队列优先级
  const pendingLanes = root.pendingLanes;
  if (pendingLanes === NoLanes) {
    // 表示当前没有待处理的任务
    return NoLanes;
  }

  let nextLanes: Lanes = NoLanes;

  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;

  // 在所有非闲置工作完成之前，即使工作挂起了，也不要进行任何闲置工作。

  // 从总任务中取非闲置的任务，大部分任务都属于非闲置
  const nonIdlePendingLanes = pendingLanes & NonIdleLanes;
  if (nonIdlePendingLanes !== NoLanes) {
    // 如果有，再把suspendedLanes排除掉
    const nonIdleUnblockedLanes = nonIdlePendingLanes & ~suspendedLanes;
    if (nonIdleUnblockedLanes !== NoLanes) {
      // 现在只剩下非闲置且没有挂起的任务

      // 取最高优先级
      // 先判断任务是否是同步更新（SyncLane | InputContinuousLane | DefaultLane），
      // 如果有就从当中取
      // 否则取最右的任务队列，也就是最高优先级的车道
      nextLanes = getHighestPriorityLanes(nonIdleUnblockedLanes);
    } else {
      // 现在只剩下非闲置且挂起的任务
      // 从suspendedLanes中找出已经ping的lanes
      const nonIdlePingedLanes = nonIdlePendingLanes & pingedLanes;
      if (nonIdlePingedLanes !== NoLanes) {
        // 跟上面同样的操作
        nextLanes = getHighestPriorityLanes(nonIdlePingedLanes);
      }
    }
  } else {
    // 只剩闲置任务了
    // 排除掉挂起的，就是闲置且非挂起
    const unblockedLanes = pendingLanes & ~suspendedLanes;
    // 下面操作就跟上面一样了
    if (unblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(unblockedLanes);
    } else {
      // 闲置且挂起的任务
      if (pingedLanes !== NoLanes) {
        nextLanes = getHighestPriorityLanes(pingedLanes);
      }
    }
  }

  if (nextLanes === NoLanes) {
    // 只有suspended会走到这里
    return NoLanes;
  }

  // 如果当前有正在进行的任务（wipLanes !== NoLanes），并且下一个任务队列（nextLanes）与之不同
  // 函数会进一步比较它们的优先级。
  if (
    wipLanes !== NoLanes &&
    wipLanes !== nextLanes &&
    // 且没有挂起的任务
    (wipLanes & suspendedLanes) === NoLanes
  ) {
    // 取各自最高优先级
    const nextLane = getHighestPriorityLane(nextLanes);
    const wipLane = getHighestPriorityLane(wipLanes);
    if (
      // 测试下一条车道的优先级是否等于或低于上一条车道。
      // 这样做是因为位的优先级随着向左移动而降低。
      nextLane >= wipLane ||
      // 默认优先级更新不应中断过渡更新。
      // 默认更新与过渡更新的唯一区别是，默认更新不支持refresh transitions。
      (nextLane === DefaultLane && (wipLane & TransitionLanes) !== NoLanes)
    ) {
      // 如果下一个任务的优先级不高于当前任务
      // 或者当前任务包含过渡任务并且下一个任务是默认任务，函数将选择继续执行当前任务（wipLanes）
      // 继续处理现有的进度树。不要打断。
      return wipLanes;
    }
  }

  return nextLanes;
}
```

有点长，总结一下流程：

- `pendingLanes`没有任务就返回`NoLanes`
- 有非闲置任务
  - 如果有非挂起的，取最高优先级
  - 如果只有挂起的，取 `suspendedLanes` 和 `pingedLanes` 交集中取最高优先级
- 只有闲置任务
  - 如果有非挂起的，取最高优先级
  - 如果只有挂起的，从 `pingedLanes` 中取最高优先级
- 如果算出的`lanes`为空则返回空
- 如果当前有正在进行的`lanes`，则与之相比
  - 如果下一次任务的优先级等于或当前任务优先级，或者当前下一次任务优先级为默认优先级 且 当前任务优先级中包含过渡任务，则返回当前任务优先级
  - 如果算出的任务优先级和当前任务优先级一样就不会取消当前任务
- 否则返回`nextLanes`

## 总结

- `scheduleUpdateOnFiber` 确保任务在正确的时机被调度
  - 如果当前有正在渲染的任务且有挂起的更新，则重新生成`wip`且从头开始
  - 将`lane`合并到`root.pendingLanes`,
  - 将渲染阶段的更新给予警告
  - 执行`ensureRootIsScheduled`
- `ensureRootIsScheduled` 将根组成一个链表，如果有多根的情况下可以保证更新的顺序，将`processRootScheduleInMicrotask`放进微任务中
- `processRootScheduleInMicrotask` 遍历调度根节点。执行`scheduleTaskForRootDuringMicrotask`
  - `scheduleTaskForRootDuringMicrotask`给 root 调度任务
    - 执行`markStarvedLanesAsExpired`将过期的任务标记为过期，将未过期的任务重新计算过期时间
    - 执行`getNextLanes`计算这个渲染周期执行什么任务
    - 如果没任务做 或者 （根在渲染中并且等待 ping）取消任务调度，返回`NoLane`
    - 如果有同步任务，就取消已有任务，因为微任务会在同步任务之前执行，不需要重新调度一个任务，返回`SyncLane`
    - 调度出一个新任务（`performConcurrentWorkOnRoot`）
  - 执行`flushSyncWorkOnAllRoots`，冲刷同步任务
