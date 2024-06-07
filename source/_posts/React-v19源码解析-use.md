---
title: React@v19源码解析---use
toc: true
tags:
  - React
date: 2024-06-07 19:55:01
---

# use

> `use` 是一个 `React Hook`，它可以让你读取类似于 `Promise` 或 `context` 的资源的值。

```js
function use<T>(usable: Usable<T>): T {
  // 参数得是个对象
  if (usable !== null && typeof usable === "object") {
    if (typeof usable.then === "function") {
      // promise对象或者类promise对象
      const thenable: Thenable<T> = (usable: any);
      return useThenable(thenable);
      // context对象
    } else if (usable.$$typeof === REACT_CONTEXT_TYPE) {
      const context: ReactContext<T> = (usable: any);
      // context相关逻辑，可以看下我之前的useContext源码分析
      return readContext(context);
    }
  }
  // 向 use() 传递了不支持的类型
  throw new Error("An unsupported type was passed to use(): " + String(usable));
}
```

可以看到 use 相当于调用的是`useThenable`or`readContext`

## useThenable

```js
function useThenable<T>(thenable: Thenable<T>): T {
  // Track the position of the thenable within this fiber.
  // thenableIndexCounter代表这个fiber内promise的索引
  const index = thenableIndexCounter;
  thenableIndexCounter += 1;
  if (thenableState === null) {
    // 创建一个空数组，用作存储promise
    thenableState = createThenableState();
  }
  // thenable是传入use的参数
  const result = trackUsedThenable(thenableState, thenable, index);
  if (
    // 渲染的阶段没有alternate,更新阶段才有
    currentlyRenderingFiber.alternate === null &&
    // 等于null说明use语句在第一行，在use之前没有别的hook
    (workInProgressHook === null
      ? currentlyRenderingFiber.memoizedState === null
      : workInProgressHook.next === null)
  ) {
    // 初始渲染，或者这是首次调用该组件，
    // 或者前一次在该 use() 之后没有调用任何钩子（可能是因为它抛出了）。
    // 后续的钩子调用应使用mount阶段的dispatcher。
    ReactSharedInternals.H = HooksDispatcherOnMount;
  }
  return result;
}
```

### trackUsedThenable

```js
export function trackUsedThenable<T>(
  thenableState: ThenableState,
  thenable: Thenable<T>,
  index: number
): T {
  // 相当于复制了一个thenableState出来
  const trackedThenables = getThenablesFromState(thenableState);
  const previous = trackedThenables[index];
  if (previous === undefined) {
    // 如果是第一个，把thenable存进去
    trackedThenables.push(thenable);
  } else {
    if (previous !== thenable) {
      // 重复使用以前的 thenable，放弃新的 thenable。
      // 我们可以假设它们代表相同的值，因为组件是幂等的。
      // 幂等：对于相同的输入，函数总是返回相同的输出。无论执行多少次，结果都跟执行一次一样
      if (__DEV__) {
        const thenableStateDev: ThenableStateDev = (thenableState: any);
        if (!thenableStateDev.didWarnAboutUncachedPromise) {
          thenableStateDev.didWarnAboutUncachedPromise = true;
          // promise必须被缓存，否则每次函数执行都会创建一个新的promise
          console.error(
            "A component was suspended by an uncached promise. Creating " +
              "promises inside a Client Component or hook is not yet " +
              "supported, except via a Suspense-compatible library or framework."
          );
        }
      }

      thenable.then(noop, noop);
      thenable = previous;
    }
  }

  // 我们跟踪 thenable 的状态和结果，
  // 这样就可以同步解包值。可以将其视为 Promise API 的扩展，
  // 或者是 Thenable 的超集的自定义接口。
  // 如果 thenable 没有状态，则将其设置为 "pending"，
  // 并附加一个监听器，在解决后更新其状态和结果。
  // Promise没有status
  switch (thenable.status) {
    case "fulfilled": {
      // 如果已经resolve那就直接返回值
      const fulfilledValue: T = thenable.value;
      return fulfilledValue;
    }
    case "rejected": {
      const rejectedError = thenable.reason;
      checkIfUseWrappedInAsyncCatch(rejectedError);
      throw rejectedError;
    }
    default: {
      if (typeof thenable.status === "string") {
        // 只有当状态未定义时，才对 thenable 进行检测。
        // 如果状态已定义，但值未知，则假定它已被某个自定义用户空间实现工具化。
        // 我们将其视为 "pending"。
        // 附加一个虚拟监听器，以确保可以进行任何懒惰初始化。
        // 当值被实际等待时，Flight 会懒散地解析 JSON。
        thenable.then(noop, noop);
      } else {
        // 这是一种我们以前从未见过的未缓存的 thenable。

        // 检测未缓存 promise 导致的无限 ping 循环。
        const root = getWorkInProgressRoot();
        if (root !== null && root.shellSuspendCounter > 100) {
          // 该根多次被挂起，但没有取得任何进展（即执行了某些操作）。
          // 这很有可能是一个无限 ping 循环，通常由意外的 Async 客户端组件引起。

          throw new Error(
            // async/await 尚不支持客户端组件，只支持服务器组件。
            // 这种错误通常是由于不小心将"'use client'"添加到了原本为服务器编写的模块中。
            "async/await is not yet supported in Client Components, only " +
              "Server Components. This error is often caused by accidentally " +
              "adding `'use client'` to a module that was originally written " +
              "for the server."
          );
        }

        const pendingThenable: PendingThenable<T> = (thenable: any);
        // 没有status会被默认为pending状态
        pendingThenable.status = "pending";
        pendingThenable.then(
          (fulfilledValue) => {
            if (thenable.status === "pending") {
              // 更新状态
              const fulfilledThenable: FulfilledThenable<T> = (thenable: any);
              fulfilledThenable.status = "fulfilled";
              fulfilledThenable.value = fulfilledValue;
            }
          },
          (error: mixed) => {
            if (thenable.status === "pending") {
              // 更新状态
              const rejectedThenable: RejectedThenable<T> = (thenable: any);
              rejectedThenable.status = "rejected";
              rejectedThenable.reason = error;
            }
          }
        );
      }

      // 再检查一次，以防 thenable 同步resolved。
      switch (thenable.status) {
        case "fulfilled": {
          const fulfilledThenable: FulfilledThenable<T> = (thenable: any);
          return fulfilledThenable.value;
        }
        case "rejected": {
          const rejectedThenable: RejectedThenable<T> = (thenable: any);
          const rejectedError = rejectedThenable.reason;
          checkIfUseWrappedInAsyncCatch(rejectedError);
          throw rejectedError;
        }
      }

      // Suspend.

      // 保存thenable，会在workLoop中catch中（handleThrow）使用
      suspendedThenable = thenable;
      throw SuspenseException;
    }
  }
}
```

#### 总结

- `use` 可以传 `promise` 和 `context`
- 如果是 `promise` 或者类 `promise`，会被保存在 `thenableState` 中，`thenableState` 代表当前 `fiber` 通过 `use` 传入的所有 `promise`，在`finishRenderingHooks`中会将`thenableState`重置
- `trackUsedThenable` 会处理 `promise` 的状态，并赋值`suspendedThenable`变量为当前`promise`，然后主动`throw SuspenseException`
- `use` 与 `Suspense` 强相关(`context` 用途除外)，主动抛出异常之后，强行终止 `render` 阶段，在`handleThrow`中对`workInProgressSuspendedReason`根据不同报错进行赋值，然后在`renderRootXxx`中根据不同的`workInProgressSuspendedReason`做对应处理
- 详情见 `Suspense` 流程讲解
- **！！！`use`可以在条件判断内使用，因为：**
  - `promise` 保存在数组中，在组件执行完之后，数组会和索引都会被清空，所以在条件语句中不会影响匹配`promise`，而像`useState`这种，`hook`是保存在`fiber.memorized`中，通过 next 链接的，和顺序有强相关
  - `context`是根据从`_currentValue`取值，没有顺序。`fiber.dependencies`每次函数执行都会更新，也没有顺序
