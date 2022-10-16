---
title: ReactNative的渲染流程(基于ReacNative0.61.5)
date: 2020-04-07 21:00:02
tags: [ReactNative,Android]
---

文章参考：https://www.jianshu.com/p/4e0217b8f8aa

Scheduler
https://zhuanlan.zhihu.com/p/84952196

React Scheduler 源码详解
https://juejin.im/post/5c32c0c86fb9a049b7808665

React Fiber源码理解
https://juejin.im/post/5d4d2641f265da03ee6a45b6

Fiber
https://stackblitz.com/edit/js-ntqfil?file=index.js

effect-list fiber链表构建过程
https://www.bilibili.com/video/av48384879/

View创建过程浅析
https://juejin.im/post/5bfbaaf1f265da615a417f69

React Native源码分析——Virtual DOM
https://www.jianshu.com/p/98ef4ce18bb3

一致性比较（Reconciliation）
http://react.html.cn/docs/reconciliation.html

完全理解React Fiber
http://www.ayqy.net/blog/dive-into-react-fiber/


React Native组件渲染
https://www.jianshu.com/p/c5418720e9f0

React的首次渲染流程逻辑：
https://juejin.im/post/5b9a45fc5188255c402af11f


我们先来分析 JS 端的渲染流程是怎样的。

### JSX 的转码
我们写 React 组件的时候，通常会使用JSX来描述组件。JSX 语法是对 JS语法的一种扩展，为了方便开发者在 JS 中编写 UI，在运行期间会通过BABEL转码，我们通过一个比较简单的示例来做转码，原始的 ReactNative 代码如下：

```
export default class Sample extends Component {
  render() {
    return (
      <View>
        <Text>你好，渲染流程</Text>
        <Text>Hello Render</Text>
      </View>
    );
  }
}
```
经过[BABEL](https://babeljs.io/repl/#?browsers=&build=&builtIns=false&spec=false&loose=false&code_lz=KYDwDg9gTgLgBAE2AMwIYFcA28DGnUDOBcAyqgLZibBygzAB2CxAwhJRA4_AN4BQcOFEZIoACgCUcfoMHCY6KAzhiBswQB4AagEtgAdwB8a9ZoAqdQ4AN5QL6agGH_ATHaBl80CCtoGgvDQHoLIGMdOyGj5-ABLAmJgQcABKIsBQXsH-pl66BsmCEgDcagC-fLlAA&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=module&lineWrap=true&presets=react&prettier=false&targets=&version=7.9.0&externalPlugins=babel-plugin-syntax-jsx%406.18.0)转码后：


```
export default class Sample extends Component {
  render() {
    return 
        React.createElement(View, null, 
            React.createElement(Text, null, "你好，渲染流程"), 
            React.createElement(Text, null, "Hello Render")
        );
  }

}
```
我们可以看到，原先 JSX 的元素都被 React.createElement 转化为 ReactElement。在 ReactElement 中使用 type字段存放原始对象（在此处就是ReactNative.View/ReactNative.Text），使用props存放 childrens、其他传入属性等。

### AppRegister.js的runApplication
按照之前讲解ReactNative的启动流程，我们可以知道。启动流程的最后一步，其实是执行到了AppRegister.js的runApplication

```
  /**
   * Loads the JavaScript bundle and runs the app.
   *
   * See http://facebook.github.io/react-native/docs/appregistry.html#runapplication
   */
  runApplication(appKey: string, appParameters: any): void {
    const msg =
      'Running "' + appKey + '" with ' + JSON.stringify(appParameters);
    infoLog(msg);
    BugReporting.addSource(
      'AppRegistry.runApplication' + runCount++,
      () => msg,
    );
    invariant(
      runnables[appKey] && runnables[appKey].run,
      `"${appKey}" has not been registered. This can happen if:\n` +
        '* Metro (the local dev server) is run from the wrong folder. ' +
        'Check if Metro is running, stop it and restart it in the current project.\n' +
        "* A module failed to load due to an error and `AppRegistry.registerComponent` wasn't called.",
    );

    SceneTracker.setActiveScene({name: appKey});
    // 根据AppKey来遍历我们注册的组件。然后调用run方法
    runnables[appKey].run(appParameters);
  },
```
这个时候，我们回去看registerComponent。我们添加到runnables数组中的对象，其中的run方法。下面我们就来看一下AppRegistry.registerComponent


### AppRegistry.registerComponent

首先，ReactNative的js代码都需要通过 AppRegistry.registerComponent注册对应appkey的Component才能被启动。

我们可以在 AppRegistry.js 中看到它注册了一个对应的回调，在 Native 启动过程中会通过jsbridge调用AppRegistry.runApplication启动js 渲染流程，在js中会调用对应runnable。即后面的renderApplication。


```
  /**
   * Registers an app's root component.
   *
   * See http://facebook.github.io/react-native/docs/appregistry.html#registercomponent
   * 参数列表：
   *  appKey: 就是专门为我们应用标记的key : HelloRNSource
   *  ComponentProvider是一个function对象。也就是我们那个App的function方法
   *  还有一个重要的参数：appParameters 这个是我们从Java那边带过来的参数对象initialProps、rootTag等
   *  appKey：HelloRNSource
   */
  registerComponent(
    appKey: string,
    componentProvider: ComponentProvider,
    section?: boolean,
  ): string {
    console.log(
      'AppRegistry.js',
      'FMsg:第二步：<<<<<<<<<<<<<<<<<<AppRegistry.registerComponent<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
    );
    let scopedPerformanceLogger = createPerformanceLogger();
    // 实例化一个runables对象
    runnables[appKey] = {
      componentProvider,
      run: appParameters => {
        renderApplication(
          componentProviderInstrumentationHook(
            componentProvider,
            scopedPerformanceLogger,
          ),
          appParameters.initialProps,
          appParameters.rootTag,
          wrapperComponentProvider && wrapperComponentProvider(appParameters),
          appParameters.fabric,
          showFabricIndicator,
          scopedPerformanceLogger,
        );
      },
    };
    if (section) {
      sections[appKey] = runnables[appKey];
    }
    return appKey;
  },
```

在 renderApplication 时会将传入的 Component 变成 ReactElement，包裹在 AppContainer 中，这个 AppContainer 主要用于外面包围一些 Debug 用的工具（如红盒）。

run方法接收一个参数，叫做appParameters。由上面的分析可知，appParameters就是Android端传递进来的WritableNativeMap对象，其代表的是一些初始化参数。run方法中没有做过多的逻辑，而是直接调用了renderApplication.js中的renderApplication方法。

### renderApplication.js的的renderApplication

我们按照代码流程进行追溯，发下他调用了renderApplication.js的的renderApplication方法。

```
/**
 * 
 * @param {Root组件，就是index.js里面注册App组件 RootComponent 
 * @param {*} initialProps  传递过来的默认属性，一般为空对象
 * @param {*} rootTag 
 * @param {*} WrapperComponent 
 * @param {*} fabric 
 * @param {*} showFabricIndicator 
 * @param {*} scopedPerformanceLogger 
 */
function renderApplication<Props: Object>(
  RootComponent: React.ComponentType<Props>,
  initialProps: Props,
  rootTag: any,
  WrapperComponent?: ?React.ComponentType<*>,
  fabric?: boolean,
  showFabricIndicator?: boolean,
  scopedPerformanceLogger?: IPerformanceLogger,
) {
  console.log(
    'renderApplication.js',
    'FMsg:<<<<<<<<<<<<<<<<renderApplication<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<',rootTag
  );
  invariant(rootTag, 'Expect to have a valid rootTag, instead got ', rootTag);
  // 使用AppContainer进行包括起来
  //这个 AppContainer 主要用于外面包围一些 Debug 用的工具（如红盒）。
  const renderable = (
    <PerformanceLoggerContext.Provider
      value={scopedPerformanceLogger ?? GlobalPerformanceLogger}>
      <AppContainer rootTag={rootTag} WrapperComponent={WrapperComponent}>
        <RootComponent {...initialProps} rootTag={rootTag} />
        {fabric === true && showFabricIndicator === true ? (
          <ReactFabricIndicator />
        ) : null}
      </AppContainer>
    </PerformanceLoggerContext.Provider>
  );

  GlobalPerformanceLogger.startTimespan('renderApplication_React_render');
  if (fabric) {
    require('../Renderer/shims/ReactFabric').render(renderable, rootTag);
  } else {
    require('../Renderer/shims/ReactNative').render(renderable, rootTag);
  }
  console.log(
    'renderApplication.js',
    'FMsg:最后一步：<<<<<<<<<<<<<<<<renderApplication<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<',rootTag
  );
  GlobalPerformanceLogger.stopTimespan('renderApplication_React_render');
}
```

renderApplication方法没有过多复杂的逻辑，而是直接调用了ReactNative的render方法。

调用栈可以看出，renderApplication.js的renderApplication方法调用了ReactNativeRenderer-dev.js的render方法。

这个就是JS层比较重要的渲染的流程的。


### ReactNativeRenderer-dev.js的render方法

我们来分析调试过程中的渲染流程
```
  /**
   * 渲染流储能
   * @param {渲染的组件} element 
   * @param {*} containerTag 
   * @param {*} callback 
   */
  render: function(element, containerTag, callback) {
    // 获取root根组件
    var root = roots.get(containerTag);
    // 如果root根组件为空，则进行创建根组件容器。
    if (!root) {
      // TODO (bvaughn): If we decide to keep the wrapper component,
      // We could create a wrapper for containerTag as well to reduce special casing.
      root = createContainer(containerTag, LegacyRoot, false, null);
      roots.set(containerTag, root);
    }
    // 获取到根组件只会进行更新容器组件
    updateContainer(element, root, null, callback);
    return getPublicRootInstance(root);
  },
```
我们先看看一下renderApplication中我们传递过来的参数

![-w533](https://i.loli.net/2020/04/07/mucjBQe92iZXk5z.jpg)


当我在应用刚开始启动的时候，我们首先渲染root组件。但我们还没有进行createContainer的时候。这个时候Root组件肯定是为null .所以这个时候，先调用createContainer

### createContainer方案调用

```
function createContainer(containerInfo, tag, hydrate, hydrationCallbacks) {
  return createFiberRoot(containerInfo, tag, hydrate, hydrationCallbacks);
}


function createFiberRoot(containerInfo, tag, hydrate, hydrationCallbacks) {
  var root = new FiberRootNode(containerInfo, tag, hydrate);

  if (enableSuspenseCallback) {
    root.hydrationCallbacks = hydrationCallbacks;
  } // Cyclic construction. This cheats the type system right now because
  // stateNode is any.

  var uninitializedFiber = createHostRootFiber(tag);
  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;
  return root;
}

```
这个方法调用完成之后，会生成一个FiberRootNode的节点。这个节点的current指向一个FiberNode节点。

![https://i.loli.net/2020/04/09/5M9xYREhqv3AKGs.jpg](https://i.loli.net/2020/04/09/5M9xYREhqv3AKGs.jpg)


执行完CreateContainer之后。代码就走到updateContainer(element, root, null, callback)的方法里面。

```
function updateContainer(element, container, parentComponent, callback) {
  // 如果是根节点。活我们获取Root节点的current。我通过createContinainer知道其实就是一个FiberNode节点
  var current$$1 = container.current;
  console.log(
    'ReactNativeRenader-dev.js',
    'FMsg:染流程第六步：<<<<<<<<<<<<<<<<updateContainer<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
  );

  var currentTime = requestCurrentTime();
  {
    // $FlowExpectedError - jest isn't a global, and isn't recognized outside of tests
    if ("undefined" !== typeof jest) {
      warnIfUnmockedScheduler(current$$1);
      warnIfNotScopedWithMatchingAct(current$$1);
    }
  }
  var suspenseConfig = requestCurrentSuspenseConfig();
  var expirationTime = computeExpirationForFiber(
    currentTime,
    current$$1,
    suspenseConfig
  );
  return updateContainerAtExpirationTime(
    element,
    container,
    parentComponent,
    expirationTime,
    suspenseConfig,
    callback
  );
}
```
这个方法做了一些准备工作，调用updateContainerAtExpirationTime
```
function updateContainerAtExpirationTime(
  element,
  container,
  parentComponent,
  expirationTime,
  suspenseConfig,
  callback
) {
  console.log(
    'ReactNativeRenader-dev.js',
    'FMsg:<<<<<<<<<<<<<<<<updateContainerAtExpirationTime<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
  );
  // TODO: If this is a nested container, this won't be the root.
  var current$$1 = container.current;

  {
    if (ReactFiberInstrumentation_1.debugTool) {
      if (current$$1.alternate === null) {
        ReactFiberInstrumentation_1.debugTool.onMountContainer(container);
      } else if (element === null) {
        ReactFiberInstrumentation_1.debugTool.onUnmountContainer(container);
      } else {
        ReactFiberInstrumentation_1.debugTool.onUpdateContainer(container);
      }
    }
  }

  var context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  return scheduleRootUpdate(
    current$$1,
    element,
    expirationTime,
    suspenseConfig,
    callback
  );
}
```
这个地方，如果是根节点的话，则需要

### scheduleRootUpdate调用

```
function scheduleRootUpdate(
  current$$1,
  element,
  expirationTime,
  suspenseConfig,
  callback
) {
  console.log(
    'ReactNativeRenader-dev.js',
    'FMsg:<<<<<<<<<<<<<<<<scheduleRootUpdate<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
  );
  {
    if (phase === "render" && current !== null && !didWarnAboutNestedUpdates) {
      didWarnAboutNestedUpdates = true;
      warningWithoutStack$1(
        false,
        "Render methods should be a pure function of props and state; " +
          "triggering nested component updates from render is not allowed. " +
          "If necessary, trigger nested updates in componentDidUpdate.\n\n" +
          "Check the render method of %s.",
        getComponentName(current.type) || "Unknown"
      );
    }
  }
  //创建一个更新队列.重点看我们下面给这个更新队列设置的几个属性
  var update = createUpdate(expirationTime, suspenseConfig);
  // Caution: React DevTools currently depends on this property
  // being called "element".
  // 表示这个更新对应的数据内容。这个主意，这个就把我们设置的各种组件ReactElement的赋值给update
  update.payload = { element: element };

  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    !(typeof callback === "function")
      ? warningWithoutStack$1(
          false,
          "render(...): Expected the last optional `callback` argument to be a " +
            "function. Instead received: %s.",
          callback
        )
      : void 0;
    // 设置这个组件每个节点更新完成之后的回调。很重要
    update.callback = callback;
  }

  if (revertPassiveEffectsChange) {
    flushPassiveEffects();
  }
  // 加入到更新队列中
  enqueueUpdate(current$$1, update);
  // 进度调度工作
  scheduleWork(current$$1, expirationTime);

  return expirationTime;
}
```

这个方法中创建了一个更新队列。默认的更新队列属性参数的如下图：

```
    /**
     * 在调度算法执行过程中，会将需要进行变更的动作以一个Update数据来表示。同一个队列中的Update，会通过next属性串联起来，实际上也就是一个单链表。
     * @param {*} expirationTime 
     * @param {*} suspenseConfig 
     */
    function createUpdate(expirationTime, suspenseConfig) {
      var update = {
        expirationTime: expirationTime,
        suspenseConfig: suspenseConfig,
        // 当前有0~3，分别是UpdateState、ReplaceState、ForceUpdate、CaptureUpdate
        // var UpdateState = 0;
        // var ReplaceState = 1;
        // var ForceUpdate = 2;
        // var CaptureUpdate = 3;
        tag: UpdateState,
        // 表示这个更新对应的数据内容
        payload: null,
        // 表示更新后的回调函数，如果这个回调有值，就会在UpdateQueue的副作用链表中挂在当前Update对象
        callback: null,
        // UpdateQueue中的Update之间通过next来串联，表示下一个Update对象
        next: null,
        nextEffect: null
      };
      {
        update.priority = getCurrentPriorityLevel();
      }
      return update;
    }
```


我们在看一下，这个给这个更新队列的设置的update.payload属性的参数：
![-w530](https://i.loli.net/2020/04/07/IqxTZkUXwBA7hsE.jpg)

![](https://i.loli.net/2020/04/09/5u6eth1PS3ynR72.jpg)


从上面这个截图我们就可以看出，我们自己定义的组件。已经添加到更新队列中


### enqueueUpdate

然后，这个执行完毕之后，调用的函数： enqueueUpdate(current$$1, update)

下面，我们来看一下这个方法。添加到更新队列中


```
function enqueueUpdate(fiber, update) {
  console.log(
    'ReactNativeRenader-dev.js',
    'FMsg:<<<<<<<<<<<<<<<<enqueueUpdate<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
  );
  // Update queues are created lazily.
  var alternate = fiber.alternate;
  var queue1 = void 0;
  var queue2 = void 0;
  if (alternate === null) {
    // There's only one fiber.  给fiber的updateQueue赋值给queue1
    // Root节点的时候updateQueue应该是个Null
    queue1 = fiber.updateQueue;
    queue2 = null;
    // 这个地方进行创建更新的队列
    if (queue1 === null) {
      queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
    }
  } else {
    // There are two owners.
    queue1 = fiber.updateQueue;
    queue2 = alternate.updateQueue;
    if (queue1 === null) {
      if (queue2 === null) {
        // Neither fiber has an update queue. Create new ones.
        queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
        queue2 = alternate.updateQueue = createUpdateQueue(
          alternate.memoizedState
        );
      } else {
        // Only one fiber has an update queue. Clone to create a new one.
        queue1 = fiber.updateQueue = cloneUpdateQueue(queue2);
      }
    } else {
      if (queue2 === null) {
        // Only one fiber has an update queue. Clone to create a new one.
        queue2 = alternate.updateQueue = cloneUpdateQueue(queue1);
      } else {
        // Both owners have an update queue.
      }
    }
  }
  // 当队列为空的时候，我们进行队列尾部填充我们这个需要更新
  if (queue2 === null || queue1 === queue2) {
    // There's only a single queue.
    // 这个地方，我们看到queue2为null  或者queue1 === queue2。我们是将update添加到queue1的队列尾部
    appendUpdateToQueue(queue1, update);
  } else {
    // There are two queues. We need to append the update to both queues,
    // while accounting for the persistent structure of the list — we don't
    // want the same update to be added multiple times.
    if (queue1.lastUpdate === null || queue2.lastUpdate === null) {
      // One of the queues is not empty. We must add the update to both queues.
      appendUpdateToQueue(queue1, update);
      appendUpdateToQueue(queue2, update);
    } else {
      // Both queues are non-empty. The last update is the same in both lists,
      // because of structural sharing. So, only append to one of the lists.
      appendUpdateToQueue(queue1, update);
      // But we still need to update the `lastUpdate` pointer of queue2.
      queue2.lastUpdate = update;
    }
  }

  {
    if (
      fiber.tag === ClassComponent &&
      (currentlyProcessingQueue === queue1 ||
        (queue2 !== null && currentlyProcessingQueue === queue2)) &&
      !didWarnUpdateInsideUpdate
    ) {
      warningWithoutStack$1(
        false,
        "An update (setState, replaceState, or forceUpdate) was scheduled " +
          "from inside an update function. Update functions should be pure, " +
          "with zero side-effects. Consider using componentDidUpdate or a " +
          "callback."
      );
      didWarnUpdateInsideUpdate = true;
    }
  }
}
```

```
function appendUpdateToQueue(queue, update) {
  console.log(
    'ReactNativeRenader-dev.js',
    'FMsg:<<<<<<<<<<<<<<<<appendUpdateToQueue<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
  );
  // Append the update to the end of the list.
  if (queue.lastUpdate === null) {
    // Queue is empty。如果队列是空的，则添加到更新的firstUpdate、lastUpdate
    queue.firstUpdate = queue.lastUpdate = update;
  } else {
    queue.lastUpdate.next = update;
    queue.lastUpdate = update;
  }
}
```

上面的方法执行完毕之后，我们的update.payload就加入到我们的更新队列。

执行完enqueueUpdate之后，我们再来看一下。
![](https://i.loli.net/2020/04/09/1obGIXcyU26RjMT.jpg)

当前FiberNode的节点已经绑定的updateQueue的节点。里面的firstUpdate的节点，我们可以看到执行update队列。

### scheduleWork

下面我们调用scheduleWork的方法的调用

```
 var scheduleWork = scheduleUpdateOnFiber;
```


```
/**
     * 进行Fiber调度更新。这个方法是比较重要
     */
    function scheduleUpdateOnFiber(fiber, expirationTime) {
      console.log(
        'ReactNativeRender-dev.js',
        'FMsg:<<<<<<<<<<<<<<<<scheduleUpdateOnFiber<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
      );
      checkForNestedUpdates();
      warnAboutInvalidUpdatesOnClassComponentsInDEV(fiber);

      var root = markUpdateTimeFromFiberToRoot(fiber, expirationTime);
      if (root === null) {
        warnAboutUpdateOnUnmountedFiberInDEV(fiber);
        return;
      }

      root.pingTime = NoWork;

      checkForInterruption(fiber, expirationTime);
      recordScheduleUpdate();

      // TODO: computeExpirationForFiber also reads the priority. Pass the
      // priority as an argument to that function and this one.
      var priorityLevel = getCurrentPriorityLevel();
      // 如果有效时间，是同步处理的逻辑。
      if (expirationTime === Sync) {
        if (
          // Check if we're inside unbatchedUpdates
          (executionContext & LegacyUnbatchedContext) !== NoContext &&
          // Check if we're not already rendering
          (executionContext & (RenderContext | CommitContext)) === NoContext
        ) {
          // Register pending interactions on the root to avoid losing traced interaction data.
          schedulePendingInteractions(root, expirationTime);

          // This is a legacy edge case. The initial mount of a ReactDOM.render-ed
          // root inside of batchedUpdates should be synchronous, but layout updates
          // should be deferred until the end of the batch.
          var callback = renderRoot(root, Sync, true);
          while (callback !== null) {
            callback = callback(true);
          }
        } else {
          // 如果是ROOT节点的话，我们先来处理一下现在的逻辑。
          scheduleCallbackForRoot(root, ImmediatePriority, Sync);
          if (executionContext === NoContext) {
            // Flush the synchronous work now, wnless we're already working or inside
            // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
            // scheduleCallbackForFiber to preserve the ability to schedule a callback
            // without immediately flushing it. We only do this for user-initiated
            // updates, to preserve historical behavior of sync mode.
            // 这个方法就是进行同步回调队列的取用处理
            flushSyncCallbackQueue();
          }
        }
      } else {
        // 进行Root节点的调度处理
        scheduleCallbackForRoot(root, priorityLevel, expirationTime);
      }

      if (
        (executionContext & DiscreteEventContext) !== NoContext &&
        // Only updates at user-blocking priority or greater are considered
        // discrete, even inside a discrete event.
        (priorityLevel === UserBlockingPriority ||
          priorityLevel === ImmediatePriority)
      ) {
        // This is the result of a discrete event. Track the lowest priority
        // discrete update per root so we can flush them early, if needed.
        if (rootsWithPendingDiscreteUpdates === null) {
          rootsWithPendingDiscreteUpdates = new Map([[root, expirationTime]]);
        } else {
          var lastDiscreteTime = rootsWithPendingDiscreteUpdates.get(root);
          if (lastDiscreteTime === undefined || lastDiscreteTime > expirationTime) {
            rootsWithPendingDiscreteUpdates.set(root, expirationTime);
          }
        }
      }
    }
```

### scheduleCallbackForRoot方法调用
进行ROOT的调度工作


```
function scheduleCallbackForRoot(root, priorityLevel, expirationTime) {
  console.log(
    'ReactNativeRender-dev.js',
    'FMsg:<<<<<<<<<<<<<<<<scheduleCallbackForRoot<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
  );
  var existingCallbackExpirationTime = root.callbackExpirationTime;
  if (existingCallbackExpirationTime < expirationTime) {
    // New callback has higher priority than the existing one.
    var existingCallbackNode = root.callbackNode;
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
    }
    root.callbackExpirationTime = expirationTime;
    // 添加到的同步消息队列
    if (expirationTime === Sync) {
      // Sync React callbacks are scheduled on a special internal queue
      root.callbackNode = scheduleSyncCallback(
        runRootCallback.bind(
          null,
          root,
          renderRoot.bind(null, root, expirationTime)
        )
      );
    } else {
      var options = null;
      if (
        !disableSchedulerTimeoutBasedOnReactExpirationTime &&
        expirationTime !== Never
      ) {
        var timeout = expirationTimeToMs(expirationTime) - now();
        options = { timeout: timeout };
      }

      root.callbackNode = scheduleCallback(
        priorityLevel,
        runRootCallback.bind(
          null,
          root,
          renderRoot.bind(null, root, expirationTime)
        ),
        options
      );
      if (
        enableUserTimingAPI &&
        expirationTime !== Sync &&
        (executionContext & (RenderContext | CommitContext)) === NoContext
      ) {
        // Scheduled an async callback, and we're not already working. Add an
        // entry to the flamegraph that shows we're waiting for a callback
        // to fire.
        startRequestCallbackTimer();
      }
    }
  }

  // Associate the current interactions with this new root+priority.
  schedulePendingInteractions(root, expirationTime);
}
```


schedulePendingInteractions 调度处理等待的队列中的数据

```
function schedulePendingInteractions(root, expirationTime) {
  console.log(
    'scheduler.develpment.js',
    'FMsg:<<<<<<<<<<<<<<<<schedulePendingInteractions<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
  );
  // This is called when work is scheduled on a root.
  // It associates the current interactions with the newly-scheduled expiration.
  // They will be restored when that expiration is later committed.
  if (!enableSchedulerTracing) {
    return;
  }

  scheduleInteractions(root, expirationTime, tracing.__interactionsRef.current);
}
```



下面，我们来回到scheduleUpdateOnFiber的方法中。在执行完scheduleCallbackForRoot之后。下面调用的flushSyncCallbackQueue。



这个方法是同步回调队列的却用处理。我们来看一下这个方法的实现flushSyncCallbackQueue();

### flushSyncCallbackQueue
```
/**
 * 进行同步队列里面的更新
 */
function flushSyncCallbackQueue() {
  console.log(
    'ReactNativeRender-dev.js',
    'FMsg:<<<<<<<<<<<<<<<<flushSyncCallbackQueue<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
  );
  if (immediateQueueCallbackNode !== null) {
    Scheduler_cancelCallback(immediateQueueCallbackNode);
  }
  flushSyncCallbackQueueImpl();
}
```
### flushSyncCallbackQueueImpl

这个方法调用了flushSyncCallbackQueueImpl()


```
    function flushSyncCallbackQueueImpl() {
      console.log(
        'ReactNativeRender-dev.js',
        'FMsg:<<<<<<<<<<<<<<<<flushSyncCallbackQueueImpl<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
      );
      if (!isFlushingSyncQueue && syncQueue !== null) {
        // Prevent re-entrancy.
        isFlushingSyncQueue = true;
        var i = 0;
        try {
          var _isSync = true;
          var queue = syncQueue;
          // 从所有的同步队列中取出
          runWithPriority(ImmediatePriority, function () {
            for (; i < queue.length; i++) {
              var callback = queue[i];
              do {
                callback = callback(_isSync);
              } while (callback !== null);
            }
          });
          syncQueue = null;
        } catch (error) {
          // If something throws, leave the remaining callbacks on the queue.
          if (syncQueue !== null) {
            syncQueue = syncQueue.slice(i + 1);
          }
          // Resume flushing in the next tick
          Scheduler_scheduleCallback(
            Scheduler_ImmediatePriority,
            flushSyncCallbackQueue
          );
          throw error;
        } finally {
          isFlushingSyncQueue = false;
        }
      }
    }
```

执行这个方法的时候，我们重点关注一下里面的这个代码块：


```
var _isSync = true;
          var queue = syncQueue;
          // 从所有的同步队列中取出
          runWithPriority(ImmediatePriority, function () {
            for (; i < queue.length; i++) {
              var callback = queue[i];
              do {
                callback = callback(_isSync);
              } while (callback !== null);
            }
          });
          syncQueue = null;
```

这个方法就是从同步队列中取出CallBack的之后，来进行执行回到任务。那这个callBack到底是什么呢？？

我们可以回过去看scheduleUpdateOnFiber的里面的renderRoot方法。

### renderRoot的调用

最主要的一个方法

```
      /**
     * 进行Root节点的渲染
     * 真正算法需要需要调用的renderRoot方法，生成一棵完整的FiberNodeTree 
     * 这个方法里面有两个重要的调用。
     * 一：commitRoot 进行ROOT节点的提交，这是下一个流程
     * 二: prepareFreshStack.我们执行执行刷新操作
     * 然后finishedWork。
     * @param {*} root 
     * @param {*} expirationTime 
     * @param {*} isSync 
     */
    function renderRoot(root, expirationTime, isSync) {
      console.log(
        'ReactNativeRender-dev.js',
        'FMsg:<<<<<<<<<<<<<<<renderRoot<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
      );
      (function () {
        if (!((executionContext & (RenderContext | CommitContext)) === NoContext)) {
          throw ReactError(Error("Should not already be working."));
        }
      })();

      if (enableUserTimingAPI && expirationTime !== Sync) {
        var didExpire = isSync;
        stopRequestCallbackTimer(didExpire);
      }

      if (root.firstPendingTime < expirationTime) {
        // If there's no work left at this expiration time, exit immediately. This
        // happens when multiple callbacks are scheduled for a single root, but an
        // earlier callback flushes the work of a later one.
        return null;
      }

      if (isSync && root.finishedExpirationTime === expirationTime) {
        // There's already a pending commit at this expiration time.
        // TODO: This is poorly factored. This case only exists for the
        // batch.commit() API.
        return commitRoot.bind(null, root);
      }

      flushPassiveEffects();

      // If we have a work-in-progress fiber, it means there's still work to do
      // in this root.
      if (workInProgress !== null) {
        var prevExecutionContext = executionContext;
        executionContext |= RenderContext;
        var prevDispatcher = ReactCurrentDispatcher.current;
        if (prevDispatcher === null) {
          // The React isomorphic package does not include a default dispatcher.
          // Instead the first renderer will lazily attach one, in order to give
          // nicer error messages.
          prevDispatcher = ContextOnlyDispatcher;
        }
        ReactCurrentDispatcher.current = ContextOnlyDispatcher;
        var prevInteractions = null;
        if (enableSchedulerTracing) {
          prevInteractions = tracing.__interactionsRef.current;
          tracing.__interactionsRef.current = root.memoizedInteractions;
        }

        startWorkLoopTimer(workInProgress);

        // TODO: Fork renderRoot into renderRootSync and renderRootAsync
        if (isSync) {
          if (expirationTime !== Sync) {
            // An async update expired. There may be other expired updates on
            // this root. We should render all the expired work in a
            // single batch.
            var currentTime = requestCurrentTime();
            if (currentTime < expirationTime) {
              // Restart at the current time.
              executionContext = prevExecutionContext;
              resetContextDependencies();
              ReactCurrentDispatcher.current = prevDispatcher;
              if (enableSchedulerTracing) {
                tracing.__interactionsRef.current = prevInteractions;
              }
              return renderRoot.bind(null, root, currentTime);
            }
          }
        } else {
          // Since we know we're in a React event, we can clear the current
          // event time. The next update will compute a new event time.
          currentEventTime = NoWork;
        }

        do {
          try {
            console.log(
              'ReactNativeRender-dev.js',
              'FMsg:<<<<<<<<<<<<<<<<workLoopSync<<<<<<<<<<<<' +
              "isSync = " + isSync

            );
            if (isSync) {
              workLoopSync();
            } else {
              workLoop();
            }
            break;
          } catch (thrownValue) {
            // Reset module-level state that was set during the render phase.
            resetContextDependencies();
            resetHooks();

            var sourceFiber = workInProgress;
            if (sourceFiber === null || sourceFiber.return === null) {
              // Expected to be working on a non-root fiber. This is a fatal error
              // because there's no ancestor that can handle it; the root is
              // supposed to capture all errors that weren't caught by an error
              // boundary.
              prepareFreshStack(root, expirationTime);
              executionContext = prevExecutionContext;
              throw thrownValue;
            }

            if (enableProfilerTimer && sourceFiber.mode & ProfileMode) {
              // Record the time spent rendering before an error was thrown. This
              // avoids inaccurate Profiler durations in the case of a
              // suspended render.
              stopProfilerTimerIfRunningAndRecordDelta(sourceFiber, true);
            }

            var returnFiber = sourceFiber.return;
            throwException(
              root,
              returnFiber,
              sourceFiber,
              thrownValue,
              renderExpirationTime
            );
            workInProgress = completeUnitOfWork(sourceFiber);
          }
        } while (true);

        executionContext = prevExecutionContext;
        resetContextDependencies();
        ReactCurrentDispatcher.current = prevDispatcher;
        if (enableSchedulerTracing) {
          tracing.__interactionsRef.current = prevInteractions;
        }

        if (workInProgress !== null) {
          // There's still work left over. Return a continuation.
          stopInterruptedWorkLoopTimer();
          if (expirationTime !== Sync) {
            startRequestCallbackTimer();
          }
          return renderRoot.bind(null, root, expirationTime);
        }
      }

      // We now have a consistent tree. The next step is either to commit it, or, if
      // something suspended, wait to commit it after a timeout.
      stopFinishedWorkLoopTimer();

      root.finishedWork = root.current.alternate;
      root.finishedExpirationTime = expirationTime;

      var isLocked = resolveLocksOnRoot(root, expirationTime);
      if (isLocked) {
        // This root has a lock that prevents it from committing. Exit. If we begin
        // work on the root again, without any intervening updates, it will finish
        // without doing additional work.
        return null;
      }

      // Set this to null to indicate there's no in-progress render.
      workInProgressRoot = null;

      switch (workInProgressRootExitStatus) {
        case RootIncomplete: {
          (function () {
            {
              throw ReactError(Error("Should have a work-in-progress."));
            }
          })();
        }
        // Flow knows about invariant, so it complains if I add a break statement,
        // but eslint doesn't know about invariant, so it complains if I do.
        // eslint-disable-next-line no-fallthrough
        case RootErrored: {
          // An error was thrown. First check if there is lower priority work
          // scheduled on this root.
          var _lastPendingTime = root.lastPendingTime;
          if (_lastPendingTime < expirationTime) {
            // There's lower priority work. Before raising the error, try rendering
            // at the lower priority to see if it fixes it. Use a continuation to
            // maintain the existing priority and position in the queue.
            return renderRoot.bind(null, root, _lastPendingTime);
          }
          if (!isSync) {
            // If we're rendering asynchronously, it's possible the error was
            // caused by tearing due to a mutation during an event. Try rendering
            // one more time without yiedling to events.
            prepareFreshStack(root, expirationTime);
            scheduleSyncCallback(renderRoot.bind(null, root, expirationTime));
            return null;
          }
          // If we're already rendering synchronously, commit the root in its
          // errored state.
          return commitRoot.bind(null, root);
        }
        case RootSuspended: {
          flushSuspensePriorityWarningInDEV();

          // We have an acceptable loading state. We need to figure out if we should
          // immediately commit it or wait a bit.

          // If we have processed new updates during this render, we may now have a
          // new loading state ready. We want to ensure that we commit that as soon as
          // possible.
          var hasNotProcessedNewUpdates =
            workInProgressRootLatestProcessedExpirationTime === Sync;
          if (
            hasNotProcessedNewUpdates &&
            !isSync &&
            // do not delay if we're inside an act() scope
            !(true && flushSuspenseFallbacksInTests && IsThisRendererActing.current)
          ) {
            // If we have not processed any new updates during this pass, then this is
            // either a retry of an existing fallback state or a hidden tree.
            // Hidden trees shouldn't be batched with other work and after that's
            // fixed it can only be a retry.
            // We're going to throttle committing retries so that we don't show too
            // many loading states too quickly.
            var msUntilTimeout =
              globalMostRecentFallbackTime + FALLBACK_THROTTLE_MS - now();
            // Don't bother with a very short suspense time.
            if (msUntilTimeout > 10) {
              if (workInProgressRootHasPendingPing) {
                // This render was pinged but we didn't get to restart earlier so try
                // restarting now instead.
                prepareFreshStack(root, expirationTime);
                return renderRoot.bind(null, root, expirationTime);
              }
              var _lastPendingTime2 = root.lastPendingTime;
              if (_lastPendingTime2 < expirationTime) {
                // There's lower priority work. It might be unsuspended. Try rendering
                // at that level.
                return renderRoot.bind(null, root, _lastPendingTime2);
              }
              // The render is suspended, it hasn't timed out, and there's no lower
              // priority work to do. Instead of committing the fallback
              // immediately, wait for more data to arrive.
              root.timeoutHandle = scheduleTimeout(
                commitRoot.bind(null, root),
                msUntilTimeout
              );
              return null;
            }
          }
          // The work expired. Commit immediately.
          return commitRoot.bind(null, root);
        }
        case RootSuspendedWithDelay: {
          flushSuspensePriorityWarningInDEV();

          if (
            !isSync &&
            // do not delay if we're inside an act() scope
            !(true && flushSuspenseFallbacksInTests && IsThisRendererActing.current)
          ) {
            // We're suspended in a state that should be avoided. We'll try to avoid committing
            // it for as long as the timeouts let us.
            if (workInProgressRootHasPendingPing) {
              // This render was pinged but we didn't get to restart earlier so try
              // restarting now instead.
              prepareFreshStack(root, expirationTime);
              return renderRoot.bind(null, root, expirationTime);
            }
            var _lastPendingTime3 = root.lastPendingTime;
            if (_lastPendingTime3 < expirationTime) {
              // There's lower priority work. It might be unsuspended. Try rendering
              // at that level immediately.
              return renderRoot.bind(null, root, _lastPendingTime3);
            }

            var _msUntilTimeout = void 0;
            if (workInProgressRootLatestSuspenseTimeout !== Sync) {
              // We have processed a suspense config whose expiration time we can use as
              // the timeout.
              _msUntilTimeout =
                expirationTimeToMs(workInProgressRootLatestSuspenseTimeout) - now();
            } else if (workInProgressRootLatestProcessedExpirationTime === Sync) {
              // This should never normally happen because only new updates cause
              // delayed states, so we should have processed something. However,
              // this could also happen in an offscreen tree.
              _msUntilTimeout = 0;
            } else {
              // If we don't have a suspense config, we're going to use a heuristic to
              var eventTimeMs = inferTimeFromExpirationTime(
                workInProgressRootLatestProcessedExpirationTime
              );
              var currentTimeMs = now();
              var timeUntilExpirationMs =
                expirationTimeToMs(expirationTime) - currentTimeMs;
              var timeElapsed = currentTimeMs - eventTimeMs;
              if (timeElapsed < 0) {
                // We get this wrong some time since we estimate the time.
                timeElapsed = 0;
              }

              _msUntilTimeout = jnd(timeElapsed) - timeElapsed;

              // Clamp the timeout to the expiration time.
              // TODO: Once the event time is exact instead of inferred from expiration time
              // we don't need this.
              if (timeUntilExpirationMs < _msUntilTimeout) {
                _msUntilTimeout = timeUntilExpirationMs;
              }
            }

            // Don't bother with a very short suspense time.
            if (_msUntilTimeout > 10) {
              // The render is suspended, it hasn't timed out, and there's no lower
              // priority work to do. Instead of committing the fallback
              // immediately, wait for more data to arrive.
              root.timeoutHandle = scheduleTimeout(
                commitRoot.bind(null, root),
                _msUntilTimeout
              );
              return null;
            }
          }
          // The work expired. Commit immediately.
          return commitRoot.bind(null, root);
        }
        case RootCompleted: {
          // The work completed. Ready to commit.
          if (
            !isSync &&
            // do not delay if we're inside an act() scope
            !(
              true &&
              flushSuspenseFallbacksInTests &&
              IsThisRendererActing.current
            ) &&
            workInProgressRootLatestProcessedExpirationTime !== Sync &&
            workInProgressRootCanSuspendUsingConfig !== null
          ) {
            // If we have exceeded the minimum loading delay, which probably
            // means we have shown a spinner already, we might have to suspend
            // a bit longer to ensure that the spinner is shown for enough time.
            var _msUntilTimeout2 = computeMsUntilSuspenseLoadingDelay(
              workInProgressRootLatestProcessedExpirationTime,
              expirationTime,
              workInProgressRootCanSuspendUsingConfig
            );
            if (_msUntilTimeout2 > 10) {
              root.timeoutHandle = scheduleTimeout(
                commitRoot.bind(null, root),
                _msUntilTimeout2
              );
              return null;
            }
          }
          return commitRoot.bind(null, root);
        }
        default: {
          (function () {
            {
              throw ReactError(Error("Unknown root exit status."));
            }
          })();
        }
      }
    }

```

### prepareFreshStack



```
    function prepareFreshStack(root, expirationTime) {
      console.log(
        'ReactNativeRender-dev.js',
        'FMsg:<<<<<<<<<<<<<<<<prepareFreshStack<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
      );
      root.finishedWork = null;
      root.finishedExpirationTime = NoWork;

      var timeoutHandle = root.timeoutHandle;
      if (timeoutHandle !== noTimeout) {
        // The root previous suspended and scheduled a timeout to commit a fallback
        // state. Now that we have additional work, cancel the timeout.
        root.timeoutHandle = noTimeout;
        // $FlowFixMe Complains noTimeout is not a TimeoutID, despite the check above
        cancelTimeout(timeoutHandle);
      }

      if (workInProgress !== null) {
        var interruptedWork = workInProgress.return;
        while (interruptedWork !== null) {
          unwindInterruptedWork(interruptedWork);
          interruptedWork = interruptedWork.return;
        }
      }
      // 设置当前渲染的工作的流程的节点是ROOT
      workInProgressRoot = root;
      workInProgress = createWorkInProgress(root.current, null, expirationTime);
      renderExpirationTime = expirationTime;
      workInProgressRootExitStatus = RootIncomplete;
      workInProgressRootLatestProcessedExpirationTime = Sync;
      workInProgressRootLatestSuspenseTimeout = Sync;
      workInProgressRootCanSuspendUsingConfig = null;
      workInProgressRootHasPendingPing = false;

      if (enableSchedulerTracing) {
        spawnedWorkDuringRender = null;
      }

      {
        ReactStrictModeWarnings.discardPendingWarnings();
        componentsThatTriggeredHighPriSuspend = null;
      }
    }
```

这个方法最重要的就是就是createWorkInProgress就是创建渲染流程的当前节点。首先，我们将当前的节点设置为root节点。


```
    // This is used to create an alternate fiber to do work on.
    function createWorkInProgress(current, pendingProps, expirationTime) {
      console.log(
        'ReactNativeRender-dev.js',
        'FMsg:<<<<<<<<<<<<<<<<createWorkInProgress<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
      );
      var workInProgress = current.alternate;
      if (workInProgress === null) {
        // We use a double buffering pooling technique because we know that we'll
        // only ever need at most two versions of a tree. We pool the "other" unused
        // node that we're free to reuse. This is lazily created to avoid allocating
        // extra objects for things that are never updated. It also allow us to
        // reclaim the extra memory if needed.
        workInProgress = createFiber(
          current.tag,
          pendingProps,
          current.key,
          current.mode
        );
        workInProgress.elementType = current.elementType;
        workInProgress.type = current.type;
        workInProgress.stateNode = current.stateNode;

        {
          // DEV-only fields
          workInProgress._debugID = current._debugID;
          workInProgress._debugSource = current._debugSource;
          workInProgress._debugOwner = current._debugOwner;
          workInProgress._debugHookTypes = current._debugHookTypes;
        }
        // 这个代码需要注意下
        workInProgress.alternate = current;
        current.alternate = workInProgress;
      } else {
        workInProgress.pendingProps = pendingProps;

        // We already have an alternate.
        // Reset the effect tag.
        workInProgress.effectTag = NoEffect;

        // The effect list is no longer valid.
        workInProgress.nextEffect = null;
        workInProgress.firstEffect = null;
        workInProgress.lastEffect = null;

        if (enableProfilerTimer) {
          // We intentionally reset, rather than copy, actualDuration & actualStartTime.
          // This prevents time from endlessly accumulating in new commits.
          // This has the downside of resetting values for different priority renders,
          // But works for yielding (the common case) and should support resuming.
          workInProgress.actualDuration = 0;
          workInProgress.actualStartTime = -1;
        }
      }

      workInProgress.childExpirationTime = current.childExpirationTime;
      workInProgress.expirationTime = current.expirationTime;

      workInProgress.child = current.child;
      workInProgress.memoizedProps = current.memoizedProps;
      workInProgress.memoizedState = current.memoizedState;
      workInProgress.updateQueue = current.updateQueue;

      // Clone the dependencies object. This is mutated during the render phase, so
      // it cannot be shared with the current fiber.
      var currentDependencies = current.dependencies;
      workInProgress.dependencies =
        currentDependencies === null
          ? null
          : {
            expirationTime: currentDependencies.expirationTime,
            firstContext: currentDependencies.firstContext,
            responders: currentDependencies.responders
          };

      // These will be overridden during the parent's reconciliation
      workInProgress.sibling = current.sibling;
      workInProgress.index = current.index;
      workInProgress.ref = current.ref;

      if (enableProfilerTimer) {
        workInProgress.selfBaseDuration = current.selfBaseDuration;
        workInProgress.treeBaseDuration = current.treeBaseDuration;
      }

      {
        workInProgress._debugNeedsRemount = current._debugNeedsRemount;
        switch (workInProgress.tag) {
          case IndeterminateComponent:
          case FunctionComponent:
          case SimpleMemoComponent:
            workInProgress.type = resolveFunctionForHotReloading(current.type);
            break;
          case ClassComponent:
            workInProgress.type = resolveClassForHotReloading(current.type);
            break;
          case ForwardRef:
            workInProgress.type = resolveForwardRefForHotReloading(current.type);
            break;
          default:
            break;
        }
      }

      return workInProgress;
    }
```

这个时候，我们用于当前刷新的节点，基本已经准备好了。
我们来看一下这个workINProgress的节点的信息

![-w537](https://i.loli.net/2020/04/07/OIrLAZ8x5y2St64.jpg)


这个方法里面，我们需要着重的看一下workLoopSync的这个方法。 

这个方法会进行队列的遍历。会依次取出我们的View树中所有的Element进行调用对应的生命周期方法。


```
        do {
          try {
            console.log(
              'ReactNativeRender-dev.js',
              'FMsg:<<<<<<<<<<<<<<<<workLoopSync<<<<<<<<<<<<' +
              "isSync = " + isSync

            );
            if (isSync) {
              workLoopSync();
            } else {
              workLoop();
            }
            break;
          } catch (thrownValue) {
            // Reset module-level state that was set during the render phase.
            resetContextDependencies();
            resetHooks();

            var sourceFiber = workInProgress;
            if (sourceFiber === null || sourceFiber.return === null) {
              // Expected to be working on a non-root fiber. This is a fatal error
              // because there's no ancestor that can handle it; the root is
              // supposed to capture all errors that weren't caught by an error
              // boundary.
              prepareFreshStack(root, expirationTime);
              executionContext = prevExecutionContext;
              throw thrownValue;
            }

            if (enableProfilerTimer && sourceFiber.mode & ProfileMode) {
              // Record the time spent rendering before an error was thrown. This
              // avoids inaccurate Profiler durations in the case of a
              // suspended render.
              stopProfilerTimerIfRunningAndRecordDelta(sourceFiber, true);
            }

            var returnFiber = sourceFiber.return;
            throwException(
              root,
              returnFiber,
              sourceFiber,
              thrownValue,
              renderExpirationTime
            );
            workInProgress = completeUnitOfWork(sourceFiber);
          }
        } while (true);
```

我们来分析一下workLoopSync这个方法

### workLoopSync调用


```
    /**
     * 这是一个循环队列算法的调度
     */
    function workLoopSync() {
      console.log(
        'ReactNativeRender-dev.js',
        'FMsg:<<<<<<<<<<<<<<<<workLoopSync<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
      );
      // Already timed out, so perform work without checking if we need to yield.
      while (workInProgress !== null) {
        workInProgress = performUnitOfWork(workInProgress);
      }
    }
```

加入到队列中。执行当前刷新的节点的相关逻辑。

### performUnitOfWork调用

```
/**
 * 调用beginWork方法，
 * 生成FiberNode的workInProgress，即current.alternate。
 * @param {*} unitOfWork 
 */
function performUnitOfWork(unitOfWork) {
  console.log(
    'ReactNativeRender-dev.js',
    'FMsg:<<<<<<<<<<<<<<<<performUnitOfWork<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
  );
  // The current, flushed, state of this fiber is the alternate. Ideally
  // nothing should rely on this, but relying on it here means that we don't
  // need an additional field on the work in progress.
  var current$$1 = unitOfWork.alternate;

  startWorkTimer(unitOfWork);
  setCurrentFiber(unitOfWork);
  
  var next = void 0;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    startProfilerTimer(unitOfWork);
    next = beginWork$$1(current$$1, unitOfWork, renderExpirationTime);
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork, true);
  } else {
    next = beginWork$$1(current$$1, unitOfWork, renderExpirationTime);
  }

  resetCurrentFiber();
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  // 这个地方我们的逻辑可以梳理一下。当上面的深度优先的算法，一直到最终的叶子节点
  // next的节点为null 的时候。
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    // 我可以知道，这个方法最终返回的应该是当前节点的兄弟节点
    next = completeUnitOfWork(unitOfWork);
  }

  ReactCurrentOwner$2.current = null;
  return next;
}
```

上面的逻辑，我们可以知道他其实调用beginWork$$1的方法。

### beginWork$$1 调用

我们来看一下beginWork$$1的方法的实现：


```
beginWork$$1 = function(current$$1, unitOfWork, expirationTime) {
    // If a component throws an error, we replay it again in a synchronously
    // dispatched event, so that the debugger will treat it as an uncaught
    // error See ReactErrorUtils for more information.

    // Before entering the begin phase, copy the work-in-progress onto a dummy
    // fiber. If beginWork throws, we'll use this to reset the state.
    var originalWorkInProgressCopy = assignFiberPropertiesInDEV(
      dummyFiber,
      unitOfWork
    );
    try {
      return beginWork$1(current$$1, unitOfWork, expirationTime);
    } catch (originalError) {
      if (
        originalError !== null &&
        typeof originalError === "object" &&
        typeof originalError.then === "function"
      ) {
        // Don't replay promises. Treat everything else like an error.
        throw originalError;
      }

      // Keep this code in sync with renderRoot; any changes here must have
      // corresponding changes there.
      resetContextDependencies();
      resetHooks();

      // Unwind the failed stack frame
      unwindInterruptedWork(unitOfWork);

      // Restore the original properties of the fiber.
      assignFiberPropertiesInDEV(unitOfWork, originalWorkInProgressCopy);

      if (enableProfilerTimer && unitOfWork.mode & ProfileMode) {
        // Reset the profiler timer.
        startProfilerTimer(unitOfWork);
      }

      // Run beginWork again.
      invokeGuardedCallback(
        null,
        beginWork$1,
        null,
        current$$1,
        unitOfWork,
        expirationTime
      );

      if (hasCaughtError()) {
        var replayError = clearCaughtError();
        // `invokeGuardedCallback` sometimes sets an expando `_suppressLogging`.
        // Rethrow this error instead of the original one.
        throw replayError;
      } else {
        // This branch is reachable if the render phase is impure.
        throw originalError;
      }
    }
  };
```

从代码追溯，我们可以看到这个方法调用的

### beginWork$1调用




```
/**
 * 进行Class组件的节点的更新
 * ClassComponent，即我们在写React代码的时候自己写的Component，即例子中的App
 * @param {*} current$$1 
 * @param {*} workInProgress 
 * @param {*} Component 
 * @param {*} nextProps 
 * @param {*} renderExpirationTime 
 */
function updateClassComponent(
  current$$1,
  workInProgress,
  Component,
  nextProps,
  renderExpirationTime
) {
  {
    if (workInProgress.type !== workInProgress.elementType) {
      // Lazy component props can't be validated in createElement
      // because they're only guaranteed to be resolved here.
      var innerPropTypes = Component.propTypes;
      if (innerPropTypes) {
        checkPropTypes(
          innerPropTypes,
          nextProps, // Resolved props
          "prop",
          getComponentName(Component),
          getCurrentFiberStackInDev
        );
      }
    }
  }

  // Push context providers early to prevent context stack mismatches.
  // During mounting we don't know the child context yet as the instance doesn't exist.
  // We will invalidate the child context in finishClassComponent() right after rendering.
  var hasContext = void 0;
  if (isContextProvider(Component)) {
    hasContext = true;
    pushContextProvider(workInProgress);
  } else {
    hasContext = false;
  }
  prepareToReadContext(workInProgress, renderExpirationTime);
  //FiberNode会通过stateNode绑定一些其他的对象，例如ReactComponent实例
  var instance = workInProgress.stateNode;
  var shouldUpdate = void 0;
  if (instance === null) {
    if (current$$1 !== null) {
      // An class component without an instance only mounts if it suspended
      // inside a non- concurrent tree, in an inconsistent state. We want to
      // tree it like a new mount, even though an empty version of it already
      // committed. Disconnect the alternate pointers.
      current$$1.alternate = null;
      workInProgress.alternate = null;
      // Since this is conceptually a new fiber, schedule a Placement effect
      workInProgress.effectTag |= Placement;
    }
    // In the initial pass we might need to construct the instance.
    // 进行ClassComponent的组件实例的构造
    constructClassInstance(
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime
    );
    // 进行ClassComponent的组件的componentWillMount
    mountClassInstance(
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime
    );
    shouldUpdate = true;
  } else if (current$$1 === null) {
    // In a resume, we'll already have an instance we can reuse.
    shouldUpdate = resumeMountClassInstance(
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime
    );
  } else {
    shouldUpdate = updateClassInstance(
      current$$1,
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime
    );
  }
  var nextUnitOfWork = finishClassComponent(
    current$$1,
    workInProgress,
    Component,
    shouldUpdate,
    hasContext,
    renderExpirationTime
  );
  {
    var inst = workInProgress.stateNode;
    if (inst.props !== nextProps) {
      !didWarnAboutReassigningProps
        ? warning$1(
            false,
            "It looks like %s is reassigning its own `this.props` while rendering. " +
              "This is not supported and can lead to confusing bugs.",
            getComponentName(workInProgress.type) || "a component"
          )
        : void 0;
      didWarnAboutReassigningProps = true;
    }
  }
  return nextUnitOfWork;
}
```
我们的ReactNative的生命周期基本都是在这里面完成


### completeUnitOfWork调用 单个节点树的完成

```
function completeUnitOfWork(unitOfWork) {
  console.log(
    'ReactNativeRender-dev.js',
    'FMsg:<<<<<<<<<<<<<<<<completeUnitOfWork<<<<<<<<<<<<<<<<完成组件队列处理'
  );
  // Attempt to complete the current unit of work, then move to the next
  // sibling. If there are no more siblings, return to the parent fiber.
  workInProgress = unitOfWork;

    // ···········省略部分代码
    
    // 这个地方，我们返回的是当前任务节点的兄弟节点
    var siblingFiber = workInProgress.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      return siblingFiber;
    }
    // Otherwise, return to the parent
    workInProgress = returnFiber;
  } while (workInProgress !== null);

  // We've reached the root.
  if (workInProgressRootExitStatus === RootIncomplete) {
    workInProgressRootExitStatus = RootCompleted;
  }
  return null;
}
```


当单个空间树都遍历完成之后，最终方法其实调用到commitRoot

### commitRoot的调用


```
    /**
     * root节点渲染提交
     * @param {} root 
     */
    function commitRoot(root) {
      console.log(
        'ReactNativeRender-dev.js',
        'FMsg:<<<<<<<<<<<<<<<<commitRoot<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<'
      );
      var renderPriorityLevel = getCurrentPriorityLevel();
      // 我们按照由优先级的调度，去执行commitRootImpl
      runWithPriority(
        ImmediatePriority,
        commitRootImpl.bind(null, root, renderPriorityLevel)
      );
      // If there are passive effects, schedule a callback to flush them. This goes
      // outside commitRootImpl so that it inherits the priority of the render.
      if (rootWithPendingPassiveEffects !== null) {
        scheduleCallback(NormalPriority, function () {
          flushPassiveEffects();
          return null;
        });
      }
      return null;
    }
```
### commitRootImpl的调用


```
 // 我们来进行组件的提交
        do {
          {
            invokeGuardedCallback(
              null,
              commitLayoutEffects,
              null,
              root,
              expirationTime
            );
            if (hasCaughtError()) {
              (function () {
                if (!(nextEffect !== null)) {
                  throw ReactError(Error("Should be working on an effect."));
                }
              })();
              var _error2 = clearCaughtError();
              captureCommitPhaseError(nextEffect, _error2);
              nextEffect = nextEffect.nextEffect;
            }
          }
        } while (nextEffect !== null);
```

这个时候开始进行反向的组件的渲染流程。进行componentDidMount的方法的回调

### commitLayoutEffects的调用


```
    function commitLayoutEffects(root, committedExpirationTime) {
      // TODO: Should probably move the bulk of this function to commitWork.
      while (nextEffect !== null) {
        setCurrentFiber(nextEffect);

        var effectTag = nextEffect.effectTag;

        if (effectTag & (Update | Callback)) {
          recordEffect();
          var current$$1 = nextEffect.alternate;
          // 进行组件生命周期的提交
          commitLifeCycles(root, current$$1, nextEffect, committedExpirationTime);
        }

        if (effectTag & Ref) {
          recordEffect();
          commitAttachRef(nextEffect);
        }

        if (effectTag & Passive) {
          rootDoesHavePassiveEffects = true;
        }

        resetCurrentFiber();
        nextEffect = nextEffect.nextEffect;
      }
    }
```

### commitLifeCycles()方法的调用

```
    function commitLifeCycles(
      finishedRoot,
      current$$1,
      finishedWork,
      committedExpirationTime
    ) {
      switch (finishedWork.tag) {
        case FunctionComponent:
        case ForwardRef:
        case SimpleMemoComponent: {
          commitHookEffectList(UnmountLayout, MountLayout, finishedWork);
          break;
        }
        case ClassComponent: {
          // 我们主要看看ClassComponent的组件的提交
          var instance = finishedWork.stateNode;
          if (finishedWork.effectTag & Update) {
            if (current$$1 === null) {
              startPhaseTimer(finishedWork, "componentDidMount");
              // We could update instance props and state here,
              // but instead we rely on them being set during last render.
              // TODO: revisit this when we implement resuming.
              {
                if (
                  finishedWork.type === finishedWork.elementType &&
                  !didWarnAboutReassigningProps
                ) {
                  !(instance.props === finishedWork.memoizedProps)
                    ? warning$1(
                      false,
                      "Expected %s props to match memoized props before " +
                      "componentDidMount. " +
                      "This might either be because of a bug in React, or because " +
                      "a component reassigns its own `this.props`. " +
                      "Please file an issue.",
                      getComponentName(finishedWork.type) || "instance"
                    )
                    : void 0;
                  !(instance.state === finishedWork.memoizedState)
                    ? warning$1(
                      false,
                      "Expected %s state to match memoized state before " +
                      "componentDidMount. " +
                      "This might either be because of a bug in React, or because " +
                      "a component reassigns its own `this.props`. " +
                      "Please file an issue.",
                      getComponentName(finishedWork.type) || "instance"
                    )
                    : void 0;
                }
              }
              // 调用组件实例的componentDidMount
              instance.componentDidMount();
              stopPhaseTimer();
            } 
    }

```


### completeUnitOfWorkde调用

我们继续回到completeUnitOfWorkde的方法调用逻辑中


```
 if (
            returnFiber !== null &&
            // Do not append effects to parents if a sibling failed to complete
            (returnFiber.effectTag & Incomplete) === NoEffect
          ) {
            // Append all the effects of the subtree and this fiber onto the effect
            // list of the parent. The completion order of the children affects the
            // side-effect order.
            if (returnFiber.firstEffect === null) {
              returnFiber.firstEffect = workInProgress.firstEffect;
            }
            if (workInProgress.lastEffect !== null) {
              if (returnFiber.lastEffect !== null) {
                returnFiber.lastEffect.nextEffect = workInProgress.firstEffect;
              }
              returnFiber.lastEffect = workInProgress.lastEffect;
            }

            // If this fiber had side-effects, we append it AFTER the children's
            // side-effects. We can perform certain side-effects earlier if needed,
            // by doing multiple passes over the effect list. We don't want to
            // schedule our own side-effect on our own list because if end up
            // reusing children we'll schedule this effect onto itself since we're
            // at the end.
            var effectTag = workInProgress.effectTag;

            // Skip both NoWork and PerformedWork tags when creating the effect
            // list. PerformedWork effect is read by React DevTools but shouldn't be
            // committed.
            if (effectTag > PerformedWork) {
              if (returnFiber.lastEffect !== null) {
                returnFiber.lastEffect.nextEffect = workInProgress;
              } else {
                returnFiber.firstEffect = workInProgress;
              }
              returnFiber.lastEffect = workInProgress;
            }
          }
        }
```

当前节点的所有子节点都遍历完成之后，会调用completeWork这个方法

### completeWork节点的调用。

我们继续看这个方法下面，回调调用的当前的父节点的兄弟节点


```
   console.log(
                'ReactNativeRender-dev.js',
                'FMsg:<<<<<<<<<<<<<<<<createInstance begin<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<',type
              );
              var _instance6 = createInstance(
                type,
                newProps,
                rootContainerInstance,
                currentHostContext,
                workInProgress
              );
```


### createInstance调用


```
    function createInstance(
      type,
      props,
      rootContainerInstance,
      hostContext,
      internalInstanceHandle
    ) {
      console.log(
        'ReactNativeRender-dev.js',
        'FMsg:<<<<<<<<<<<<<<<<createInstance<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<', type
      );
      var tag = allocateTag();
      var viewConfig = getViewConfigForType(type);

      {
        // 属性的处理
        for (var key in viewConfig.validAttributes) {
          if (props.hasOwnProperty(key)) {
            ReactNativePrivateInterface.deepFreezeAndThrowOnMutationInDev(
              props[key]
            );
          }
        }
      }

      var updatePayload = create(props, viewConfig.validAttributes);
      console.log(
        'ReactNativeRender-dev.js',
        'FMsg:<<<<<<<<<<<<<<<<UIManager.createView<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<', viewConfig.uiViewClassName
      );
      ReactNativePrivateInterface.UIManager.createView(
        tag, // reactTag
        viewConfig.uiViewClassName, // viewName
        rootContainerInstance, // rootTag
        updatePayload // props
      );

      var component = new ReactNativeFiberHostComponent(tag, viewConfig);

      precacheFiberNode(internalInstanceHandle, tag);
      updateFiberProps(tag, props);

      // Not sure how to avoid this cast. Flow is okay if the component is defined
      // in the same file but if it's external it can't see the types.
      return component;
    }
```

这个时候，我们可以看到ReactNativePrivateInterface.UIManager.createView() 这个就调用到原生组件的UIManager的createView的方法里面。关于怎么调用到Java那边。这个我们在ReactNative的通信机制里面会讲到。所以这里我们直接到Java那边。


我们直接看Java那边的代码


```
  @ReactMethod
  public void createView(int tag, String className, int rootViewTag, ReadableMap props) {
    if (DEBUG) {
      String message =
              "(UIManager.createView) tag: " + tag + ", class: " + className + ", props: " + props;
      FLog.d(ReactConstants.TAG, message);
      PrinterHolder.getPrinter().logMessage(ReactDebugOverlayTags.UI_MANAGER, message);
    }
    mUIImplementation.createView(tag, className, rootViewTag, props);
  }
```





```
  /** Invoked by React to create a new node with a given tag, class name and properties. */
  public void createView(int tag, String className, int rootViewTag, ReadableMap props) {
    Log.d(TAG, "FMsg:UI渲染流程第二步(2)：createView() called with: tag = [" + tag + "], className = [" + className + "], rootViewTag = [" + rootViewTag + "], props = [" + props + "]");
    synchronized (uiImplementationThreadLock) {
      ReactShadowNode cssNode = createShadowNode(className);
      ReactShadowNode rootNode = mShadowNodeRegistry.getNode(rootViewTag);
      Assertions.assertNotNull(rootNode, "Root node with tag " + rootViewTag + " doesn't exist");
      cssNode.setReactTag(tag); // Thread safety needed here
      cssNode.setViewClassName(className);
      cssNode.setRootTag(rootNode.getReactTag());
      cssNode.setThemedContext(rootNode.getThemedContext());

      mShadowNodeRegistry.addNode(cssNode);

      ReactStylesDiffMap styles = null;
      if (props != null) {
        styles = new ReactStylesDiffMap(props);
        cssNode.updateProperties(styles);
      }

      handleCreateView(cssNode, rootViewTag, styles);
    }
  }
```






```
  protected void handleCreateView(
      ReactShadowNode cssNode, int rootViewTag, @Nullable ReactStylesDiffMap styles) {
    Log.d(TAG, "FMsg:UI渲染流程第三步(3)：handleCreateView() called with: cssNode = [" + cssNode + "], rootViewTag = [" + rootViewTag + "], styles = [" + styles + "]");
    if (!cssNode.isVirtual()) {
      mNativeViewHierarchyOptimizer.handleCreateView(cssNode, cssNode.getThemedContext(), styles);
    }
  }
```




```
  /** Handles a createView call. May or may not actually create a native view. */
  public void handleCreateView(
      ReactShadowNode node,
      ThemedReactContext themedContext,
      @Nullable ReactStylesDiffMap initialProps) {
    Log.d(TAG, "FMsg:UI渲染流程第四步(4)：handleCreateView() called with: node = [" + node + "], themedContext = [" + themedContext + "], initialProps = [" + initialProps + "]");
    if (!ENABLED) {
      assertNodeSupportedWithoutOptimizer(node);
      int tag = node.getReactTag();
      mUIViewOperationQueue.enqueueCreateView(
          themedContext, tag, node.getViewClass(), initialProps);
      return;
    }

    boolean isLayoutOnly =
        node.getViewClass().equals(ViewProps.VIEW_CLASS_NAME)
            && isLayoutOnlyAndCollapsable(initialProps);
    node.setIsLayoutOnly(isLayoutOnly);

    if (node.getNativeKind() != NativeKind.NONE) {
      mUIViewOperationQueue.enqueueCreateView(
          themedContext, node.getReactTag(), node.getViewClass(), initialProps);
    }
  }
```




```
  public void enqueueCreateView(
      ThemedReactContext themedContext,
      int viewReactTag,
      String viewClassName,
      @Nullable ReactStylesDiffMap initialProps) {
    Log.d(TAG, "FMsg:UI渲染流程第五步(5)：enqueueCreateView() called with: themedContext = [" + themedContext + "], viewReactTag = [" + viewReactTag + "], viewClassName = [" + viewClassName + "], initialProps = [" + initialProps + "]");
    synchronized (mNonBatchedOperationsLock) {
      mNonBatchedOperations.addLast(
          new CreateViewOperation(themedContext, viewReactTag, viewClassName, initialProps));
    }
  }
```

这里我们把它包装成了一个Operations的类。然后放在队列中的里面。可以想象我们需要把这些Operations取出来，然后执行我们的生成View渲染View的相关操作。

我们可以看UIViewOperationQueue这里面的关于mNonBatchedOperations的取用。


```
  /**
     * 调用View的操作的分发
     * @param frameTimeNanos
     */
    private void dispatchPendingNonBatchedOperations(long frameTimeNanos) {
      while (true) {
        long timeLeftInFrame = FRAME_TIME_MS - ((System.nanoTime() - frameTimeNanos) / 1000000);
        if (timeLeftInFrame < mMinTimeLeftInFrameForNonBatchedOperationMs) {
          break;
        }

        UIOperation nextOperation;
        synchronized (mNonBatchedOperationsLock) {
          if (mNonBatchedOperations.isEmpty()) {
            break;
          }
          // 我们可以清楚的看到，所有关于View的操作都是从这里面从队列里面出去来
          nextOperation = mNonBatchedOperations.pollFirst();
          Log.d(TAG, "FMsg:dispatchPendingNonBatchedOperations() called with: nextOperation = [" + nextOperation + "]");
        }

        try {
          long nonBatchedExecutionStartTime = SystemClock.uptimeMillis();
          nextOperation.execute();
          mNonBatchedExecutionTotalTime +=
              SystemClock.uptimeMillis() - nonBatchedExecutionStartTime;
        } catch (Exception e) {
          mIsInIllegalUIState = true;
          throw e;
        }
      }
    }
```

我们来看一下这个逻辑是谁那边来调用的。


```
    @Override
    public void doFrameGuarded(long frameTimeNanos) {
      if (mIsInIllegalUIState) {
        FLog.w(
            ReactConstants.TAG,
            "Not flushing pending UI operations because of previously thrown Exception");
        return;
      }

      Systrace.beginSection(Systrace.TRACE_TAG_REACT_JAVA_BRIDGE, "dispatchNonBatchedUIOperations");
      try {
        dispatchPendingNonBatchedOperations(frameTimeNanos);
      } finally {
        Systrace.endSection(Systrace.TRACE_TAG_REACT_JAVA_BRIDGE);
      }

      flushPendingBatches();

      ReactChoreographer.getInstance()
          .postFrameCallback(ReactChoreographer.CallbackType.DISPATCH_UI, this);
    }
```



```
public abstract class GuardedFrameCallback extends ChoreographerCompat.FrameCallback {
  private static final String TAG = "GuardedFrameCallback";
  private final ReactContext mReactContext;

  protected GuardedFrameCallback(ReactContext reactContext) {
    mReactContext = reactContext;
  }

  @Override
  public final void doFrame(long frameTimeNanos) {
    //Log.d(TAG, "FMsg:doFrame() called with: frameTimeNanos = [" + frameTimeNanos + "]");
    try {
      doFrameGuarded(frameTimeNanos);
    } catch (RuntimeException e) {
      mReactContext.handleException(e);
    }
  }

  /**
   * Like the standard doFrame but RuntimeExceptions will be caught and passed to {@link
   * com.facebook.react.bridge.ReactContext#handleException(RuntimeException)}.
   */
  protected abstract void doFrameGuarded(long frameTimeNanos);
}
```
看到这个地方就一目了然了。这个UIViewOperrations队列是由Choreographer来进行驱动取用的。

那么队列的任务取出来之后，怎么进行任务的处理的呢？

我们回过头去看dispatchPendingNonBatchedOperations这个方法的实现。


```
synchronized (mNonBatchedOperationsLock) {
          if (mNonBatchedOperations.isEmpty()) {
            break;
          }
          // 我们可以清楚的看到，所有关于View的操作都是从这里面从队列里面出去来
          nextOperation = mNonBatchedOperations.pollFirst();
          Log.d(TAG, "FMsg:dispatchPendingNonBatchedOperations() called with: nextOperation = [" + nextOperation + "]");
        }

        try {
          long nonBatchedExecutionStartTime = SystemClock.uptimeMillis();
          nextOperation.execute();
          mNonBatchedExecutionTotalTime +=
              SystemClock.uptimeMillis() - nonBatchedExecutionStartTime;
        } catch (Exception e) {
          mIsInIllegalUIState = true;
          throw e;
        }
      }
```

我们看，针对UIOperation的批处理的数据，分别执行execute方法。

ViewOperation的抽象类，有很多的继承类。分别代码VIew的不同的操作方法。这里我们主要分析CreateViewOperation的代码实现。


```
  private final class CreateViewOperation extends ViewOperation {

    private final ThemedReactContext mThemedContext;
    private final String mClassName;
    private final @Nullable ReactStylesDiffMap mInitialProps;

    public CreateViewOperation(
        ThemedReactContext themedContext,
        int tag,
        String className,
        @Nullable ReactStylesDiffMap initialProps) {
      super(tag);
      mThemedContext = themedContext;
      mClassName = className;
      mInitialProps = initialProps;
      Systrace.startAsyncFlow(Systrace.TRACE_TAG_REACT_VIEW, "createView", mTag);
    }

    @Override
    public void execute() {
      Systrace.endAsyncFlow(Systrace.TRACE_TAG_REACT_VIEW, "createView", mTag);
      Log.d(TAG, "FMsg:渲染流程第六步：!!!!!!!!!!!!!mNativeViewHierarchyManager createView !!!!!!!!!!!!");
      mNativeViewHierarchyManager.createView(mThemedContext, mTag, mClassName, mInitialProps);
    }
  }
```


我们可以看到execute的方法其实很简单。直接调用的NativeViewHierarchyManager的createView


```
  public synchronized void createView(
      ThemedReactContext themedContext,
      int tag,
      String className,
      @Nullable ReactStylesDiffMap initialProps) {
    UiThreadUtil.assertOnUiThread();
    SystraceMessage.beginSection(
            Systrace.TRACE_TAG_REACT_VIEW, "NativeViewHierarchyManager_createView")
        .arg("tag", tag)
        .arg("className", className)
        .flush();
    try {
      ViewManager viewManager = mViewManagers.get(className);
      Log.d(TAG, "FMsg:渲染流程第七步：createView() called with: themedContext = [" + themedContext + "], tag = [" + tag + "], className = [" + className + "], initialProps = [" + initialProps + "]");
      View view = viewManager.createView(themedContext, null, null, mJSResponderHandler);
      Log.d(TAG, "FMsg:渲染流程第七步：createView() view === " + view.toString());
      // 从这个整个渲染流程的就已经结束了！！！。但是我们还需要学习。既然View的已经存入到mTagsToViews
      // 那么下面我们思考一下，我们是什么时候把他取出来。然后add 到ReactRootView中
      mTagsToViews.put(tag, view);
      mTagsToViewManagers.put(tag, viewManager);

      // Use android View id field to store React tag. This is possible since we don't inflate
      // React views from layout xmls. Thus it is easier to just reuse that field instead of
      // creating another (potentially much more expensive) mapping from view to React tag
      view.setId(tag);
      if (initialProps != null) {
        viewManager.updateProperties(view, initialProps);
      }
    } finally {
      Systrace.endSection(Systrace.TRACE_TAG_REACT_VIEW);
    }
  }
```
从这个整个渲染流程的就已经结束了！！！。但是我们还需要学习。既然View的已经存入到mTagsToViews

下面我们贴一张图，来总结一下上面的流程：

![](https://i.loli.net/2020/04/09/iV8kOUPZaIxc3lj.jpg)


那么下面我们思考一下，我们是什么时候把他取出来。然后add 到ReactRootView中??
