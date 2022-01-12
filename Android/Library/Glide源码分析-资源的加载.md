### Glide源码分析-图片的加载

在开源码之前，我们先想想Glide能够做什么？它有什么特色和有点？下面我简单列下：

1. 图片的异步加载，支持设置加载尺寸，支持设置加载动画，支持加载中和加载失败的图片。
2. 支持加载图片的格式丰富：JPG,PNG,GIF,WEBP等，同时支持缩略图
3. 支持加载本地资源和网络资源
4. 支持内存缓存，硬盘缓存等缓存策略
5. 生命周期集成，根据Activity／fragment生命周期自动管理请求
6. 支持图片变换

在上一篇中，我们简单的了解了Glide框架的一些设计类，后续我们将会对具体的实现细节进行分析，同时，在分析源码过程中，我们应该思考，从源码的角度设计者是如何实现以上提到的Glide特性。接下来我们看看Glide图片加载具体的实现过程

先来看Glide的with方法，它可以可Context，View，Activity，Fragment等类型作为参数

```java
Glide.java
-----------    
@NonNull
public static RequestManager with(@NonNull Context context) {
    return getRetriever(context).get(context);
}

@NonNull
public static RequestManager with(@NonNull Activity activity) {
    return getRetriever(activity).get(activity);
}

@NonNull
public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
}

@NonNull
public static RequestManager with(@NonNull Fragment fragment) {
    return getRetriever(fragment.getActivity()).get(fragment);
}

/** @deprecated */
@Deprecated
@NonNull
public static RequestManager with(@NonNull android.app.Fragment fragment) {
    return getRetriever(fragment.getActivity()).get(fragment);
}

@NonNull
public static RequestManager with(@NonNull View view) {
    return getRetriever(view.getContext()).get(view);
}
```

Glide为了方便用户在不同场景下使用，所以可以调用多种不同参数的with方法。它内部的方法实现很简单，就是调用getRetriever方法。我们看看它的实现

```java
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
        return get(context).getRequestManagerRetriever();
}
```

get方法会去创建Glide对象，Glide是个单例，它内部只会初始化一次，初始化完成后返回一个RequestManagerRetriever。我们先看Glide的初始化

```java
Glide build(@NonNull Context context) {
    if (engine == null) {
      engine =
          new Engine(
              memoryCache,
              diskCacheFactory,
              diskCacheExecutor,
              sourceExecutor,
              GlideExecutor.newUnlimitedSourceExecutor(),
              GlideExecutor.newAnimationExecutor(),
              isActiveResourceRetentionAllowed);
    }
    ...
    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory);

    return new Glide(
        context,
        engine,
        memoryCache,
        bitmapPool,
        arrayPool,
        requestManagerRetriever,
        connectivityMonitorFactory,
        logLevel,
        defaultRequestOptions.lock(),
        defaultTransitionOptions,
        defaultRequestListeners,
        isLoggingRequestOriginsEnabled);
  }
```

Glide通过build模式创建，在创建之前也同时创建了一个RequestManagerRetriever,在with方法中通过getRetriever方法获取到的就是这个RequestManagerRetriever。同时在创建Glide对象的时候，还初始化了很多模块。

```java
Glide(@NonNull Context context, @NonNull Engine engine, @NonNull MemoryCache memoryCache, @NonNull BitmapPool bitmapPool, @NonNull ArrayPool arrayPool, @NonNull RequestManagerRetriever requestManagerRetriever, @NonNull ConnectivityMonitorFactory connectivityMonitorFactory, int logLevel, @NonNull RequestOptions defaultRequestOptions, @NonNull Map<Class<?>, TransitionOptions<?, ?>> defaultTransitionOptions, @NonNull List<RequestListener<Object>> defaultRequestListeners, boolean isLoggingRequestOriginsEnabled) {
        this.memoryCategory = MemoryCategory.NORMAL;
        this.engine = engine;
        this.bitmapPool = bitmapPool;
        this.arrayPool = arrayPool;
        this.memoryCache = memoryCache;
        this.requestManagerRetriever = requestManagerRetriever;
        this.connectivityMonitorFactory = connectivityMonitorFactory;
        DecodeFormat decodeFormat = (DecodeFormat)defaultRequestOptions.getOptions().get(Downsampler.DECODE_FORMAT);
        this.bitmapPreFiller = new BitmapPreFiller(memoryCache, bitmapPool, decodeFormat);
        Resources resources = context.getResources();
        this.registry = new Registry();
        this.registry.register(new DefaultImageHeaderParser());
        if (VERSION.SDK_INT >= 27) {
            this.registry.register(new ExifInterfaceImageHeaderParser());
        }
        List<ImageHeaderParser> imageHeaderParsers = this.registry.getImageHeaderParsers();
        Downsampler downsampler = new Downsampler(imageHeaderParsers, resources.getDisplayMetrics(), bitmapPool, arrayPool);
        ...
        StreamBitmapDecoder streamBitmapDecoder = new StreamBitmapDecoder(downsampler, arrayPool);
        ResourceDrawableDecoder resourceDrawableDecoder = new ResourceDrawableDecoder(context);
        StreamFactory resourceLoaderStreamFactory = new StreamFactory(resources);
        UriFactory resourceLoaderUriFactory = new UriFactory(resources);
        FileDescriptorFactory resourceLoaderFileDescriptorFactory = new FileDescriptorFactory(resources);
        AssetFileDescriptorFactory resourceLoaderAssetFileDescriptorFactory = new AssetFileDescriptorFactory(resources);
        BitmapEncoder bitmapEncoder = new BitmapEncoder(arrayPool);
        BitmapBytesTranscoder bitmapBytesTranscoder = new BitmapBytesTranscoder();
        GifDrawableBytesTranscoder gifDrawableBytesTranscoder = new GifDrawableBytesTranscoder();
        ContentResolver contentResolver = context.getContentResolver();
        this.registry.append(ByteBuffer.class, new ByteBufferEncoder()).append(InputStream.class, new StreamEncoder(arrayPool)).append("Bitmap", ByteBuffer.class, Bitmap.class, byteBufferBitmapDecoder).append("Bitmap", InputStream.class, Bitmap.class, streamBitmapDecoder)..append("BitmapDrawable", InputStream.class, BitmapDrawable.class, new BitmapDrawableDecoder(resources, streamBitmapDecoder))....
        ImageViewTargetFactory imageViewTargetFactory = new ImageViewTargetFactory();
        this.glideContext = new GlideContext(context, arrayPool, this.registry, imageViewTargetFactory, defaultRequestOptions, defaultTransitionOptions, defaultRequestListeners, engine, isLoggingRequestOriginsEnabled, logLevel);
    }
```

在创建Glide的时候就已经创建了各种Decoder和Encoder，还有很多Factory都一起保存到GlideContext中。所以在完成了with操作后，Glide内部其实已经做了大量的准备工作。那么这个RequestManagerRetriever是个什么呢？简单来讲它负责创建RequestManager，RequestManager是用来负责管理和发起Glide请求的类，它实现了LifecycleListener接口，从这点来讲，它具有感知Activity或者Fragment生命周期的能力，那么由它负责管理Glide的请求，可以实现自动托管，由它负责在页面或者组件销毁后自动取消请求以防止内存泄漏的发生。那么它是如何做到感应组件的生命周期呢？这就需要回头看RequestManagerRetriever是如何创建它的。

```java
RequestManagerRetriever.java
----------------------------    
@NonNull
public RequestManager get(@NonNull Activity activity) {
    if (Util.isOnBackgroundThread()) {
        return get(activity.getApplicationContext());
    } else {
        assertNotDestroyed(activity);
        android.app.FragmentManager fm = activity.getFragmentManager();
        return fragmentGet(
            activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
}

private RequestManager fragmentGet(@NonNull Context context,
      @NonNull android.app.FragmentManager fm,
      @Nullable android.app.Fragment parentHint,
      boolean isParentVisible) {
    RequestManagerFragment current = getRequestManagerFragment(fm, parentHint, isParentVisible);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      Glide glide = Glide.get(context);
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      current.setRequestManager(requestManager);
    }
    return requestManager;
}

private RequestManagerFragment getRequestManagerFragment(
    @NonNull final android.app.FragmentManager fm,
    @Nullable android.app.Fragment parentHint,
    boolean isParentVisible) {
    RequestManagerFragment current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
        current = pendingRequestManagerFragments.get(fm);
        if (current == null) {
            current = new RequestManagerFragment();
            current.setParentFragmentHint(parentHint);
            if (isParentVisible) {
                current.getGlideLifecycle().onStart();
            }
            pendingRequestManagerFragments.put(fm, current);
            fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
            handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
        }
    }
    return current;
}
```

RequestManagerRetriever通过一组get方法来创建RequestManager，这里以Activity为例。从上面的代码来看，它为Activity页面绑定了一个RequestManagerFragment，所以这个RequestManagerFragment具有感应Activity生命周期的能力，同时会创建一个RequestManager和这个RequestManagerFragment进行绑定，所以RequestManager也就能够感应到生命周期变化。其他的get方法类似，但是需要特别注意的是如果是在子线程或者通过ApplicationContext来创建Glide，那么它会有一个单独的RequestManager，它和应用的生命周期管理，所以也不需要绑定到Framgent上。



好了，有了RequestManager我们就可以通过Glide来开始发起请求了。请求由一组load方法发起，这里我们看通过url加载图片的即可，其他情况可自行分析。

```java
RequestManager.java
--------------    
public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
}

public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
}

public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
}
```

load过程实际上只是为了创建RequestBuilder对象，可以通过这个RequestBuilder对象来创建真正的请求。

```java
RequestBuilder.java
-------------------    
public class RequestBuilder<TranscodeType> extends BaseRequestOptions<RequestBuilder<TranscodeType>>
    implements Cloneable,
    ModelTypes<RequestBuilder<TranscodeType>> {
    ...    
    public RequestBuilder<TranscodeType> apply(@NonNull BaseRequestOptions<?> requestOptions) {
        Preconditions.checkNotNull(requestOptions);
        return super.apply(requestOptions);
    }
    ...
}
```

在请求之前可以配置options，包括缓存策略，placeholder等，也就是我们经常使用的RequestOptions(这个区别于Glide3)，然后通过apply方法配置到RequestBuilder中。实际上RequestBuilder本身就是一个BaseRequestOptions的子类，而RequestOptions又是一个BaseRequestOptions。

```java
RequestManager.java
-------------------
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {

    BaseRequestOptions<?> requestOptions = this;
    ...

    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
  }
```

那么可以猜想通过into方法最终会发起请求，然后将图片资源加载到ImageView中。在这个inito中它通过GlideContext创建了一个ImageViewTarget，我们知道获取到图片资源后会交给一个Target进行处理，这里要显示到ImageView中，那么它会创建一个ImageViewTraget来处理此事。这个我们最后再看，先继续内部的into方法如何处理

```java
private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    
    Request request = buildRequest(target, targetListener, options, callbackExecutor);
	...
    requestManager.clear(target);
    target.setRequest(request);
    requestManager.track(target, request);

    return target;
}
```

这里通过buildRequest方法创建一个Request对象，然后通过RequestManager的track方法发起Glide请求。到这里我们大致理清了Glide的处理逻辑，即通过将操作封装成一个Request，通过RequestManager发起处理这个请求，然后将结果递交给Target处理。当然细节我们还需要继续深挖。

```java
RequestTracker.java
-------------------    
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
    targetTracker.track(target);
    requestTracker.runRequest(request);
}
```

RequestManager通过内部的RequestTracker的runRequest方法发起请求，RequestTracker负责请求的发起，重试，取消等操作。这里我们看看它如何发起请求

```java
public void runRequest(@NonNull Request request) {
    requests.add(request);
    if (!isPaused) {
        request.begin();
    } else {
        request.clear();
        pendingRequests.add(request);
    }
}
```

在into方法中，通过buildRequest创建Request方法