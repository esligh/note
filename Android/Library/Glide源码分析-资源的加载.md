### Glide源码分析-图片的加载

在开源码之前，我们先想想Glide能够做什么？它有什么特色和有点？下面我简单列下：

1. 图片的异步加载，支持设置加载尺寸，支持设置加载动画，支持加载中和加载失败的图片。
2. 支持加载图片的格式丰富：JPG,PNG,GIF,WEBP等，同时支持缩略图
3. 支持加载本地资源和网络资源
4. 支持内存缓存，硬盘缓存等缓存策略
5. 生命周期集成，根据Activity／fragment生命周期自动管理请求

   6.支持图片变换

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

Glide为了方便用户在不同场景下使用，所以可以调用多种不同的参数的with方法。



