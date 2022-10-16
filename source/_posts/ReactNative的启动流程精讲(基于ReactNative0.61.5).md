---
title: ReactNative的启动流程精讲(基于ReactNative0.61.5)
date: 2020-03-13 21:00:02
tags: [ReactNative,Android]
---

# ReactNative的启动流程精讲(基于ReactNative0.61.5)

文章参考：https://github.com/sucese/react-native/blob/master/doc/ReactNative%E6%BA%90%E7%A0%81%E7%AF%87/3ReactNative%E6%BA%90%E7%A0%81%E7%AF%87%EF%BC%9A%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.md

文章参考：https://www.jianshu.com/p/baff68f85d41



### 创建ReactNativeHost

ReactNative的启动流程，我们首先看Application中的代码，实例化ReactNativeHost。ReactNativeHost主要的工作就是创建了ReactInstanceManager，它将一些信息传递给了ReactInstanceManager。同时，我们看一下ReactNativeHost的提供的方法。

```
public abstract class ReactNativeHost {
   
      protected ReactInstanceManager createReactInstanceManager() {
        ReactInstanceManagerBuilder builder = ReactInstanceManager.builder()
          //应用上下文
          .setApplication(mApplication)
          //JSMainModuleP相当于应用首页的js Bundle，可以传递url从服务器拉取js Bundle
          //当然这个只在dev模式下可以使用
          .setJSMainModulePath(getJSMainModuleName())
          //是否开启dev模式
          .setUseDeveloperSupport(getUseDeveloperSupport())
          //红盒的回调
          .setRedBoxHandler(getRedBoxHandler())
          //自定义UI实现机制，这个我们一般用不到
          .setUIImplementationProvider(getUIImplementationProvider())
          .setInitialLifecycleState(LifecycleState.BEFORE_CREATE);
    
        //添加ReactPackage
        for (ReactPackage reactPackage : getPackages()) {
          builder.addPackage(reactPackage);
        }
    
        //获取js Bundle的加载路径
        String jsBundleFile = getJSBundleFile();
        if (jsBundleFile != null) {
          builder.setJSBundleFile(jsBundleFile);
        } else {
          builder.setBundleAssetName(Assertions.assertNotNull(getBundleAssetName()));
        }
        return builder.build();
      }
}
```
下面是ReactNativeHost提供的一些方法：

```
protected @Nullable RedBoxHandler getRedBoxHandler();
protected @Nullable JavaScriptExecutorFactory getJavaScriptExecutorFactory();
protected final Application getApplication();
protected UIImplementationProvider getUIImplementationProvider();

```


### 实例化ReactActivityDelegate：
启动ReactActivity的子类实例，在这个Activty的启动流程中，正式开始我们ReactNative启动和加载流程。
我们先来分析一下ReactActivityDelegate。我么可以看到ReactActivityDelegate的一些提供的方法。就知道这个类的一些职责。他主要就是Activity的宿主的一些生命周期的委托调用。也就是所有ReactNative和宿主Activity的关联都是通过这个类类简历联系的。

当然我们也看到。其实所有的生命周期的方法。这个类要么委托给ReactDelegate。要么就是直接ReactInstanceManager进行处理。所以这个类的职责也是非常简单的。

可以说ReactNative和她的宿主Activity达到了很好的解耦。

```
public abstract class ReactActivity extends AppCompatActivity
    implements DefaultHardwareBackBtnHandler, PermissionAwareActivity {

  private final ReactActivityDelegate mDelegate;

  protected ReactActivity() {
    mDelegate = createReactActivityDelegate();
  }

  /**
   * Returns the name of the main component registered from JavaScript. This is used to schedule
   * rendering of the component. e.g. "MoviesApp"
   */
  protected @Nullable String getMainComponentName() {
    return null;
  }

  /** Called at construction time, override if you have a custom delegate implementation. */
  protected ReactActivityDelegate createReactActivityDelegate() {
    return new ReactActivityDelegate(this, getMainComponentName());
  }
  .......
}
```
这个类很简单，就是实例化的一个Activity的委托对象。然后所有的生命周期调用全部使用ReactActivityDelegate的代理对象完成。

那么下面，我们看一下这个类的实现。


```
public ReactActivityDelegate(ReactActivity activity, @Nullable String mainComponentName) {
    Log.d(TAG, "第二步：FMsg: 实例化ReactActivityDelegate called with: activity = [" + activity + "], mainComponentName = [" + mainComponentName + "]");
    mActivity = activity;
    mMainComponentName = mainComponentName;
  }
```
我们看一下。Activity的onCreate的方法的代理实现。

### 实例化ReactDelegate

```
protected void onCreate(Bundle savedInstanceState) {
    String mainComponentName = getMainComponentName();
    mReactDelegate =
        new ReactDelegate(
            getPlainActivity(), getReactNativeHost(), mainComponentName, getLaunchOptions()) {
          @Override
          protected ReactRootView createRootView() {
            return ReactActivityDelegate.this.createRootView();
          }
        };
    //mMainComponentName就是上面ReactActivity.getMainComponentName()返回的组件名    
    if (mMainComponentName != null) {
      //载入app页面
      loadApp(mainComponentName);
    }
  }
```

这个方法一个，实例化ReactDelegate对象。二是调用loadApp的方法。

我们来看一下ReactDelegate的这个对象。他的职责到底是什么呢？？



```
protected void loadApp(String appKey) {
    mReactDelegate.loadApp(appKey);
    getPlainActivity().setContentView(mReactDelegate.getReactRootView());
  }
```
这个方法很简单。还是调用了ReactDelegate的loadApp方法。至此位置ReactActivityDelegate的启动流程中的职责也就告一段落。下面我们看一下ReactDelegate里面的loadApp

```
  public void loadApp(String appKey) {
    if (mReactRootView != null) {
      throw new IllegalStateException("Cannot loadApp while app is already running.");
    }
    //创建ReactRootView作为根视图,它本质上是一个FrameLayout
    mReactRootView = createRootView();、
    //启动RN应用.这个地方我们要注意下。ReactInstanceManager就是从这个地方开始实例化的。
    mReactRootView.startReactApplication(
        getReactNativeHost().getReactInstanceManager(), appKey, mLaunchOptions);
  }
```

最终这个方法是调用到ReactRootView。这个View我们知道，其实就是我们通过setContentView设置给Activity宿主的FameLayout


### 实例化ReactInstanceManager

我们可以从代码中看到，在startReactApplication的时候，我们通过ReactNativeHost的实例化对象来获取ReactInstanceManager


```
 /** Get the current {@link ReactInstanceManager} instance, or create one. */
  public ReactInstanceManager getReactInstanceManager() {
    if (mReactInstanceManager == null) {
      ReactMarker.logMarker(ReactMarkerConstants.GET_REACT_INSTANCE_MANAGER_START);
      mReactInstanceManager = createReactInstanceManager();
      ReactMarker.logMarker(ReactMarkerConstants.GET_REACT_INSTANCE_MANAGER_END);
    }
    return mReactInstanceManager;
  }
  
  
  protected ReactInstanceManager createReactInstanceManager() {
    ReactMarker.logMarker(ReactMarkerConstants.BUILD_REACT_INSTANCE_MANAGER_START);
    Log.d(TAG, "FMsg:第五步：createReactInstanceManager() called");
    ReactInstanceManagerBuilder builder =
        ReactInstanceManager.builder()
            .setApplication(mApplication)
             // 设置JSMain的根路径地址。也就是index.js  //"index.android"
            .setJSMainModulePath(getJSMainModuleName())
            .setUseDeveloperSupport(getUseDeveloperSupport())
            //红盒的回调
            .setRedBoxHandler(getRedBoxHandler())
            // 获取JS执行引擎的工作累的方法。默认是JSC。当然我们也可以自定义设置成V8的引擎
            // 如果需要自定义设置，那么Application中重写这个方法
            .setJavaScriptExecutorFactory(getJavaScriptExecutorFactory())
             //自定义UI实现机制，这个我们一般用不到
            .setUIImplementationProvider(getUIImplementationProvider())
            .setJSIModulesPackage(getJSIModulePackage())
            .setInitialLifecycleState(LifecycleState.BEFORE_CREATE);
    // 类实例化的作用就是传递给ReactInstanceManager所有的Packages
    Log.i(TAG, "FMsg:第五步：createReactInstanceManager: addPackage  size = " + getPackages().size());
    // 这个使我们Application实例化的时候，我们提供的ReactPackage
    for (ReactPackage reactPackage : getPackages()) {
      builder.addPackage(reactPackage);
    }

    String jsBundleFile = getJSBundleFile();
    Log.i(TAG, "FMsg:第五步：createReactInstanceManager: jsBundleFile = "+jsBundleFile);
    if (jsBundleFile != null) {
      builder.setJSBundleFile(jsBundleFile);
    } else {
      builder.setBundleAssetName(Assertions.assertNotNull(getBundleAssetName()));
    }
    // 这个建造者模式，可以好好分析一下。里面有很多的默认参数
    ReactInstanceManager reactInstanceManager = builder.build();
    ReactMarker.logMarker(ReactMarkerConstants.BUILD_REACT_INSTANCE_MANAGER_END);
    return reactInstanceManager;
  }

 
```
其实从这个地方。ReactNativeHost的历史使命就完成了。所有我们复写的他的提供方法都是在这个地方传递给ReactInstance的构造化实例对象。

下面，我们来看一下ReactInstanceManager的构造函数


```
 /* package */ ReactInstanceManager(
      Context applicationContext,
      @Nullable Activity currentActivity,
      @Nullable DefaultHardwareBackBtnHandler defaultHardwareBackBtnHandler,
      JavaScriptExecutorFactory javaScriptExecutorFactory,
      @Nullable JSBundleLoader bundleLoader,
      @Nullable String jsMainModulePath,
      List<ReactPackage> packages,
      boolean useDeveloperSupport,
      @Nullable NotThreadSafeBridgeIdleDebugListener bridgeIdleDebugListener,
      LifecycleState initialLifecycleState,
      @Nullable UIImplementationProvider mUIImplementationProvider,
      NativeModuleCallExceptionHandler nativeModuleCallExceptionHandler,
      @Nullable RedBoxHandler redBoxHandler,
      boolean lazyViewManagersEnabled,
      @Nullable DevBundleDownloadListener devBundleDownloadListener,
      int minNumShakes,
      // 这个参数也比较重要，这个参数就是ReactNative的刷新帧的频率。默认-1.表示直接使用Android的刷新帧频率
      int minTimeLeftInFrameForNonBatchedOperationMs,
      @Nullable JSIModulePackage jsiModulePackage,
      @Nullable Map<String, RequestHandler> customPackagerCommandHandlers) {
    Log.d(ReactConstants.TAG, "ReactInstanceManager.ctor()");
    Log.d(TAG, "FMsg:第五步：ReactInstanceManager() called with: currentActivity = [" + currentActivity + "], defaultHardwareBackBtnHandler = [" + defaultHardwareBackBtnHandler + "], javaScriptExecutorFactory = [" + javaScriptExecutorFactory + "], bundleLoader = [" + bundleLoader + "], jsMainModulePath = [" + jsMainModulePath + "], packages = [" + packages + "], useDeveloperSupport = [" + useDeveloperSupport + "], bridgeIdleDebugListener = [" + bridgeIdleDebugListener + "], initialLifecycleState = [" + initialLifecycleState + "], mUIImplementationProvider = [" + mUIImplementationProvider + "], nativeModuleCallExceptionHandler = [" + nativeModuleCallExceptionHandler + "], redBoxHandler = [" + redBoxHandler + "], lazyViewManagersEnabled = [" + lazyViewManagersEnabled + "], devBundleDownloadListener = [" + devBundleDownloadListener + "], minNumShakes = [" + minNumShakes + "], minTimeLeftInFrameForNonBatchedOperationMs = [" + minTimeLeftInFrameForNonBatchedOperationMs + "], jsiModulePackage = [" + jsiModulePackage + "], customPackagerCommandHandlers = [" + customPackagerCommandHandlers + "]");
    initializeSoLoaderIfNecessary(applicationContext);

    DisplayMetricsHolder.initDisplayMetricsIfNotInitialized(applicationContext);

    mApplicationContext = applicationContext;
    mCurrentActivity = currentActivity;
    mDefaultBackButtonImpl = defaultHardwareBackBtnHandler;
    // 其实默认传入的null  如果我们想使用其他JS引擎，比如V8.我们可以在这里面进行传入
    mJavaScriptExecutorFactory = javaScriptExecutorFactory;
    mBundleLoader = bundleLoader;
    mJSMainModulePath = jsMainModulePath;
    mPackages = new ArrayList<>();
    mUseDeveloperSupport = useDeveloperSupport;
    Systrace.beginSection(
        Systrace.TRACE_TAG_REACT_JAVA_BRIDGE, "ReactInstanceManager.initDevSupportManager");
    mDevSupportManager =
        DevSupportManagerFactory.create(
            applicationContext,
            createDevHelperInterface(),
            mJSMainModulePath,
            useDeveloperSupport,
            redBoxHandler,
            devBundleDownloadListener,
            minNumShakes,
            customPackagerCommandHandlers);
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
    mBridgeIdleDebugListener = bridgeIdleDebugListener;
    mLifecycleState = initialLifecycleState;
    mMemoryPressureRouter = new MemoryPressureRouter(applicationContext);
    mNativeModuleCallExceptionHandler = nativeModuleCallExceptionHandler;

    /// 这个地方就是添加ReactPackage
    synchronized (mPackages) {
      PrinterHolder.getPrinter()
          .logMessage(ReactDebugOverlayTags.RN_CORE, "RNCore: Use Split Packages");
      // 这个方法很重要，这个方法我们会将系统内置的那些CoreModulesPackage添加进去
      // AndroidInfoModule.class,
      // DeviceEventManagerModule.class,
      // DeviceInfoModule.class,
      // DevSettingsModule.class,
      // ExceptionsManagerModule.class,
      // HeadlessJsTaskSupportModule.class,
      // SourceCodeModule.class,
      // Timing.class,
      // UIManagerModule.class
      mPackages.add(
          new CoreModulesPackage(
              this,
              new DefaultHardwareBackBtnHandler() {
                @Override
                public void invokeDefaultOnBackPressed() {
                  ReactInstanceManager.this.invokeDefaultOnBackPressed();
                }
              },
              mUIImplementationProvider,
              lazyViewManagersEnabled,
              minTimeLeftInFrameForNonBatchedOperationMs));
      if (mUseDeveloperSupport) {
        mPackages.add(new DebugCorePackage());
      }
      mPackages.addAll(packages);
    }
    mJSIModulePackage = jsiModulePackage;

    // Instantiate ReactChoreographer in UI thread.
    // 这个就是实例化Android的序列帧的监听对象
    ReactChoreographer.initialize();
    if (mUseDeveloperSupport) {
      mDevSupportManager.startInspector();
    }
  }
```

从ReactInstanceManager的构造函数，我们可以看到。所有ReactNative的启动的初始化资源基本都已经准备完毕。所以startReactApplication的方法。我们必须要传入ReactInstanceManager实例化对象。

其实通过查看ReactInstance的类的内容，我们可以看到这个类的方法其实也是比较简单的。他的关于启动流程最大的一个作用就是创建ReactContext。那么这个创建ReactContext是什么开始的呢？

我们来继续看ReactRootView的startReactApplication方法。

### startReactApplication

```
  @ThreadConfined(UI)
  public void startReactApplication(
      ReactInstanceManager reactInstanceManager,
      String moduleName,
      @Nullable Bundle initialProperties,
      @Nullable String initialUITemplate) {
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "startReactApplication");
    try {
      // 线程检查
      UiThreadUtil.assertOnUiThread();

      // TODO(6788889): Use POJO instead of bundle here, apparently we can't just use WritableMap
      // here as it may be deallocated in native after passing via JNI bridge, but we want to reuse
      // it in the case of re-creating the catalyst instance
      Assertions.assertCondition(
          mReactInstanceManager == null,
          "This root view has already been attached to a catalyst instance manager");

      mReactInstanceManager = reactInstanceManager;
      mJSModuleName = moduleName;
      mAppProperties = initialProperties;
      mInitialUITemplate = initialUITemplate;

      if (mUseSurface) {
        // TODO initialize surface here
      }
      //创建RN上下文上下文对象
      mReactInstanceManager.createReactContextInBackground();
      //attachToReactInstanceManager 调用的是mReactInstanceManager.attachRootView(this)
      attachToReactInstanceManager();

    } finally {
      Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
    }
  }
```
这个方法比较深，我们可以一步步往下看。进入到这个方法，就进入到ReactInstance的核心功能。也是ReactNative启动的核心流程。也是最重要的流程中。createReactContext的创建流程。

```
@ThreadConfined(UI)
  public void createReactContextInBackground() {
    Log.d(ReactConstants.TAG, "ReactInstanceManager.createReactContextInBackground()");
    Log.d(TAG, "FMsg:第六步：ReactRootView开始创建ReactContext createReactContextInBackground() called");
    UiThreadUtil
        .assertOnUiThread(); // Assert before setting mHasStartedCreatingInitialContext = true
    if (!mHasStartedCreatingInitialContext) {
      mHasStartedCreatingInitialContext = true;
      recreateReactContextInBackgroundInner();
    }
  }
  
  
   @ThreadConfined(UI)
  private void recreateReactContextInBackgroundInner() {
    Log.d(ReactConstants.TAG, "ReactInstanceManager.recreateReactContextInBackgroundInner()");
    Log.d(TAG, "FMsg:第六步：调用内部方法recreateReactContextInBackgroundInner() called");
    PrinterHolder.getPrinter()
        .logMessage(ReactDebugOverlayTags.RN_CORE, "RNCore: recreateReactContextInBackground");
    UiThreadUtil.assertOnUiThread();
    // 调试相关的逻辑，咱们暂时忽略
    if (mUseDeveloperSupport && mJSMainModulePath != null) {
      final DeveloperSettings devSettings = mDevSupportManager.getDevSettings();

      if (!Systrace.isTracing(TRACE_TAG_REACT_APPS | TRACE_TAG_REACT_JS_VM_CALLS)) {
        if (mBundleLoader == null) {
          mDevSupportManager.handleReloadJS();
        } else {
          mDevSupportManager.isPackagerRunning(
              new PackagerStatusCallback() {
                @Override
                public void onPackagerStatusFetched(final boolean packagerIsRunning) {
                  UiThreadUtil.runOnUiThread(
                      new Runnable() {
                        @Override
                        public void run() {
                          if (packagerIsRunning) {
                            mDevSupportManager.handleReloadJS();
                          } else if (mDevSupportManager.hasUpToDateJSBundleInCache()
                              && !devSettings.isRemoteJSDebugEnabled()) {
                            // If there is a up-to-date bundle downloaded from server,
                            // with remote JS debugging disabled, always use that.
                            onJSBundleLoadedFromServer(null);
                          } else {
                            // If dev server is down, disable the remote JS debugging.
                            devSettings.setRemoteJSDebugEnabled(false);
                            recreateReactContextInBackgroundFromBundleLoader();
                          }
                        }
                      });
                }
              });
        }
        return;
      }
    }

    recreateReactContextInBackgroundFromBundleLoader();
  }
  
  
   @ThreadConfined(UI)
  private void recreateReactContextInBackgroundFromBundleLoader() {
    Log.d(
        ReactConstants.TAG,
        "ReactInstanceManager.recreateReactContextInBackgroundFromBundleLoader()");
    Log.d(TAG, "FMsg:第六步：线上环境来将会使用后台任务创建ReactContext recreateReactContextInBackgroundFromBundleLoader() called");
    PrinterHolder.getPrinter()
        .logMessage(ReactDebugOverlayTags.RN_CORE, "RNCore: load from BundleLoader");
    recreateReactContextInBackground(mJavaScriptExecutorFactory, mBundleLoader);
  }
  
  
    @ThreadConfined(UI)
  private void runCreateReactContextOnNewThread(final ReactContextInitParams initParams) {
    Log.d(ReactConstants.TAG, "ReactInstanceManager.runCreateReactContextOnNewThread()");
    Log.d(TAG, "FMsg:第六步：runCreateReactContextOnNewThread() called with: initParams = [" + initParams + "]");
    UiThreadUtil.assertOnUiThread();
    synchronized (mAttachedReactRoots) {
      synchronized (mReactContextLock) {
        // 生命周期的回收，主要是针对可能会造成的多次初始化RN环境的
        if (mCurrentReactContext != null) {
          tearDownReactContext(mCurrentReactContext);
          mCurrentReactContext = null;
        }
      }
    }

    mCreateReactContextThread =
        new Thread(
            null,
            new Runnable() {
              @Override
              public void run() {
                ReactMarker.logMarker(REACT_CONTEXT_THREAD_END);
                synchronized (ReactInstanceManager.this.mHasStartedDestroying) {
                  while (ReactInstanceManager.this.mHasStartedDestroying) {
                    try {
                      ReactInstanceManager.this.mHasStartedDestroying.wait();
                    } catch (InterruptedException e) {
                      continue;
                    }
                  }
                }
                // As destroy() may have run and set this to false, ensure that it is true before we
                // create
                mHasStartedCreatingInitialContext = true;

                try {
                  Process.setThreadPriority(Process.THREAD_PRIORITY_DISPLAY);
                  ReactMarker.logMarker(VM_INIT);
                  Log.i(TAG, "FMsg:第六步：runCreateReactContextOnNewThread run: " + Thread.currentThread().getName());
                  final ReactApplicationContext reactApplicationContext =
                      createReactContext(
                          initParams.getJsExecutorFactory().create(),
                          initParams.getJsBundleLoader());

                  mCreateReactContextThread = null;
                  ReactMarker.logMarker(PRE_SETUP_REACT_CONTEXT_START);

                  // 这个地方也是一个保护机制，我们暂时不需要关注
                  final Runnable maybeRecreateReactContextRunnable =
                      new Runnable() {
                        @Override
                        public void run() {
                          if (mPendingReactContextInitParams != null) {
                            runCreateReactContextOnNewThread(mPendingReactContextInitParams);
                            mPendingReactContextInitParams = null;
                          }
                        }
                      };
                  Runnable setupReactContextRunnable =
                      new Runnable() {
                        @Override
                        public void run() {
                          try {
                            setupReactContext(reactApplicationContext);
                          } catch (Exception e) {
                            mDevSupportManager.handleException(e);
                          }
                        }
                      };

                  reactApplicationContext.runOnNativeModulesQueueThread(setupReactContextRunnable);
                  UiThreadUtil.runOnUiThread(maybeRecreateReactContextRunnable);
                } catch (Exception e) {
                  mDevSupportManager.handleException(e);
                }
              }
            },
            "create_react_context");
    ReactMarker.logMarker(REACT_CONTEXT_THREAD_START);
    mCreateReactContextThread.start();
  }
```

这个地方就是我们比较重要的创建ReactContext对象的逻辑。

```
   /** @return instance of {@link ReactContext} configured a {@link CatalystInstance} set */
  private ReactApplicationContext createReactContext(
      JavaScriptExecutor jsExecutor, JSBundleLoader jsBundleLoader) {
    Log.d(TAG, "FMsg:第六步：ReactInstanceManager.createReactContext " + Thread.currentThread().getName());
    Log.d(TAG, "FMsg:第六步：createReactContext() called with: jsExecutor = [" + jsExecutor + "], jsBundleLoader = [" + jsBundleLoader + "]");
    ReactMarker.logMarker(CREATE_REACT_CONTEXT_START, jsExecutor.getName());
    // 这个地方就进行了实例化reactContext
    // ReactApplicationContext extends ReactContext extends ContextWrapper
    // 构造函数传入的是Context。 eactApplicationContext是ReactContext的包装类。
    final ReactApplicationContext reactContext = new ReactApplicationContext(mApplicationContext);

    // 这个就是NativeModuleCallExceptionHandler.异常捕获器
    NativeModuleCallExceptionHandler exceptionHandler =
        mNativeModuleCallExceptionHandler != null
            ? mNativeModuleCallExceptionHandler
            : mDevSupportManager;
    reactContext.setNativeModuleCallExceptionHandler(exceptionHandler);

    // 创建JavaModule注册表Builder，用来创建JavaModule注册表，JavaModule注册表将所有的JavaModule注册到CatalystInstance中。
    NativeModuleRegistry nativeModuleRegistry = processPackages(reactContext, mPackages, false);
    // 最最重要的一个对象，催化器实例对象
    // jsExecutor、nativeModuleRegistry、nativeModuleRegistry等各种参数处理好之后，开始构建CatalystInstanceImpl实例。
    CatalystInstanceImpl.Builder catalystInstanceBuilder =
        new CatalystInstanceImpl.Builder()
                // 设置ReactNative的消息队列的配置数据
            .setReactQueueConfigurationSpec(ReactQueueConfigurationSpec.createDefault())
                // 通过JS执行器工厂实例化的JS线程执行器
            .setJSExecutor(jsExecutor)
            .setRegistry(nativeModuleRegistry)
            .setJSBundleLoader(jsBundleLoader)
            .setNativeModuleCallExceptionHandler(exceptionHandler);

    ReactMarker.logMarker(CREATE_CATALYST_INSTANCE_START);
    // CREATE_CATALYST_INSTANCE_END is in JSCExecutor.cpp
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "createCatalystInstance");
    final CatalystInstance catalystInstance;
    try {
      // 没什么好说的。空值判断。但是催化器的构造函数我们需要研究一下。
      // 里面针对消息队列的线程进行初始化
      catalystInstance = catalystInstanceBuilder.build();
    } finally {
      Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
      ReactMarker.logMarker(CREATE_CATALYST_INSTANCE_END);
    }

    // 关联ReacContext与CatalystInstance实例化
    // 解析上面实例化的是三个线程封装对象
    reactContext.initializeWithInstance(catalystInstance);

    if (mJSIModulePackage != null) {
      // 这个默认是为null的。这个数据最初是在ReactNativeHost里面传入的
      catalystInstance.addJSIModules(
          mJSIModulePackage.getJSIModules(
              reactContext, catalystInstance.getJavaScriptContextHolder()));
      // 这个地方没太看懂
      if (ReactFeatureFlags.useTurboModules) {
        catalystInstance.setTurboModuleManager(
            catalystInstance.getJSIModule(JSIModuleType.TurboModuleManager));
      }
    }
    if (mBridgeIdleDebugListener != null) {
      catalystInstance.addBridgeIdleDebugListener(mBridgeIdleDebugListener);
    }
    if (Systrace.isTracing(TRACE_TAG_REACT_APPS | TRACE_TAG_REACT_JS_VM_CALLS)) {
      catalystInstance.setGlobalVariable("__RCTProfileIsProfiling", "true");
    }
    ReactMarker.logMarker(ReactMarkerConstants.PRE_RUN_JS_BUNDLE_START);
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "runJSBundle");
    // 加载JSBundle的数据
    catalystInstance.runJSBundle();
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);

    return reactContext;
  }
```

这个是非常重要的方法，我们需要一点点来分析一下。

1、首先他将Application的对象用ReactApplicationContext进行包装，生成ReactApplication对象。这个没什么好说的。简单的包装而已。

2、创建JavaModule的注册表

3、进行CatalystInstanceImpl实例化

4、关联ReacContext与CatalystInstance实例化 解析上面实例化的是三个线程封装对象。这个地方。

 我们主要作用是让reactContext来持有catalystInstance对象 两个作用：1、获取这个实力上面传入ReactQueueConfigurationSpec实例，或者三个消息队列线程封装的对象        2、主要是通过催化器实例获取JSModule、NativeModule

其次，ReacContext还有就是一些生命周期的管理


### 实例化CatalystInstanceImpl

下面就是最重要的催化剂实例的构造方法


```
private CatalystInstanceImpl(
      final ReactQueueConfigurationSpec reactQueueConfigurationSpec,
      final JavaScriptExecutor jsExecutor,
      final NativeModuleRegistry nativeModuleRegistry,
      final JSBundleLoader jsBundleLoader,
      NativeModuleCallExceptionHandler nativeModuleCallExceptionHandler) {
    Log.d(ReactConstants.TAG, "Initializing React Xplat Bridge.");
    Log.d(TAG, "FMsg:第七步：CatalystInstanceImpl() called with: reactQueueConfigurationSpec = [" + reactQueueConfigurationSpec + "], jsExecutor = [" + jsExecutor + "], nativeModuleRegistry = [" + nativeModuleRegistry + "], jsBundleLoader = [" + jsBundleLoader + "], nativeModuleCallExceptionHandler = [" + nativeModuleCallExceptionHandler + "]");
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "createCatalystInstanceImpl");

    // 重点关注一下这一方法。这是一个Native方法。这样我们就调用到Native层
    mHybridData = initHybrid();
    // ReactNative的消息队列相关的实例化
    // TODO 这个地方要重点看一下
    // 这个主要是分别在对应的线程中创建消息任务队列的Handler(搬运工)
    // 当然还创建了两个线程(主线程不用创建)
    mReactQueueConfiguration =
        ReactQueueConfigurationImpl.create(
            reactQueueConfigurationSpec, new NativeExceptionHandler());
    mBridgeIdleListeners = new CopyOnWriteArrayList<>();
    // NativeModules注入器
    mNativeModuleRegistry = nativeModuleRegistry;
    mJSModuleRegistry = new JavaScriptModuleRegistry();
    // JSBundlerLoader
    mJSBundleLoader = jsBundleLoader;
    mNativeModuleCallExceptionHandler = nativeModuleCallExceptionHandler;
    mNativeModulesQueueThread = mReactQueueConfiguration.getNativeModulesQueueThread();
    mTraceListener = new JSProfilerTraceListener(this);
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);

    Log.d(TAG, "FMsg: 第七步：CatalystInstanceImpl() called Initializing React Xplat Bridge before initializeBridge");
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "initializeCxxBridge");
    Log.d(TAG, "FMsg:第七步：CatalystInstanceImpl() called with: initializeCxxBridge");

    // TODO 这个地方要重点看一下. 又一个非常重要的方法。
    // 初始化Java和Native之间的Bridge桥。这个是一个Native方法
    initializeBridge(
        //
        new BridgeCallback(this),
        // JSCore的JS引擎
        jsExecutor,
        // JS消息队列线程封装对象
        mReactQueueConfiguration.getJSQueueThread(),
            // Native消息队列线程封装对象
        mNativeModulesQueueThread,
        // 非C++的Modules就是JavaModule.也就是我们在Application里面封装的Modules
        // 这些NativeModules都会被封装成JavaModuleWrapper对象送给Native层
        mNativeModuleRegistry.getJavaModules(this),
        // 获取我们所有的Module中的C++的modules。这个是通过注解来标记那个NativeModule是C++
        // 但是我们翻了一下源码，没看到哪个NativeModule是C++
        mNativeModuleRegistry.getCxxModules());
    Log.d(ReactConstants.TAG, "第七步：实例化Initializing React Xplat Bridge after initializeBridge");
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);

    mJavaScriptContextHolder = new JavaScriptContextHolder(getJavaScriptContext());
  }
```
下面，我们看一下Native层的调用

```
jni::local_ref<CatalystInstanceImpl::jhybriddata> CatalystInstanceImpl::initHybrid(
    jni::alias_ref<jclass>) {
   cout<<"===========FMsg:CatalystInstanceImpl initHybrid  called==============="<<endl;
  return makeCxxInstance();
}
```


```
void CatalystInstanceImpl::initializeBridge(
    jni::alias_ref<ReactCallback::javaobject> callback,
    // This executor is actually a factory holder.
    JavaScriptExecutorHolder* jseh,
    jni::alias_ref<JavaMessageQueueThread::javaobject> jsQueue,
    jni::alias_ref<JavaMessageQueueThread::javaobject> nativeModulesQueue,
    jni::alias_ref<jni::JCollection<JavaModuleWrapper::javaobject>::javaobject> javaModules,
    jni::alias_ref<jni::JCollection<ModuleHolder::javaobject>::javaobject> cxxModules) {
    LOG(INFO) <<"===========FMsg:CatalystInstanceImpl initializeBridge==============="<<endl;
    // .............省略部分代码

  moduleRegistry_ = std::make_shared<ModuleRegistry>(
    buildNativeModuleList(
       std::weak_ptr<Instance>(instance_),
       javaModules,
       cxxModules,
       moduleMessageQueue_));

  instance_->initializeBridge(
    std::make_unique<JInstanceCallback>(
    callback,
    moduleMessageQueue_),
    jseh->getExecutorFactory(),
    folly::make_unique<JMessageQueueThread>(jsQueue),
    moduleRegistry_);
}
```

至此，Java层的代码执行完毕。我们彻底就进入Native层的代码逻辑了。上面Jni代码中，调用的instance_->initializeBridge()。 这个逻辑层，我们要到Native层去查找逻辑实现了。


```
void Instance::initializeBridge(
    std::unique_ptr<InstanceCallback> callback,
    std::shared_ptr<JSExecutorFactory> jsef,
    std::shared_ptr<MessageQueueThread> jsQueue,
    std::shared_ptr<ModuleRegistry> moduleRegistry) {
  LOG(INFO) <<"===========FMsg:Instance initializeBridge==============="<<endl;
  callback_ = std::move(callback);
  moduleRegistry_ = std::move(moduleRegistry);
  jsQueue->runOnQueueSync([this, &jsef, jsQueue]() mutable {
    nativeToJsBridge_ = folly::make_unique<NativeToJsBridge>(
        jsef.get(), moduleRegistry_, jsQueue, callback_);

    std::lock_guard<std::mutex> lock(m_syncMutex);
    m_syncReady = true;
    m_syncCV.notify_all();
  });

  CHECK(nativeToJsBridge_);
}
```
上线的代码中。催化器对象在启动的流程中的主要作用就完成了。主要就是将催化器初始化的一些Java、Native、JS层的一些通信对象准备好。


我们在回过头去，看catalystInstance.runJSBundle()方法的调用。

```
  @Override
  public void runJSBundle() {
    Log.d(ReactConstants.TAG, "CatalystInstanceImpl.runJSBundle()");
    Log.d(TAG, "FMsg:第九步：runJSBundle() called");
    Assertions.assertCondition(!mJSBundleHasLoaded, "JS bundle was already loaded!");
    // incrementPendingJSCalls();
    mJSBundleLoader.loadScript(CatalystInstanceImpl.this);

    synchronized (mJSCallsPendingInitLock) {

      // Loading the bundle is queued on the JS thread, but may not have
      // run yet.  It's safe to set this here, though, since any work it
      // gates will be queued on the JS thread behind the load.
      mAcceptCalls = true;

      for (PendingJSCall function : mJSCallsPendingInit) {
        function.call(this);
      }
      mJSCallsPendingInit.clear();
      mJSBundleHasLoaded = true;
    }

    // This is registered after JS starts since it makes a JS call
    Systrace.registerListener(mTraceListener);
  }
```



```
  public static JSBundleLoader createFileLoader(
      final String fileName, final String assetUrl, final boolean loadSynchronously) {
    Log.d(TAG, "FMsg:第九步：createFileLoader() called with: fileName = [" + fileName + "], assetUrl = [" + assetUrl + "], loadSynchronously = [" + loadSynchronously + "]");
    return new JSBundleLoader() {
      @Override
      public String loadScript(JSBundleLoaderDelegate delegate) {
        delegate.loadScriptFromFile(fileName, assetUrl, loadSynchronously);
        return fileName;
      }
    };
  }
```



```
  @Override
  public void loadScriptFromFile(String fileName, String sourceURL, boolean loadSynchronously) {
    Log.d(TAG, "FMsg:第九步：loadScriptFromFile() called with: fileName = [" + fileName + "], sourceURL = [" + sourceURL + "], loadSynchronously = [" + loadSynchronously + "]");
    mSourceURL = sourceURL;
    jniLoadScriptFromFile(fileName, sourceURL, loadSynchronously);
  }
  
```
这个jniLoadScriptFromFile也是一个Native的方法，我们可以看到他会调用到CatalystInstanceImpl.cpp的

```
void CatalystInstanceImpl::jniLoadScriptFromFile(const std::string& fileName,
                                                 const std::string& sourceURL,
                                                 bool loadSynchronously) {
  LOG(INFO) <<"===========FMsg:CatalystInstanceImpl jniLoadScriptFromFile==============="<<endl;
  if (Instance::isIndexedRAMBundle(fileName.c_str())) {
    instance_->loadRAMBundleFromFile(fileName, sourceURL, loadSynchronously);
  } else {
    std::unique_ptr<const JSBigFileString> script;
    RecoverableError::runRethrowingAsRecoverable<std::system_error>(
      [&fileName, &script]() {
        script = JSBigFileString::fromPath(fileName);
      });
    instance_->loadScriptFromString(std::move(script), sourceURL, loadSynchronously);
  }
}
```
调用到Instancee.cpp的

```
void Instance::loadScriptFromString(std::unique_ptr<const JSBigString> string,
                                    std::string sourceURL,
                                    bool loadSynchronously) {
  LOG(INFO) <<"===========FMsg:Instance loadScriptFromString==============="<<endl;
  SystraceSection s("Instance::loadScriptFromString", "sourceURL",
                    sourceURL);
  if (loadSynchronously) {
    loadApplicationSync(nullptr, std::move(string), std::move(sourceURL));
  } else {
    loadApplication(nullptr, std::move(string), std::move(sourceURL));
  }
}


void Instance::loadApplication(std::unique_ptr<RAMBundleRegistry> bundleRegistry,
                               std::unique_ptr<const JSBigString> string,
                               std::string sourceURL) {
  LOG(INFO) <<"===========FMsg:Instance loadApplication=========string======"<<endl;
  callback_->incrementPendingJSCalls();
  SystraceSection s("Instance::loadApplication", "sourceURL",
                    sourceURL);
  nativeToJsBridge_->loadApplication(std::move(bundleRegistry), std::move(string),
                                     std::move(sourceURL));
}
```


下面有调用到了


```
void NativeToJsBridge::loadApplication(
    std::unique_ptr<RAMBundleRegistry> bundleRegistry,
    std::unique_ptr<const JSBigString> startupScript,
    std::string startupScriptSourceURL) {
   LOG(INFO) <<"===========FMsg:NativeToJsBridge loadApplication==============="<<endl;
  runOnExecutorQueue(
      [this,
       bundleRegistryWrap=folly::makeMoveWrapper(std::move(bundleRegistry)),
       startupScript=folly::makeMoveWrapper(std::move(startupScript)),
       startupScriptSourceURL=std::move(startupScriptSourceURL)]
        (JSExecutor* executor) mutable {
    auto bundleRegistry = bundleRegistryWrap.move();
    if (bundleRegistry) {
      executor->setBundleRegistry(std::move(bundleRegistry));
    }
     //executor从runOnExecutorQueue()返回的map中取得，与OnLoad中的JSCJavaScriptExecutorHolder对应，也与
     //Java中的JSCJavaScriptExecutor对应。它的实例在JSExecutor.cpp中实现。
    try {
      LOG(INFO) <<"===========FMsg:NativeToJsBridge executor  loadApplicationScript======" << startupScriptSourceURL << "。"<<endl;
      executor->loadApplicationScript(std::move(*startupScript),
                                      std::move(startupScriptSourceURL));
    } catch (...) {
      m_applicationScriptHasFailure = true;
      throw;
    }
  });
}
```



```

void NativeToJsBridge::runOnExecutorQueue(std::function<void(JSExecutor*)> task) {
  if (*m_destroyed) {
    return;
  }

  std::shared_ptr<bool> isDestroyed = m_destroyed;
  m_executorMessageQueueThread->runOnQueue([this, isDestroyed, task=std::move(task)] {
    if (*isDestroyed) {
      return;
    }

    // The executor is guaranteed to be valid for the duration of the task because:
    // 1. the executor is only destroyed after it is unregistered
    // 2. the executor is unregistered on this queue
    // 3. we just confirmed that the executor hasn't been unregistered above
    task(m_executor.get());
  });
}
```

下面，我们看到他调用了JSIExecutor的loadApplicationScript。下面我们来看一下这个方法的实现。


```
void JSIExecutor::loadApplicationScript(
    std::unique_ptr<const JSBigString> script,
    std::string sourceURL) {
  SystraceSection s("JSIExecutor::loadApplicationScript");
  LOG(INFO) <<"===========FMsg:JSIExecutor loadApplicationScript===============sourceURL = " << sourceURL << "。" <<endl;
  // TODO: check for and use precompiled HBC

  runtime_->global().setProperty(
      *runtime_,
      "nativeModuleProxy",
      Object::createFromHostObject(
          *runtime_, std::make_shared<NativeModuleProxy>(*this)));

  runtime_->global().setProperty(
      *runtime_,
      "nativeFlushQueueImmediate",
      Function::createFromHostFunction(
          *runtime_,
          PropNameID::forAscii(*runtime_, "nativeFlushQueueImmediate"),
          1,
          [this](
              jsi::Runtime &,
              const jsi::Value &,
              const jsi::Value *args,
              size_t count) {
            if (count != 1) {
              throw std::invalid_argument(
                  "nativeFlushQueueImmediate arg count must be 1");
            }
            // 应该是JSIExecutor⾥里里⾯面callNativeModules中的delegate_->callNativeModules
            LOG(INFO) <<"===========FMsg:JSIExecutor loadApplicationScript======callNativeModules======="<<endl;
            callNativeModules(args[0], false);
            return Value::undefined();
          }));

  runtime_->global().setProperty(
      *runtime_,
      "nativeCallSyncHook",
      Function::createFromHostFunction(
          *runtime_,
          PropNameID::forAscii(*runtime_, "nativeCallSyncHook"),
          1,
          [this](
              jsi::Runtime &,
              const jsi::Value &,
              const jsi::Value *args,
              size_t count) { return nativeCallSyncHook(args, count); }));

#if DEBUG
  runtime_->global().setProperty(
      *runtime_,
      "globalEvalWithSourceUrl",
      Function::createFromHostFunction(
          *runtime_,
          PropNameID::forAscii(*runtime_, "globalEvalWithSourceUrl"),
          1,
          [this](
              jsi::Runtime &,
              const jsi::Value &,
              const jsi::Value *args,
              size_t count) { return globalEvalWithSourceUrl(args, count); }));
#endif

  if (runtimeInstaller_) {
    runtimeInstaller_(*runtime_);
  }

  bool hasLogger(ReactMarker::logTaggedMarker);
  std::string scriptName = simpleBasename(sourceURL);
  if (hasLogger) {
    ReactMarker::logTaggedMarker(
        ReactMarker::RUN_JS_BUNDLE_START, scriptName.c_str());
  }
  //使用Webkit JSC去解释执行JS
  runtime_->evaluateJavaScript(
      std::make_unique<BigStringBuffer>(std::move(script)), sourceURL);
  flush();
  if (hasLogger) {
    ReactMarker::logMarker(ReactMarker::CREATE_REACT_CONTEXT_STOP);
    ReactMarker::logTaggedMarker(
        ReactMarker::RUN_JS_BUNDLE_STOP, scriptName.c_str());
  }
}
```


最后代码调用到evaluateJavaScript方法中。这个方法的实现JSCRuntime.cpp中。我们简简单看一下：


```
jsi::Value JSCRuntime::evaluateJavaScript(
    const std::shared_ptr<const jsi::Buffer> &buffer,
    const std::string& sourceURL) {
  // LOG(INFO) <<"===========FMsg:JSCRuntime evaluateJavaScript===============sourceURL = " << sourceURL << "。" <<endl;
  std::string tmp(
      reinterpret_cast<const char*>(buffer->data()), buffer->size());
  JSStringRef sourceRef = JSStringCreateWithUTF8CString(tmp.c_str());
  JSStringRef sourceURLRef = nullptr;
  if (!sourceURL.empty()) {
    sourceURLRef = JSStringCreateWithUTF8CString(sourceURL.c_str());
  }
  JSValueRef exc = nullptr;
  JSValueRef res =
      JSEvaluateScript(ctx_, sourceRef, nullptr, sourceURLRef, 0, &exc);
  JSStringRelease(sourceRef);
  if (sourceURLRef) {
    JSStringRelease(sourceURLRef);
  }
  checkException(res, exc);
  return createValue(res);
}
```


从上面的代码中，我们我们开始加载JSBundle的逻辑已经执行完毕。但是这个时候其实ReactNative并没有启动起来。

我们下面来看一下启动流程的最后一步：

我们来回到ReactRootView的startReactApplication这个方法中的最后一步：


```
  private void attachToReactInstanceManager() {
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "attachToReactInstanceManager");
    Log.d(TAG, "FMsg:attachToReactInstanceManager() called");
    try {
      if (mIsAttachedToInstance) {
        return;
      }

      mIsAttachedToInstance = true;
        // ReactInstanceManager绑定到当前的RootView
      Assertions.assertNotNull(mReactInstanceManager).attachRootView(this);
      getViewTreeObserver().addOnGlobalLayoutListener(getCustomGlobalLayoutListener());
    } finally {
      Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
    }
  }
```
方法很简单，我们来看attachRootView的实现

```
  public void attachRootView(ReactRoot reactRoot) {
    Log.d(TAG, "FMsg:attachRootView() called with: reactRoot = [" + reactRoot + "]");
    UiThreadUtil.assertOnUiThread();
    // ReactInstanceManager 是支持多 RootView 的.其实这个地方大家可以猜想，其实我们的ReactNative的页面，可以作为一个小的页面View
    // 附着到宿主Activity上。
    mAttachedReactRoots.add(reactRoot);

    // Reset reactRoot content as it's going to be populated by the application content from JS.
    clearReactRoot(reactRoot);

    // If react context is being created in the background, JS application will be started
    // automatically when creation completes, as reactRoot reactRoot is part of the attached
    // reactRoot reactRoot list.
    ReactContext currentContext = getCurrentReactContext();
    if (mCreateReactContextThread == null && currentContext != null) {
      // 最后，取出 ReactContext，让 ReactRootView 与 CatalystInstance 相关联：
      attachRootViewToInstance(reactRoot);
    }
  }
```



```
  private void attachRootViewToInstance(final ReactRoot reactRoot) {
    Log.d(ReactConstants.TAG, "ReactInstanceManager.attachRootViewToInstance()");
    Log.d(TAG, "FMsg:第十步：attachRootViewToInstance() called with: reactRoot = [" + reactRoot + "]");
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "attachRootViewToInstance");

    // 首先通过 UIManagerHelper 拿到 UIManager：
    // UIManager 是什么呢？它有两个实现类：FabricUIManager、UIManagerModule
    // 我们来看后者，它的注释中说：这是一个原生模块，允许JS创建和更新原生视图。其实这个就是我们后来看RN的渲染流程中非常重要的类
    UIManager uiManager =
        UIManagerHelper.getUIManager(mCurrentReactContext, reactRoot.getUIManagerType());

    @Nullable Bundle initialProperties = reactRoot.getAppProperties();

    final int rootTag =
        uiManager.addRootView(
            reactRoot.getRootViewGroup(),
            initialProperties == null
                ? new WritableNativeMap()
                : Arguments.fromBundle(initialProperties),
            reactRoot.getInitialUITemplate());
    reactRoot.setRootViewTag(rootTag);
    if (reactRoot.getUIManagerType() == FABRIC) {
      // Fabric requires to call updateRootLayoutSpecs before starting JS Application,
      // this ensures the root will hace the correct pointScaleFactor.
      uiManager.updateRootLayoutSpecs(
          rootTag, reactRoot.getWidthMeasureSpec(), reactRoot.getHeightMeasureSpec());
      reactRoot.setShouldLogContentAppeared(true);
    } else {
      Log.d(TAG, "FMsg:第十步：attachRootViewToInstance() called with: reactRoot = [" + reactRoot + "]");
      // 这个方法就是我们真的去调用JS的相关代码，我们可以看看这个代码
      reactRoot.runApplication();
    }
    Systrace.beginAsyncSection(
        TRACE_TAG_REACT_JAVA_BRIDGE, "pre_rootView.onAttachedToReactInstance", rootTag);
    // 这个地方其实，我们有疑问：
    // 为什么使用UiThreadUtil.runOnUiThread。而不是
    //  mCurrentReactContext.runOnUiQueueThread();???
    UiThreadUtil.runOnUiThread(
        new Runnable() {
          @Override
          public void run() {
            Systrace.endAsyncSection(
                TRACE_TAG_REACT_JAVA_BRIDGE, "pre_rootView.onAttachedToReactInstance", rootTag);
            reactRoot.onStage(ReactStage.ON_ATTACH_TO_INSTANCE);
          }
        });
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
  }
```

这个方法的最后一步，我们就可以看到reactRoot.runApplication()  这个方法很明显，就是运行起来JS的相关代码逻辑。


```
  @Override
  public void runApplication() {
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "ReactRootView.runApplication");
    Log.d(TAG, "FMsg:第十一步：ReactRootView runApplication() called");
    try {
      if (mReactInstanceManager == null || !mIsAttachedToInstance) {
        return;
      }

      ReactContext reactContext = mReactInstanceManager.getCurrentReactContext();
      if (reactContext == null) {
        return;
      }

      CatalystInstance catalystInstance = reactContext.getCatalystInstance();
      String jsAppModuleName = getJSModuleName();

      if (mUseSurface) {
        // TODO call surface's runApplication
      } else {
        if (mWasMeasured) {
          updateRootLayoutSpecs(mWidthMeasureSpec, mHeightMeasureSpec);
        }

        WritableNativeMap appParams = new WritableNativeMap();
        appParams.putDouble("rootTag", getRootViewTag());
        @Nullable Bundle appProperties = getAppProperties();
        if (appProperties != null) {
          appParams.putMap("initialProps", Arguments.fromBundle(appProperties));
        }

        mShouldLogContentAppeared = true;
        Log.i(TAG, "FMsg: 第十步：==============runApplication: ==========================" + jsAppModuleName);
        Log.i(TAG, "FMsg: 第十步：==============runApplication: ==========================" + appParams);
        catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams);
      }
    } finally {
      Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
    }
  }
```
这个方法也是比较简单的。通过ReactContext的实例，拿到催化器实例。然后通过催化器实例获取JSModule的runApplication


```
  @Override
  public <T extends JavaScriptModule> T getJSModule(Class<T> jsInterface) {
    // Log.d(TAG, "FMsg:getJSModule() called with: jsInterface = [" + jsInterface + "]");
    // mJSModuleRegistry注入器获取当前的注入器。至于怎么去获取的。我们可以再分析
    return mJSModuleRegistry.getJavaScriptModule(this, jsInterface);
  }
```

可以看到，最终调用的是 catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams) ， AppRegistry.class 是JS层暴露给Java层的接口方法。它的真正实现在 AppRegistry.js 里， AppRegistry.js 是运行所有 RN 应用的 JS 层入口，我们来看看它的实现：在 Libraries/ReactNative 中的 AppRegistry.js


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
    runnables[appKey].run(appParameters);
  },
```

到这里就会去调用JS进行渲染，在通过 UIManagerModule 将JS组件转换成Android组件，最终显示在 ReactRootView 上。

最后总结一下，就是先在应用终端启动并创建上下文对象，启动 JS Runtime ，进行布局，将JS端的代码通过C++层， UIManagerMoodule 转化成 Android 组件，再进行渲染，最后将渲染的View添加到 ReactRootView 上，最终呈现在用户面前。
