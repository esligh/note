### AssetManager


```java
ActivityThread.java
-------------------  
public ActivityClientRecord(IBinder token, Intent intent, int ident,
    ActivityInfo info, Configuration overrideConfig, CompatibilityInfo compatInfo,
    String referrer, IVoiceInteractor voiceInteractor, Bundle state,
    PersistableBundle persistentState, List<ResultInfo> pendingResults,
    List<ReferrerIntent> pendingNewIntents, boolean isForward,
    ProfilerInfo profilerInfo, ClientTransactionHandler client) {
    this.token = token;
    this.ident = ident;
    this.intent = intent;
    this.referrer = referrer;
    this.voiceInteractor = voiceInteractor;
    this.activityInfo = info;
    this.compatInfo = compatInfo;
    this.state = state;
    this.persistentState = persistentState;
    this.pendingResults = pendingResults;
    this.pendingIntents = pendingNewIntents;
    this.isForward = isForward;
    this.profilerInfo = profilerInfo;
    this.overrideConfig = overrideConfig;
    this.packageInfo = client.getPackageInfoNoCheck(activityInfo.applicationInfo,
        compatInfo);
    init();
}
```

Activity在第一个启动的时候，在ActivityThread中会去为其创建一个ActivityClientRecord,并将其添加到一个由AcvityThead管理的集合中。在创建时候的时候会去创建一个LoadedApk对象，它代表了一个已经加载的apk，即其内部的pakcageInfo对象
它通过ActivityThread来加载。

```java
ActivityThread.java
-------------------

@Override
public final LoadedApk getPackageInfoNoCheck(ApplicationInfo ai,
        CompatibilityInfo compatInfo) {
    return getPackageInfo(ai, compatInfo, null, false, true, false);
}

private LoadedApk getPackageInfo(ApplicationInfo aInfo, CompatibilityInfo compatInfo,
            ClassLoader baseLoader, boolean securityViolation, boolean includeCode,
            boolean registerPackage) {
    final boolean differentUser = (UserHandle.myUserId() != UserHandle.getUserId(aInfo.uid));
    synchronized (mResourcesManager) {
        WeakReference<LoadedApk> ref;
        if (differentUser) {
            // Caching not supported across users
            ref = null;
        } else if (includeCode) {
            ref = mPackages.get(aInfo.packageName);
        } else {
            ref = mResourcePackages.get(aInfo.packageName);
        }

        LoadedApk packageInfo = ref != null ? ref.get() : null;
        if (packageInfo == null || (packageInfo.mResources != null
                && !packageInfo.mResources.getAssets().isUpToDate())) {
            if (localLOGV) Slog.v(TAG, (includeCode ? "Loading code package "
                    : "Loading resource-only package ") + aInfo.packageName
                    + " (in " + (mBoundApplication != null
                            ? mBoundApplication.processName : null)
                    + ")");
            packageInfo =
                new LoadedApk(this, aInfo, compatInfo, baseLoader,
                        securityViolation, includeCode &&
                        (aInfo.flags&ApplicationInfo.FLAG_HAS_CODE) != 0, registerPackage);

            if (mSystemThread && "android".equals(aInfo.packageName)) {
                packageInfo.installSystemApplicationInfo(aInfo,
                        getSystemContext().mPackageInfo.getClassLoader());
            }

            if (differentUser) {
                // Caching not supported across users
            } else if (includeCode) {
                mPackages.put(aInfo.packageName,
                        new WeakReference<LoadedApk>(packageInfo));
            } else {
                mResourcePackages.put(aInfo.packageName,
                        new WeakReference<LoadedApk>(packageInfo));
            }
        }
        return packageInfo;
    }
}
```

在ActivityThread中缓存了LoadedApk的集合，不过都是以弱引用的形式存储在集合中，如果不在集合中会去new一个新的LoadedApk。

```java
LoadedApk.java
--------------
public Resources getResources() {
    if (mResources == null) {
        final String[] splitPaths;
        try {
            splitPaths = getSplitPaths(null);
        } catch (NameNotFoundException e) {
            // This should never fail.
            throw new AssertionError("null split not found");
        }

        mResources = ResourcesManager.getInstance().getResources(null, mResDir,
                splitPaths, mOverlayDirs, mApplicationInfo.sharedLibraryFiles,
                Display.DEFAULT_DISPLAY, null, getCompatibilityInfo(),
                getClassLoader());
    }
    return mResources;
}
```
通过LoadedApk可以拿到其内部的Resources对象，它就是我们在Activity中通过context拿到的Resources对象，实际上它首次是通过ResroucesManager对象通过getResources方法获取到的。

### Context如何和LoadedApk关联

在首次启动Activity时候会触发handleLaunchActivity，在其内部通过performLaunchActivity来为Activity创建Context,那么它应该是在这时候将LoadedApk的resource和Context关联

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    ContextImpl appContext = createBaseContextForActivity(r);
    ...
}

private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
    ...
    ContextImpl appContext = ContextImpl.createActivityContext(
            this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);
    ...
    return appContext;
}
```
我们知道在perforumLaunchActivity执行之前，ActivityClientRecord对象早已创建，所以LoadedApk对象也已经构造放在了mPackages集合中。

```java
 static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
            Configuration overrideConfiguration) {
    ...

    ContextImpl context = new ContextImpl(null, mainThread, packageInfo, activityInfo.splitName,
            activityToken, null, 0, classLoader);

    ...

    final ResourcesManager resourcesManager = ResourcesManager.getInstance();

    // Create the base resources for which all configuration contexts for this Activity
    // will be rebased upon.
    context.setResources(resourcesManager.createBaseActivityResources(activityToken,
            packageInfo.getResDir(),
            splitDirs,
            packageInfo.getOverlayDirs(),
            packageInfo.getApplicationInfo().sharedLibraryFiles,
            displayId,
            overrideConfiguration,
            compatInfo,
            classLoader));
    ...        
    return context;
}
```


