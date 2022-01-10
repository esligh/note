### View的创建过程

在Activity显示View之前，我们需要通过setContentView创建好View树，即DecorView，这样在resume阶段，可以将其显示出来。接下来我们看看在setContentView中是如何创建View树的

```java

PhoneWindow.java
-----------------
public void setContentView(int layoutResID) {
   if (mContentParent == null) {
      installDecor();
   } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
      mContentParent.removeAllViews();
   }

   mLayoutInflater.inflate(layoutResID, mContentParent);

   ....
}
```

setContentView它大致完成了以下事情：
1. 创建DecorView ，即View树的根节点，在这里面同时会初始化mContentParent，它是我们布局文件的根容器。我们通过xml定义的布局视图会添加到这个节点上。DecorView中包含一个Vertical的LinearLayout，Linearlayout由两部分组成，第一部分为标题栏，第二部分为内容栏，内容栏是一个FrameLayout，我们在Activity中调用setContentView就是将View添加到这个FrameLayout中。
2. 如果mContentParent不为空，那么在未设置FEATURE_CONTENT_TRANSITIONS的情况下，再次调用setContentView的结果就需要先将之前添加的视图对象移除，也就说setContentView是可以多次调用进行切换的，切换的时候只是改变mContentParent节点的视图，DecorView并不会再次创建。这里的mContentParent就是内容栏，它实际上是个FrameLayout。

3. 创建View树的view对象，这是通过LayoutInflater完成的。

接下来我们一步一步看这些是如何完成的。

#### 创建DecorView

上面提到，DecorView是View树的根节点，从应用的角度来看，它非常重要，因为它涉及到用户所能看到的具体的内容，所以很多重要的事物都和它紧密相关，比如事件分发，绘制等。但是从系统服务(WMS,AMS等)的角度来看，系统并不知道它的存在，也不关心。因为系统根据应用的需求提供了一块画布，至于应用该如何去使用这块画布，那是应用的事情，应用只要在画布上绘制完成后告诉去执行刷新显示就行了。但是从开发的角度来讲，如果由开发者一步一步去绘制将会非常繁琐，因为会有很多重复性的工作，所以在应用层，系统将这部分繁琐的工作简化了， 托管了一部分的工作让开发者只关心自身业务相关的View的创建，然后将创建的View添加到DecorView上就算是一个完整的View树。下面我们看看DecorView的创建

```java
DecorView.java
--------------    
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks
```

DecorView它本质上是个FrameLayout，即一个ViewGroup。它通过installDecor来创建

```java
PhoneWindow.java
----------------    
private void installDecor() {
   mForceDecorInstall = false;
   if (mDecor == null) {
      mDecor = generateDecor(-1);
      ...
   } else {
      mDecor.setWindow(this);
   }
   if (mContentParent == null) {
      mContentParent = generateLayout(mDecor);
      ... 
   }
}

protected DecorView generateDecor(int featureId) {
    ...
    return new DecorView(context, featureId, this, getAttributes());
}
```
对于一个Activity来说，无论内部的视图如何变化，DecorView只会创建一次，它是通过generateDecor进行创建，这时候如果mContentParent未创建，则通过generateLayout进行初始化，实际上它内部是通过findViewById查找对应id为**com.android.internal.R.id.content**的ViewGroup。

#### Inflate 过程



