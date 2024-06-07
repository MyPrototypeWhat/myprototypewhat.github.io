---
title: React源码解析---React-schduler
toc: true
tags:
  - React
date: 2024-06-07 19:55:57
---

# React-Scheduler

> version@0.20.1

## 前言

`Scheduler`顾名思义就是一个调度器，负责`React`中**任务的调度**。众所周知 JS 是单线程，通过`task`和`micro task`来调度任务的执行。

核心概念分为三个：**时间切片**、**任务切片**和**优先级调度**

- 时间切片：将时间按帧切分**(默认 5ms)**执行任务，以达到不阻塞浏览器渲染。当页面有某处更新或者交互的时候，用户无感知阻塞或卡顿。
- 任务切片：如果一个任务过长，在一帧内无法完成，将中断任务，在下一帧重新调用。
- 优先级调度：通过不同优先级来决定某些任务优先调度。
  - 为了尽快的找到最高优先级的任务，使用了`小顶堆`的数据结构（不过多介绍）

## 选择

熟悉 JS 的同学肯定知道跟浏览器渲染帧相关的两个`API`：`requestIdleCallback（浏览器空闲时调用，下文简称rIC）`和`requestAnimationFrame（每一帧绘制之前调用，下文简称rAF）`，两个 api 看似可以达到不占用主线程，优先浏览器渲染，不阻塞的效果，但是真的适用吗？很显然，都有缺陷。

- `rIC`

  1. 兼容性太差，`Safari`直接**不兼容**...

     ![image-20221102175821806](/Users/xuan/Library/Application Support/typora-user-images/image-20221102175821806.png)

  2. **执行时间不一定：**浏览器空闲时执行间隔为**50ms**，也就是 20FPS，一秒执行 20 次，这显然间隔太长了。

     - 例如：持续滚动页面，这时执行的间隔时间就会非常不稳定

     - 还有一点，**当页面至于后台时，干脆不执行了**...

- `rAF`

  1. 执行顺序不一定，`rAF`是官方推荐用于做流畅动画的`api`，所以它的回调执行在页面**渲染更新前**。执行顺序可能在**宏任务**（`task`）**前或者后**（涉及到`EventLoop`，篇幅问题不过多介绍）。`rAF`在各个平台的浏览器表现不一。
  2. `React`可能执行两次更新

综上所述，React 团队打算自己实现一个策略，用于时间分片。最终使用`MessageChannel`实现

- 执行顺序，`microTask > messageChannel > setTimeout`，`messageChannel`为`dom event`，所以优先级要大于`setTimeout`
- 为什么用`task`而不用`microTask`？不用`microTask`的原因是，`microTask`将在页面更新前全部执行完，达不到将主线程还给浏览器的目的。

  - 根据事件循环规则来看每次执行一个`task`就会执行所有`microTask`，并且在这个过程中新增的`microTask`都会一并执行，所以`React`的渲染如果在`microTask`中，无法中断，
    - 因为`React`在中断渲染之后会检查是否还有任务，如果有就再次调度一个`performConcurrentWorkOnRoot`，根据事件循环来看这时再有`microTask`会立即执行，所以每次都会执行完全部任务，无法达到一个`tick`执行一个`task`的目的

- 为什么不使用`setTimeout`？因为`setTimeout(_,0)`即使设置为`0`，还会有**`4ms`的问题**。在`MessageChannel`无法使用的时候，降级使用`setTimeout`
- 为什么不使用`postMessage`？因为`postMessage`会因为持续的滚动等操作被阻塞住。浏览器会为了保证用户交互的响应，**将四分之三的优先权给了鼠标键盘事件**，其余的时间会交给其他的`task`，所以就导致了持续的滚动阻塞了`postMessage`，`Vue 2.0.0-rc.7`有个[issue](https://github.com/vuejs/vue/issues/3771)就是描述这个问题的。

## 预备知识点

`Scheduler`被单独拆成一个包，放在`React`项目中，目录为[react](https://github.com/facebook/react)/[packages](https://github.com/facebook/react/tree/main/packages)/[scheduler](https://github.com/facebook/react/tree/main/packages/scheduler)/[src](https://github.com/facebook/react/tree/main/packages/scheduler/src)/[forks](https://github.com/facebook/react/tree/main/packages/scheduler/src/forks)/Scheduler.js

### 全局变量

- 根据优先级对应不同`timeout`

  ```js
  var maxSigned31BitInt = 1073741823;
  // Times out immediately
  var IMMEDIATE_PRIORITY_TIMEOUT = -1;
  // Eventually times out
  var USER_BLOCKING_PRIORITY_TIMEOUT = 250;
  var NORMAL_PRIORITY_TIMEOUT = 5000;
  var LOW_PRIORITY_TIMEOUT = 10000;
  // Never times out
  var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt;
  ```

- 全局函数

  ```js
  // 获取currentTime(当前时间)
  let getCurrentTime = () => performance.now();
  // 延时器
  const localSetTimeout = typeof setTimeout === "function" ? setTimeout : null;
  // 清除延时器
  const localClearTimeout =
    typeof clearTimeout === "function" ? clearTimeout : null;
  // 环境支持的话
  const isInputPending = navigator.scheduling.isInputPending.bind(
    navigator.scheduling
  );
  ```

- 任务相关变量

  ```js
  var taskQueue = [];
  var timerQueue = [];
  // 当前任务
  var currentTask = null;
  // 当前任务优先级
  var currentPriorityLevel = NormalPriority;
  // flushWork中设置为true，表示当前任务正在执行，防止再次进入
  var isPerformingWork = false;
  // 表示任务是否被调度,调用requestHostCallback函数前设置为false（触发postMessage之前），在flushWork中设置为false
  var isHostCallbackScheduled = false;
  // 表示是否有延时器正在执行,延时器执行完毕之后设置为false
  var isHostTimeoutScheduled = false;
  ...
  // 代表当前postMessage触发的回调正在执行
  let isMessageLoopRunning = false;
  // scheduledHostCallback = flushWork
  let scheduledHostCallback = null;
  // 延时器id
  let taskTimeoutID = -1;
  ```

- 简单来说，任务分为两个堆——`taskQueue`和`timerQueue`，两个变量都是`js数组形式`的`小顶堆`，`taskQueue`根据`expirationTime`（过期时间）由小到大排序，`timerQueue`根据`startTime`（开始时间）**由大到小排序**
- `taskQueue`和`timerQueue`分别表示任务需要立刻执行和延迟执行，通过(`startTime`>`currentTime`)来判断任务是添加进`taskQueue`中还是`timerQueue`

### 局部变量

任务对象的属性

```js
var newTask = {
  // 自增的id，用来判断插入顺序，当sortIndex相同时，通过id判断优先级执行顺序
  id: taskIdCounter++,
  // performSyncWorkOnroot等，react render阶段的入口函数
  callback,
  // 优先级
  priorityLevel,
  // 开始时间 startTime=currentTime+delay(如果有的话)
  startTime,
  // 过期时间(startTime+timeout) timeout为不同优先级预设的时间
  expirationTime,
  // 堆排序的主要依据,timerQueue中为startTime，taskQueue中为expirationTime
  sortIndex: -1,
};
```

- 预备知识点完成，下面是函数部分

## 函数

### unstable_scheduleCallback

- 入口函数，分成四个部分

  - 第一部分，计算`startTime`，如果有`delay`就加上
  - 第二部分，根据传入的优先级，计算对应优先级的`timeout`
  - 第三部分，计算`expirationTime`，创建任务对象（`newTask`）
  - 第四部分，根据`startTime > currentTime`来判断是`push`进`timerQueue`中还是`taskQueue`

```js
function unstable_scheduleCallback(priorityLevel, callback, options) {
  // 第一部分
  var currentTime = getCurrentTime();

  var startTime;
  if (typeof options === "object" && options !== null) {
    var delay = options.delay;
    if (typeof delay === "number" && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }
  // 第二部分
  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }
  // 第三部分
  var expirationTime = startTime + timeout;

  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };
  if (enableProfiling) {
    newTask.isQueued = false;
  }
  // 第四部分
  if (startTime > currentTime) {
    // 延迟任务.
    newTask.sortIndex = startTime;
    // push时会根据startTime进行排序
    push(timerQueue, newTask);
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // taskQueue中没有任务，并且timerQueue中有任务，拿到优先级最高的任务(当前任务)(startTime最小)
      if (isHostTimeoutScheduled) {
        // 如果当前有上一个被通过setTimeout延迟执行的任务就取消掉
        cancelHostTimeout();
      } else {
        // 如果没有，就设置为true，代表当前有被调度的任务
        isHostTimeoutScheduled = true;
      }
      // 将延迟任务通过setTimeout变为立即执行任务
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    // 如果当前没有正在调度的任务，并且没有正在执行的任务
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      // 立即执行
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```

### requestHostTimeout

这部分代码很简单，就是一个延时器

```js
function requestHostTimeout(callback, ms) {
  // callback = handleTimeout
  taskTimeoutID = localSetTimeout(() => {
    callback(getCurrentTime());
  }, ms);
}
```

### handleTimeout

判断当前是否有被调度的任务，如果有就取出`timeQueue`第一位，继续等待执行，如果没有就直接调度该任务

```js
function handleTimeout(currentTime) {
  // 当前延时器回调执行了，isHostTimeoutScheduled为false代表释放当前延时器，下一个延时任务可以被调度
  isHostTimeoutScheduled = false;
  // 将timerQueue中已经过期了的任务插入到taskQueue中
  advanceTimers(currentTime);
  // 判断当前是否有被调度的任务
  if (!isHostCallbackScheduled) {
    // 没有并且taskQueue中有任务
    if (peek(taskQueue) !== null) {
      // 开始调度taskQueue中的任务
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    } else {
      // 有则取出继续等待调度
      const firstTimer = peek(timerQueue);
      if (firstTimer !== null) {
        requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
      }
    }
  }
}
```

### advanceTimers

将`timerQueue`中已经过期了的任务插入到`taskQueue`中

```js
function advanceTimers(currentTime) {
  // 检查不再延迟的任务，并将其添加到队列中。
  let timer = peek(timerQueue);
  // 遍历timerQueue
  while (timer !== null) {
    if (timer.callback === null) {
      // 任务被取消，出堆
      pop(timerQueue);
    } else if (timer.startTime <= currentTime) {
      // 计时器响了。转移到任务队列。
      pop(timerQueue);
      // 因为要插进taskQueue，所以要重新计算sortIndex
      timer.sortIndex = timer.expirationTime;
      push(taskQueue, timer);
    } else {
      // 后面的任务的时间有剩余
      return;
    }
    timer = peek(timerQueue);
  }
}
```

以上是调度`timerQueue`的过程，其中执行的函数和调度`taskQueue`中函数有重复，放在下面讲解

---

### requestHostCallback

入参为`flushWork`，将`flushwork`赋值给全局变量，并且触发消息通知

```js
function requestHostCallback(callback) {
  // callback = flushWork
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    // 执行postMessage
    schedulePerformWorkUntilDeadline();
  }
}
```

### schedulePerformWorkUntilDeadline

对于设备环境做了兼容

```js
let schedulePerformWorkUntilDeadline;
if (typeof localSetImmediate === "function") {
  // Node.js 和 old IE.
  schedulePerformWorkUntilDeadline = () => {
    localSetImmediate(performWorkUntilDeadline);
  };
} else if (typeof MessageChannel !== "undefined") {
  // DOM and Worker environments.
  // 由于setTimeout的4ms延迟，所以使用MessageChannel
  const channel = new MessageChannel();
  const port = channel.port2;
  channel.port1.onmessage = performWorkUntilDeadline;
  schedulePerformWorkUntilDeadline = () => {
    port.postMessage(null);
  };
} else {
  // 非浏览器环境使用setTimeout
  schedulePerformWorkUntilDeadline = () => {
    localSetTimeout(performWorkUntilDeadline, 0);
  };
}
```

- 已`MessageChannel`为例，执行`schedulePerformWorkUntilDeadline`会触发`performWorkUntilDeadline`执行，**但是会放在下一轮事件循环中执行**

### performWorkUntilDeadline

作为`postMessage`触发的回调，主要负责执行全局变量`scheduledHostCallback`，通过返回值判定是否触发下一轮`postMessage`

```js
const performWorkUntilDeadline = () => {
  // scheduledHostCallback 在 requestHostCallback 中被赋值为 flushWork
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    // 获取函数真正执行的当前时间，提供给后续时间片判断(shouldYieldToHost函数)
    startTime = currentTime;
    const hasTimeRemaining = true;
    // 故意不使用try-catch，因为这会使一些调试技术变得更加困难。
    // 相反，如果'scheduledHostCallback'出现错误，
    // 那么'hasMoreWork'将保持为true，我们将继续工作循环。
    let hasMoreWork = true;
    try {
      // scheduledHostCallback = flushWork
      hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
    } finally {
      if (hasMoreWork) {
        // 代表当前任务没结束（返回一个函数、报错、）
        schedulePerformWorkUntilDeadline();
      } else {
        // 重置全局变量
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      }
    }
  } else {
    isMessageLoopRunning = false;
  }
};
```

### flushWork

**核心**，负责执行 workLoop 并且返回执行结果，在执行结束后重置全局变量

```js
function flushWork(hasTimeRemaining, initialTime) {
  // 设为false，为了能够执行requestHostCallback，调度下次任务
  isHostCallbackScheduled = false;
  if (isHostTimeoutScheduled) {
    // 如果当前有延时器就取消掉,当前任务优先级更高
    // 因为接下来执行callback前后会再次执行advanceTimers，并且执行callback也是会有时间损耗的
    isHostTimeoutScheduled = false;
    cancelHostTimeout();
  }
  // 任务开始执行
  isPerformingWork = true;
  const previousPriorityLevel = currentPriorityLevel;
  try {
    // 直接看这里，返回值为外部函数作用域的hasMoreWork变量
    return workLoop(hasTimeRemaining, initialTime);
  } finally {
    // 任务执行完成之后重置全局变量
    currentTask = null;
    currentPriorityLevel = previousPriorityLevel;
    isPerformingWork = false;
  }
}
```

### workLoop

**核心**，任务循环，真正执行`callback`的地方。在执行`callback`前后都会执行`advanceTimers`，确保`taskQueue`中任务的优先级

```js
function workLoop(hasTimeRemaining, initialTime) {
  // initialTime为 postMessage触发回调当时的时间
  // hasTimeRemaining 始终为true
  let currentTime = initialTime;
  // 检测是否有过期的任务，放在taskQueue队列中
  advanceTimers(currentTime);
  currentTask = peek(taskQueue);
  while (currentTask !== null) {
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // 此判断代表任务还未过期，但是没时间了(5ms已过)，终止循环
      // shouldYieldToHost 判断为
      // if(getCurrentTime() - startTime < frameInterval(默认5ms)) return false ，执行到现在还没到5ms，不需要暂停

      // shouldYieldToHost 还有对isInputPending情况的判断，兼容性不高不做考虑
      // navigator.scheduling.isInputPending为react团队和chrome团队协商出的api，主要用于判断当前是否有input等事件正在执行，有兴趣可以了解下
      break;
    }
    const callback = currentTask.callback;
    if (typeof callback === "function") {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      // 当前任务是否超时
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      // 拿到返回
      // 任务切片
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();
      if (typeof continuationCallback === "function") {
        // 如果是个函数，更新callback,作为下一轮事件循环使用
        currentTask.callback = continuationCallback;
      } else {
        // 判断当前任务是否是最高优先级任务，是则pop
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
      // 检测是否有过期的任务，放在taskQueue队列中
      advanceTimers(currentTime);
    } else {
      // 任务被取消，出堆
      pop(taskQueue);
    }
    // 更新currentTask继续循环
    currentTask = peek(taskQueue);
  }
  if (currentTask !== null) {
    // 走到这个判断会有两种情况
    // callback返回一个函数 或者 达到当前deadline(默认5ms的限制)
    return true;
  } else {
    const firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      // taskQueue中已经没有可执行的任务了，取出timerQueue中的任务，进行调度
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    return false;
  }
}
```

## 总结

总结下大致流程：通过将任务划分成`立即执行`和`延迟执行`两个堆，立即执行的堆中任务会通过`MessageChannel`触发，延迟执行的堆会通过`setTimeout`将任务延迟到对应事件后添加进`taskQueue`中触发。

每次 workLoop 都会从`taskQueue`中取出任务，执行任务，如果任务执行完之后还有剩余时间，则继续执行，直到没有剩余时间或者任务队列为空。如果 `5ms` 到了，但是还有任务，则通过 `postMessage` 开启下一轮 `workLoop。`达到让出主线程的能力
