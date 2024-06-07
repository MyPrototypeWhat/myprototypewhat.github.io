---
title: React@v19源码---Diff
toc: true
date: 2024-06-07 19:23:04
tags:
---

- [Diff](#diff)
  - [reconcileChildrenArray](#reconcilechildrenarray)
    - [总结](#总结)
    - [lastPlacedIndex](#lastplacedindex)

# Diff

常说的 diff 主要集中在多子节点的更新。

## reconcileChildrenArray

> 先看代码也行，先往下滑看解析也行

```js
function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null, // 第一个旧子节点
  newChildren: Array<any>,
  lanes: Lanes,
  debugInfo: ReactDebugInfo | null
): Fiber | null {
  // 新构建出来的fiber链表的头节点
  let resultingFirstChild: Fiber | null = null;
  // 新构建出来链表的最后那个fiber节点，用于构建整个链表
  let previousNewFiber: Fiber | null = null;
  // 旧节点的节点
  let oldFiber = currentFirstChild;
  // 表示当前已经新建的 Fiber 的 index 的最大值，用于判断是插入操作，还是移动操作等
  let lastPlacedIndex = 0;
  // 表示遍历 newChildren 的索引指针
  let newIdx = 0;
  // 下次循环要处理的fiber节点
  let nextOldFiber = null;
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (oldFiber.index > newIdx) {
      // 说明旧节点的位置在新节点的右边
      // oldIndex 大于 newIndex，那么需要旧的 fiber 等待新的 fiber，一直等到位置相同
      nextOldFiber = oldFiber;
      oldFiber = null;
    } else {
      // 说明旧节点的位置在新节点的左边
      nextOldFiber = oldFiber.sibling;
    }
    const newFiber = updateSlot(
      returnFiber,
      oldFiber,
      newChildren[newIdx],
      lanes,
      debugInfo
    );
    if (newFiber === null) {
      // 返回null说明无法复用也无法创建，结束循环
      if (oldFiber === null) {
        oldFiber = nextOldFiber;
      }
      break;
    }
    if (shouldTrackSideEffects) {
      // 更新流程
      if (oldFiber && newFiber.alternate === null) {
        // 说明新节点是新创建的，不是复用的，所以直接删除旧节点
        deleteChild(returnFiber, oldFiber);
      }
    }
    // 最后一个放置节点的索引（即最大的位置）
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    // 操作链表
    if (previousNewFiber === null) {
      // 头部
      resultingFirstChild = newFiber;
    } else {
      // previousNewFiber是尾部
      previousNewFiber.sibling = newFiber;
    }
    // 更新表尾
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }
  // 新节点遍历完了
  if (newIdx === newChildren.length) {
    // 将剩下的旧节点全删了
    deleteRemainingChildren(returnFiber, oldFiber);
    if (getIsHydrating()) {
      const numberOfForks = newIdx;
      pushTreeFork(returnFiber, numberOfForks);
    }
    // 返回结果
    return resultingFirstChild;
  }
  // 新节点没遍历完，旧节点遍历完了
  if (oldFiber === null) {
    // 遍历剩余的新节点，生成新fiber节点
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(
        returnFiber,
        newChildren[newIdx],
        lanes,
        debugInfo
      );
      if (newFiber === null) {
        continue;
      }
      // 打上Placement(插入)标记
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      // 操作链表
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      // 更新表尾
      previousNewFiber = newFiber;
    }
    if (getIsHydrating()) {
      const numberOfForks = newIdx;
      pushTreeFork(returnFiber, numberOfForks);
    }
    return resultingFirstChild;
  }

  // 将所有子节点添加到key map中，以便快速查找
  // key=>fiber 这种形式的映射
  const existingChildren = mapRemainingChildren(oldFiber);

  // 尝试使用key映射找到可复用的节点
  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = updateFromMap(
      existingChildren,
      returnFiber,
      newIdx,
      newChildren[newIdx],
      lanes,
      debugInfo
    );
    if (newFiber !== null) {
      if (shouldTrackSideEffects) {
        if (newFiber.alternate !== null) {
          // 是复用的就将映射中对应的那一组删掉
          existingChildren.delete(
            newFiber.key === null ? newIdx : newFiber.key
          );
        }
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      // 操作链表
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
  }

  if (shouldTrackSideEffects) {
    // 将映射表中没用到的旧节点全删了
    existingChildren.forEach((child) => deleteChild(returnFiber, child));
  }

  if (getIsHydrating()) {
    const numberOfForks = newIdx;
    pushTreeFork(returnFiber, numberOfForks);
  }
  return resultingFirstChild;
}
```

### 总结

流程分为几个阶段：

1. 先遍历
   1. 遍历新子节点,判断新旧节点 `index` 大小，更新下一轮要处理的节点(`nextOldFiber`)
   2. 执行`updateSlot`，创建`newChild`,根据 `newChildren[newIdx]` 的类型和`$$typeof`,判断新旧子节点 `key` 是否相同，不同则返回 `null`，进入下个阶段。
   3. 相同则复用旧子节点，判断`newChild`是否是新创建的节点，是则删除旧子节点。
   4. 更新 `lastPlacedIndex`（初始为 `0`），代表最后一个放置节点的索引（即最大的位置）。
   5. 更新 `previousNewFiber。`
2. 新子节点遍历完了，代表剩下的旧子节点可以全删了。
3. 旧子节点遍历完了，代表剩下的新子节点全部需要重新创建
4. 新旧都没有遍历完，因为`阶段 1` 的某些原因提前结束了遍历（比如 `key` 不相等）。
   1. 将剩余旧子节点的 `key` 生成映射（`key=>fiber`，没`key`就用`index`）
   2. 遍历剩余`newChildren`，在映射中找 `key` 相同的节点（没 `key` 就用 `index`），有就复用，没有就创建。
   3. 将用过的 `key` 在映射中删掉。
   4. 更新 `lastPlacedIndex`
   5. 遍历完之后将映射中没用到的 `key` 对应的 `fiber` 全删了

需要注意的是，`newChildren` 是数组，数组元素是 `element` 对象，不是 `fiber`，也就是说这是一个 `for` 循环遍历和 `oldFiber.sibling` 链表的遍历

### lastPlacedIndex

> 代表最后一个放置节点的索引（即最大的位置）
> 中心思想就是，fiber 的位置能不动就不动，实在没办法再插入

```js
function placeChild(
  newFiber: Fiber,
  lastPlacedIndex: number,
  newIndex: number
): number {
  // 修正为新节点的index
  newFiber.index = newIndex;
  const current = newFiber.alternate;
  if (current !== null) {
    const oldIndex = current.index;
    if (oldIndex < lastPlacedIndex) {
      // 移动
      newFiber.flags |= Placement | PlacementDEV;
      return lastPlacedIndex;
    } else {
      // 保持原位
      return oldIndex;
    }
  } else {
    // 插入
    newFiber.flags |= Placement | PlacementDEV;
    return lastPlacedIndex;
  }
}
```

举个例子：

```
旧子节点：A => B => C => D
新子节点：A => C => D => B
字母代表key
```

1. 首先遍历新子节点，`A` 可以复用
2. `B C` `key` 不同，不能复用，跳出循环
3. 新旧子节点都没有遍历完，生成映射
4. 遍历剩余的新子节点，`C D` `key` `相同，复用，lastPlacedIndex` 依次更新为 `2，3`
5. 遍历 `B`，在旧节点中 `B` 的索引为 `1`，小于 `lastPlacedIndex`，所以需要移动，将新子节点 `fiber` 打上 `Placement`
