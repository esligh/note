### Glide源码分析-框架类分析

Glide是一个用来加载图片的三方库，它内部的实现逻辑非常复杂，类之间的关系错综复杂，如果一开始就扎进源码去分析它的实现将非常困难。所以在正式阅读源码之前，我们先从设计层面来分析下Glide的整体框架类，以及设计这些类背后的目的。这对于我们后续分析具体实现将会大有裨益，另一方面，我们还会从全局的角度来思考Glide的设计思想。

#### 类及接口分析

在Glide内部有众多的抽象类，接口，理解这些抽象类和接口能帮助我们更好的从设计层面了解Glide的实现。在正式阅读源码之前我们有必要理清一些关键类的职责。

#### Drawable

Drawable是系统抽象出来的一种可绘制对象，比如ShapeDrawable,ColorDrawable,BitmapDrawable等都是Drawable的具体子类。在Glide内部实现了它自己的Drawable类，比如GifDrawable,BitmapDrawable等，这些可显示对象最终会被显示到ImageView组件中。

```java
public abstract class Drawable {
    ...
    public abstract void draw(@NonNull Canvas canvas);
    
    public void setBounds(@NonNull Rect bounds) {
        setBounds(bounds.left, bounds.top, bounds.right, bounds.bottom);
    }
    
     public abstract void setAlpha(@IntRange(from=0,to=255) int alpha);
     public abstract void setColorFilter(@Nullable ColorFilter colorFilter);
    ...
}
```

#### Resouce

Resource接口描述的是一种可以复用的资源，Glide作为一种图片加载框架，必然需要对资源，内存的使用做适当的控制，因为系统分配给应用的内存空间是有限的。Resource实际上是一种包裹类。

```java
public interface Resource<Z> {
  /**
   * Returns the {@link Class} of the wrapped resource.
   */
  @NonNull
  Class<Z> getResourceClass();
  /**
   * Returns an instance of the wrapped resource.
   */
  @NonNull
  Z get();
  /**
   * Returns the size in bytes of the wrapped resource to use to determine how much of the memory
   * cache this resource uses.
   */
  int getSize();
  /**
   * Cleans up and recycles internal resources.
   */  
  void recycle();
}
```

主要子类如BitmapResource，BytesResource,DrawableResource等

#### ResourceTranscoder

ResourceTranscoder负责对资源进行转换，比如将Resource<Bitmap> 转为 Resource<BitmapDrawable>

```java
public interface ResourceTranscoder<Z, R> {

  /**
   * Transcodes the given resource to the new resource type and returns the new resource.
   *
   * @param toTranscode The resource to transcode.
   */
  @Nullable
  Resource<R> transcode(@NonNull Resource<Z> toTranscode, @NonNull Options options);
}
```

#### Key

Key用来唯一标识一些数据，因为Glide涉及到缓存，对象池等，必须要能够对资源进行唯一识别。

```java
public interface Key {
    
  void updateDiskCacheKey(@NonNull MessageDigest messageDigest);
    
  @Override
  boolean equals(Object o);
    
  @Override
  int hashCode();
}
```

它要求必须实现Object的equals和hashCode方法。子类有DataCacheKey，ResourceCacheKey，GlideUrl等，这里看看GlideUrl的实现

```java
public class GlideUrl implements Key {
    private final Headers headers;
  @Nullable private final URL url;
  @Nullable private final String stringUrl;

  @Nullable private String safeStringUrl;
  @Nullable private URL safeUrl;
  @Nullable private volatile byte[] cacheKeyBytes;
    
  public String getCacheKey() {
    return stringUrl != null ? stringUrl : Preconditions.checkNotNull(url).toString();
  }
  @Override
  public void updateDiskCacheKey(@NonNull MessageDigest messageDigest) {
    messageDigest.update(getCacheKeyBytes());
  }

  private byte[] getCacheKeyBytes() {
    if (cacheKeyBytes == null) {
      cacheKeyBytes = getCacheKey().getBytes(CHARSET);
    }
    return cacheKeyBytes;
  }
  @Override
  public boolean equals(Object o) {
    if (o instanceof GlideUrl) {
      GlideUrl other = (GlideUrl) o;
      return getCacheKey().equals(other.getCacheKey())
          && headers.equals(other.headers);
    }
    return false;
  }

  @Override
  public int hashCode() {
    if (hashCode == 0) {
      hashCode = getCacheKey().hashCode();
      hashCode = 31 * hashCode + headers.hashCode();
    }
    return hashCode;
  }  
}
```

从GlideUrl的实现来看，它是以Url来作为key值的。

#### Target

Target描述的是加载资源的目标对象，它内部有一组接口说明了资源加载的状态，典型的加载流程为onLoadStarted -> onResourceReady or onLoadFailed -> onLoadCleared。

```java
public interface Target<R> extends LifecycleListener {
  /**
   * A lifecycle callback that is called when a load is started.
   */
  void onLoadStarted(@Nullable Drawable placeholder);

  /**
   * A <b>mandatory</b> lifecycle callback that is called when a load fails.
   */
  void onLoadFailed(@Nullable Drawable errorDrawable);

  /**
   * The method that will be called when the resource load has finished.
   */
  void onResourceReady(@NonNull R resource, @Nullable Transition<? super R> transition);

  /**
   * A <b>mandatory</b> lifecycle callback that is called when a load is cancelled and its resources
   * are freed.
   */
  void onLoadCleared(@Nullable Drawable placeholder);
  ...  
}
```

为了方便使用，抽象类BaseTraget实现了Target接口的大多数方法。BaseTarget还有一个子类ViewTarget，它负责装在资源到一个View中。

```java

@Deprecated
public abstract class ViewTarget<T extends View, Z> extends BaseTarget<Z> {
}
```

它的子类如ImageViewTarget，负责将资源装载到一个ImageView中。它也是一个抽象类，向外部暴露了setResource方法，通过这个方法来讲具体的资源加载到ImageView中。它的实现如下：

```java
public abstract class ImageViewTarget<Z> extends ViewTarget<ImageView, Z>
    implements Transition.ViewAdapter {
    
    ...
    @Override
  	public void onLoadFailed(@Nullable Drawable errorDrawable) {
    	super.onLoadFailed(errorDrawable);
    	setResourceInternal(null);
    	setDrawable(errorDrawable);
  	}
    
    @Override
    public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
    	if (transition == null || !transition.transition(resource, this)) {
          setResourceInternal(resource);
        } else {
          maybeUpdateAnimatable(resource);
        }
  	}
    ...
    private void setResourceInternal(@Nullable Z resource) {
    // Order matters here. Set the resource first to make sure that the Drawable has a valid and
    // non-null Callback before starting it.
        setResource(resource);
        maybeUpdateAnimatable(resource);
  	}    
    ...
    protected abstract void setResource(@Nullable Z resource);
}
```

BitmapImageViewTarget,DrawableImageViewTarget等都是ImageViewTarget的实现类，他们实现了setResource接口，即通过imageView的setImageBitmap和setImageDrawable方法将资源设置到ImageView中。这里我们看看DrawableImageViewTarget的实现。

```java
public class DrawableImageViewTarget extends ImageViewTarget<Drawable> {
  public DrawableImageViewTarget(ImageView view) {
    super(view);
  }
  ...
  @Override
  protected void setResource(@Nullable Drawable resource) {
    view.setImageDrawable(resource);
  }
}
```

#### Request

Request描述了加载资源的请求对象，它负责加载资源到相应的Target中。

```java
public interface Request {
  /**
   * Starts an asynchronous load.
   */
  void begin();

  /**
   * Prevents any bitmaps being loaded from previous requests, releases any resources held by this
   * request, displays the current placeholder if one was provided, and marks the request as having
   * been cancelled.
   */
  void clear();

  /**
   * Returns true if this request is running and has not completed or failed.
   */
  boolean isRunning();

  /**
   * Returns true if the request has completed successfully.
   */
  boolean isComplete();
  ...
}
```

重要子类如SingleRequest

#### Encoder

Encoder是一个将特定类型的数据序列化输出到文件中的接口描述

```java
public interface Encoder<T> {
  /**
   * Writes the given data to the given output stream and returns True if the write  completed
   * successfully and should be committed.
   *
   * @param data The data to write.
   * @param file The File to write the data to.
   * @param options The put of options to apply when encoding.
   */
  boolean encode(@NonNull T data, @NonNull File file, @NonNull Options options);
}
```

它有个拓展的接口ResourceEncoder，负责将Resource描述的资源进行序列化输出存储。

```java
public interface ResourceEncoder<T> extends Encoder<Resource<T>> {
  // specializing the generic arguments
  @NonNull
  EncodeStrategy getEncodeStrategy(@NonNull Options options);
}
```

重要子类如BitmapDrawableEncoder，GifDrawableEncoder，BitamapEncoder等。我们这里看看BitamapEncoder的实现

```java
public class BitmapEncoder implements ResourceEncoder<Bitmap> {
  ...  
  @Override
  public boolean encode(@NonNull Resource<Bitmap> resource, @NonNull File file,
      @NonNull Options options) {
    final Bitmap bitmap = resource.get();
    Bitmap.CompressFormat format = getFormat(bitmap, options);
    try {
      int quality = options.get(COMPRESSION_QUALITY);
      boolean success = false;
      OutputStream os = null;
      try {
        os = new FileOutputStream(file);
        if (arrayPool != null) {
          os = new BufferedOutputStream(os, arrayPool);
        }
        bitmap.compress(format, quality, os);
        os.close();
        success = true;
      } catch (IOException e) {
      } finally {
        if (os != null) {
          try {
            os.close();
          } catch (IOException e) {
            // Do nothing.
          }
        }
      }
      return success;
    } finally {
      GlideTrace.endSection();
    }
  }
}
```

BitmapEncoder的实现很简单，就是对Bitmap资源进行质量压缩后，输出到文件流中，也就是将bitmap保存到指定文件中。

#### ResourceDecoder

ResourceDecoder是一个可以将资源源中decode出一个Resource的接口，资源的源可以是File，InputStream等。

```java
/**
 * An interface for decoding resources.
 *
 * @param <T> The type the resource will be decoded from (File, InputStream etc).
 * @param <Z> The type of the decoded resource (Bitmap, Drawable etc).
 */
public interface ResourceDecoder<T, Z> {
  ...  
  @Nullable
  Resource<Z> decode(@NonNull T source, int width, int height, @NonNull Options options)
      throws IOException;
}
```

ResourceDecorder的模板类型T代表了可以进行decode的资源源类型，它可以是File,InputStream等，类型Z是通过decode拿到的Resource资源类型。即可以是Bitmap，Drawable等，它也有众多的实现类，比如StreamBitmapDecoder从流中读取到Bitmap,StreamGifDecoder从流中读取GifDrawable，ResourceDrawableDecoder从Uri中读取到Drawable等。这里我们看看StreamBitmapDecoder的简单实现

```java
public class StreamBitmapDecoder implements ResourceDecoder<InputStream, Bitmap> {

  private final Downsampler downsampler;
  private final ArrayPool byteArrayPool;

  public StreamBitmapDecoder(Downsampler downsampler, ArrayPool byteArrayPool) {
    this.downsampler = downsampler;
    this.byteArrayPool = byteArrayPool;
  }
  ...
      
  @Override
  public Resource<Bitmap> decode(@NonNull InputStream source, int width, int height,
      @NonNull Options options)
      throws IOException {
    // Use to fix the mark limit to avoid allocating buffers that fit entire images.
    final RecyclableBufferedInputStream bufferedStream;
    final boolean ownsBufferedStream;
    if (source instanceof RecyclableBufferedInputStream) {
      bufferedStream = (RecyclableBufferedInputStream) source;
      ownsBufferedStream = false;
    } else {
      bufferedStream = new RecyclableBufferedInputStream(source, byteArrayPool);
      ownsBufferedStream = true;
    }
    ExceptionCatchingInputStream exceptionStream =
        ExceptionCatchingInputStream.obtain(bufferedStream);

    MarkEnforcingInputStream invalidatingStream = new MarkEnforcingInputStream(exceptionStream);
    ...
    return downsampler.decode(invalidatingStream, width, height, options, callbacks);
  }    
}
```

StreamBitmapDecoder的decode方法简单对source流做封装处理后，交给Downsampler，Downsampler负责专门从流中解析出Bitmap对象然后返回。

#### DataFetcher

DataFetcher是一种懒加载数据资源的策略，通过加载的数据资源可以进行具体资源的加载。数据资源的形式可以是InputStream，AssetFileDescriptor，ParcelFileDescriptor，而从这些数据资源中可以转换为Bitmap，GifDrawable等资源。

```java
public interface DataFetcher<T> {
    
  interface DataCallback<T> {

    /**
     * Called with the loaded data if the load succeeded, or with {@code null} if the load failed.
     */
    void onDataReady(@Nullable T data);

    /**
     * Called when the load fails.
     *
     * @param e a non-null {@link Exception} indicating why the load failed.
     */
    void onLoadFailed(@NonNull Exception e);
  }
  void cancel();
  void loadData(@NonNull Priority priority, @NonNull DataCallback<? super T> callback);
  DataSource getDataSource();
  ...  
}
```

资源是通过内部的loadData来加载，然后通过DataCallback回调来通知给使用者。它有众多的子类，如ThumbFetcher，HttpUrlFetcher，AssetPathFetcher等等。这里我们看下HttpUrlFetcher，它负责从网络下载资源

```java
public class HttpUrlFetcher implements DataFetcher<InputStream> {
    ...
    private final GlideUrl glideUrl;
    private HttpURLConnection urlConnection;
  	private InputStream stream;
  	private volatile boolean isCancelled;
    ...
    @Override
  public void loadData(@NonNull Priority priority,
      @NonNull DataCallback<? super InputStream> callback) {
    try {
      InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
      callback.onDataReady(result);
    } catch (IOException e) {
      callback.onLoadFailed(e);
    } finally {
    }
  }    
  private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl,
      Map<String, String> headers) throws IOException {
    ...
    urlConnection = connectionFactory.build(url);
    for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
      urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
    }
    urlConnection.setConnectTimeout(timeout);
    urlConnection.setReadTimeout(timeout);
    urlConnection.setUseCaches(false);
    urlConnection.setDoInput(true);

    // Stop the urlConnection instance of HttpUrlConnection from following redirects so that
    // redirects will be handled by recursive calls to this method, loadDataWithRedirects.
    urlConnection.setInstanceFollowRedirects(false);

    // Connect explicitly to avoid errors in decoders if connection fails.
    urlConnection.connect();
    // Set the stream so that it's closed in cleanup to avoid resource leaks. See #2352.
    stream = urlConnection.getInputStream();
    if (isCancelled) {
      return null;
    }
    final int statusCode = urlConnection.getResponseCode();
    if (isHttpOk(statusCode)) {
      return getStreamForSuccessfulRequest(urlConnection);
    } 
    ...  
  }  
}
```

HttpUrlFetcher负责从网络加载资源，它返回一个InputStream. 可以看到，Glide内部使用的网络请求库是HttpUrlConnection.

#### ModelLoader

ModelLoader可以将各种复杂的数据模型转换成一种聚合类型，这个聚合类型数据可以通过DataFetcher来获取。这个接口有两个目标：

**1) 转换具体的Model类型为一个可以加载出资源对象的类型Data**

**2) 允许这个Model类型结合view的尺寸去加载指定大小的资源**

简单来说，ModelLoader将我们配置的请求加载类型，比如String,File等转换成可以通过DataFetcher来获取的类型。所以它和DataFetcher紧密相关。

```java
public interface ModelLoader<Model, Data> {
    
    class LoadData<Data> {
        public final Key sourceKey;
        public final List<Key> alternateKeys;
        public final DataFetcher<Data> fetcher;

        public LoadData(@NonNull Key sourceKey, @NonNull DataFetcher<Data> fetcher) {
          this(sourceKey, Collections.<Key>emptyList(), fetcher);
        }

        public LoadData(@NonNull Key sourceKey, @NonNull List<Key> alternateKeys,
            @NonNull DataFetcher<Data> fetcher) {
          this.sourceKey = Preconditions.checkNotNull(sourceKey);
          this.alternateKeys = Preconditions.checkNotNull(alternateKeys);
          this.fetcher = Preconditions.checkNotNull(fetcher);
        }
    }
    
  	@Nullable
  	LoadData<Data> buildLoadData(@NonNull Model model, int width, int height,
      @NonNull Options options);
    
    boolean handles(@NonNull Model model);
}
```

类型Model代表的是需要转换的复杂数据类型，Data是可以通过DataFetcher加载资源的类型。同时，ModelLoader内部有一类LoadData，它根据Data类型封装了DataFetcher，而LoadData它是通过buildLoadData返回的。

DataLoader也有很多子类，比如HttpGlideUrlLoader,UriLoader,FileLoader,ByteBufferLoader等等。这里我们看HttpGlideUrlLoader的实现

```java
public class HttpGlideUrlLoader implements ModelLoader<GlideUrl, InputStream> {
  ...
  @Override
  public LoadData<InputStream> buildLoadData(@NonNull GlideUrl model, int width, int height,
      @NonNull Options options) {
    // GlideUrls memoize parsed URLs so caching them saves a few object instantiations and time
    // spent parsing urls.
    GlideUrl url = model;
    ...
    return new LoadData<>(url, new HttpUrlFetcher(url, timeout));
  }

  @Override
  public boolean handles(@NonNull GlideUrl model) {
    return true;
  }
    
  public static class Factory implements ModelLoaderFactory<GlideUrl, InputStream> {
    private final ModelCache<GlideUrl, GlideUrl> modelCache = new ModelCache<>(500);

    @NonNull
    @Override
    public ModelLoader<GlideUrl, InputStream> build(MultiModelLoaderFactory multiFactory) {
      return new HttpGlideUrlLoader(modelCache);
    }
    ...
  }
}
```

HttpGlideUrlLoader的model类型为GlideUrl,data类型为InputStream,前者是Glide对于普通的url进行了封装，后者可以通过DataLoader内部的DataFetcher获取到，然后交给ResourceDecoder解析出具体的资源类型。可以看到buildLoadData内部它通过HttpUrlFetcher来构造LoadData，这个HttpUrlFetcher就是一个DataFetcher，它被封装在LoadData中以备使用。

#### Transformation

Transformation即转换，可以对于某一种资源进行转换处理，转换前后的类型一致。这个我们比较熟悉，比如我们需要将一个普通的矩形图片转换成带圆角的或者圆形的图片就需要为Glide配置对应的Transformation，它负责来处理这种转换。

```java
public interface Transformation<T> extends Key {
  /**
   * Transforms the given resource and returns the transformed resource.
   */  
  @NonNull
  Resource<T> transform(@NonNull Context context, @NonNull Resource<T> resource,
      int outWidth, int outHeight);
}
```

它的子类有DrawableTransformation，BitmapTransformation，MultiTransformation，其中MultiTransformation比较特殊，它会经过多个Transformation来进行转换，我们看看它的实现

```java
public class MultiTransformation<T> implements Transformation<T> {
  	private final Collection<? extends Transformation<T>> transformations;
    public MultiTransformation(@NonNull Collection<? extends Transformation<T>> transformationList) {
    	this.transformations = transformationList;
  	}
    ...
    public Resource<T> transform(
      @NonNull Context context, @NonNull Resource<T> resource, int outWidth, int outHeight) {
        Resource<T> previous = resource;

        for (Transformation<T> transformation : transformations) {
          Resource<T> transformed = transformation.transform(context, previous, outWidth, outHeight);
          if (previous != null && !previous.equals(resource) && !previous.equals(transformed)) {
            previous.recycle();
          }
          previous = transformed;
        }
        return previous;
    }
}
```

能够进行多次转换是因为它内部有一个Transformation集合，在transform时通过每个Transformation进行处理就达到了多次转换的目的。



经过以上的分析，我们大概对于Glide有了一些整体的了解。比如资源的形式会是以Resouce来体现，资源会通过Encoder或者Decoder来进行序列化处理。资源源数据会通过DataFetcher来获取，ModelLoader负责将模型类型转换成一个DataFetcher，Transformation会对资源做一系列的转换。最终资源会和一个Target关联，用来显示或者获取到最终处理后的资源。清楚了以上的内容，我们再来分析根据这个主线去分析Glide的源码相对来说会容易很多。