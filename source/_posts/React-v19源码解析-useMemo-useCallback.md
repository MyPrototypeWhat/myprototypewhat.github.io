---
title: React@v19源码解析---useMemo/useCallback
toc: true
tags:
  - React
date: 2024-06-07 19:50:10
---

# useMemo

> 很简单，直接看源码

## mountMemo / updateMemo

```js
function mountMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null
): T {
  // 创建hook对象
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

function updateMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null
): T {
  // 找到hook对象
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (nextDeps !== null) {
    const prevDeps: Array<mixed> | null = prevState[1];
    // 比较依赖是否更改
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0];
    }
  }
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

### 总结

可以看到有一些共同处：执行回调，拿到返回值,将**返回值**和**依赖**组成数组赋值 `hook.memoizedState`，注意一点，从上面代码可以看出，如果依赖没有更改，并不会执行回调，而是直接返回之前的值。

# useCallback

## mountCallback / updateCallback

```js
function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  hook.memoizedState = [callback, nextDeps];
  return callback;
}

function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (nextDeps !== null) {
    const prevDeps: Array<mixed> | null = prevState[1];
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      // 如果依赖没有更改，直接返回之前的函数
      return prevState[0];
    }
  }
  // 如果有更改，会重新将函数挂在memoizedState
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

### 总结

跟 `useMemo` 差不多，只是返回值不一样，`useMemo` `返回的是函数返回值，useCallback` 返回的是函数本身。都是挂载到 `hook.memoizedState` 上。
需要注意 `useCallback` 常被诟病的闭包问题就出现在这，如果依赖没有发生改变，那么函数就不会重新挂载
