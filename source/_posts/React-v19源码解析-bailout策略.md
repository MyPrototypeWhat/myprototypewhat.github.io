---
title: React@v19源码解析---bailout策略
toc: true
tags:
  - React
date: 2024-06-15 18:48:30
---

# Bailout 策略

> 是`React`内部优化的手段之一，提前命中`bailout`策略可以减少没必要的`Re-render`
> 之前被面试官问到了，也不知道有没有给她说清楚，重新梳理一下

## 命中条件

### beginWork

```js
if (
  // 如果新旧props不一致
  oldProps !== newProps ||
  // 因为全局变量disableLegacyContext，始终返回false
  hasLegacyContextChanged()
) {
  didReceiveUpdate = true;
} else {
  // props没有改变。检查是否有挂起的update或context更改。
  const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
    current,
    renderLanes
  );
  if (
    !hasScheduledUpdateOrContext &&
    // 如果这是第二次通过错误或挂起边界，则可能没有在“current”上安排工作，因此我们检查此标志。
    (workInProgress.flags & DidCapture) === NoFlags
  ) {
    didReceiveUpdate = false;
    // 如果没有计划更新，尝试提前退出
    // 这里要么返回null，提前进入completeWork阶段，要么返回子节点继续渲染
    return attemptEarlyBailoutIfNoScheduledUpdate(
      current,
      workInProgress,
      renderLanes
    );
  }
}
```

`条件1`：在`update`阶段，首先判断新旧 `props` 是否有变化，这个地方可以解释为什么父组件 `setState` 后，子组件都会重新渲染，因为新旧`props`不等。

- 生成 fiber 大致分为两种情况，一个是创建 `fiber`，一个是复用 `fiber`。
- **所以只有当父组件命中 `bailout` 策略之后**，才会复用 `fiber`，然后子组件的新旧 `props` 才会`===`
- 如果这里的思路捋不顺，可以先往下看。

#### checkScheduledUpdateOrContext

```js
function checkScheduledUpdateOrContext(
  current: Fiber,
  renderLanes: Lanes
): boolean {
  // 在命中bailout之前，需要检查是否有更新
  const updateLanes = current.lanes;
  // 判断旧fiber节点中是否有renderLanes
  if (includesSomeLane(updateLanes, renderLanes)) {
    return true;
  }
  return false;
}
```

`条件2`：`hasScheduledUpdateOrContext`为 false，并且没有 `DidCapture` 标记（`Suspense` 相关，想了解的可以翻翻我之前的文章）意味着当前 `fiber` 没有更新。

##### 总结

**到目前为止都是检测自身的 fiber 是否有更新**，接下来是判断子节点是否有更新，有则复用旧子节点，创建新子节点。无则终止流程。

#### attemptEarlyBailoutIfNoScheduledUpdate/bailoutOnAlreadyFinishedWork

```js
function attemptEarlyBailoutIfNoScheduledUpdate(
  current: Fiber,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  // 主要是做一些记录，
  // 就SuspenseComponent特殊一些，会判断子节点是否有更新，如果有就进入updateSuspenseComponent
  // 没有就直接执行bailoutOnAlreadyFinishedWork判断fallback是否有更新
  switch (
    workInProgress.tag
    //...
  ) {
  }
  return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
}

function bailoutOnAlreadyFinishedWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  if (current !== null) {
    // 重用以前的依赖项
    workInProgress.dependencies = current.dependencies;
  }
  // 标记当前Fiber节点的lanes，表示其更新已被跳过
  markSkippedUpdateLanes(workInProgress.lanes);

  // 检查子节点是否有待处理的工作。如果没有，则可以跳过子节点的渲染。
  if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
    return null;
  }

  // 当前节点没有更新，但其子节点有。克隆子节点并继续
  // 同样也会克隆兄弟节点（如果有的话）
  // 克隆完成之后将第一个子节点挂在workInProgress.child
  cloneChildFibers(current, workInProgress);
  // 返回子节点
  return workInProgress.child;
}
```

##### 总结

如果子节点也没有更新则直接返回 null，那么在`performUnitOfWork`中，`next`变量为`beginWork`返回值，`next===null`会进入 `compeletWork` 阶段，如果子节点有更新则克隆子节点，然后返回第一个子节点，继续 `beginWork`。

**那么如何判断子节点是否有更新呢？**

- `includesSomeLane(renderLanes, workInProgress.childLanes)`
- `childLanes`是所有子节点的 `lanes` 总和
- `renderLanes`是什么呢？
  - 在`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`中通过`getNextLanes`中计算出的`lanes`，通过参数传递给`renderRootXxx`，通过参数传递给`prepareFreshStack`，通过`getEntangledLanes`将`entangledRenderLanes`赋值为`lanes`，`performUnitOfWork`中将`entangledRenderLanes`作为参数传给了`beginWork`。

看到这第一次 `bailout` 判断就结束了，在组件内还会有一次 `bailout` 判断，以函数组件举例

### updateFunctionComponent

```js
function updateFunctionComponent(
  current: null | Fiber,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes
) {
  let context;

  let nextChildren;
  let hasId;
  // 遍历workInProgress.dependencies
  // workInProgress.dependencies保存的是这个组件使用的context
  // 如果有context，并且这个context有更新了，则didReceiveUpdate=true
  prepareToReadContext(workInProgress, renderLanes);
  // 执行函数组件
  // didReceiveUpdate可能会变为true,例如
  // useState中会计算新旧state是否相同，不同则didReceiveUpdate=true
  nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,
    nextProps,
    context,
    renderLanes
  );
  hasId = checkDidRenderIdHook();
  // 更新阶段 && didReceiveUpdate为false
  if (current !== null && !didReceiveUpdate) {
    // 命中bailout
    bailoutHooks(current, workInProgress, renderLanes);
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }

  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

#### bailoutHooks

```js
export function bailoutHooks(
  current: Fiber,
  workInProgress: Fiber,
  lanes: Lanes
): void {
  // 复用旧节点updateQueue
  workInProgress.updateQueue = current.updateQueue;
  // 去掉副作用和更新标记
  workInProgress.flags &= ~(PassiveEffect | UpdateEffect);
  // 移除对应lanes
  current.lanes = removeLanes(current.lanes, lanes);
}
```

#### 总结

如果这个 `fiber props` 没有更改，并且有更新（`fiber.lanes` 有值），可以暂时认为它不需要更新`didReceiveUpdate=false`，然后在`updateFunctionComponent`中检查 `context` 是否有更改，执行函数组件，检查相关`hooks`是否有收到更新

- 如果都没有就命中`bailout`
