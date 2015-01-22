---
layout: post
title: "Android中触摸事件分发"
description: ""
category: "知识整理"
tags: [Android,Event]
---

这篇主要记录一些事件分发，手势操作和UI滚动的知识，其实这是一连串的过程，手势操作导致了事件的分发和UI变化。

## 事件分发

事件分发是通过Actvity的 `dispatchTouchEvent(MotionEvent)` 方法来分发的，Activity会在接收到触摸事件的时候调用这个方法把事件发送到
窗口。如果我们在Activity中覆盖这个方法，会导致整个View接收到不到任何触摸事件。

每个View都会有各种`OnXXXXListener`事件，开发者可以注册这个事件获取回调事件，比较常见的有

1.  View.OnClickListener 点击回调
2.  View.OnLongClickListener 长按回调
3.  View.OnTouchListener 触摸回调，手指在手机上落下、移动、离开都是触摸回调

这些回调事件是View框架留给开发者的事件处理接口，这些接口会影响事件分发的流程，所以先不讨论。

对于比较原子的View类型，比如Button、TextView等等他们的事件分发是比较简单的。

```
public boolean dispatchTouchEvent(MotionEvent event) {
        ...
       if (onTouchEvent(event)) {
           return true;
       }
        ...
}
```

事件会传递到 `onTouchEvent` 方法，它的默认实现处理了 `View.OnClickListener`和`View.OnLongClickListener`这些事件。如果我们需要添加自己的事件处理，可以考虑在这里面操作。

但是对于ViewGroup这种包含子View的控件还是比较复杂的。他要多处理一个事件 `onInterceptTouchEvent` 同时负责给子View分发事件

```
public boolean dispatchTouchEvent(MotionEvent event) {
      final boolean intercepted;
      ...
      if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
      } else {
        intercepted = false;                
      }
      ...
      
      if(!intercepted){
        dispatchTransformedTouchEvent()
      }
      ...
}
```

这个事件是这样处理的，当ViewGroup的 `dispatchTouchEvent` 方法被调用的时候，它会先调用 `onInterceptTouchEvent` 方法，如果这个方法返回true，那么事件分发就结束了，否则的话，它会开始向子View分发消息，也就是在 `dispatchTransformedTouchEvent` 这个函数中做的事情。如果有子View处理了这个事件，那么就分发完了，如果没有子View处理那么它会
调用`super.dispatchTouchEvent`方法（ViewGroup的父类是View，这个方法会触发自父类的 `onTouchEvent`方法)。

可见ViewGroup的主要任务是负责给子View分发事件，它的 `onTouchEvent` 方法只有在触摸事件没有被处理的时候才会被调用，如果我们希望拦截ViewGroup中的事件，不能重载`onTouchEvent`而是应该重载`onInterceptTouchEvent` 方法（主观上的感觉ViewGroup的`onInterceptTouchEvent`更类似于View的`onTouchEvent`方法）。  

下面再看下回调事件，就是各种 `OnXXXXListener` 监听，View主要有三种监听事件（ViewGroup继承了这三种事件）

1.  View.OnClickListener 点击回调
2.  View.OnLongClickListener 长按回调
3.  View.OnTouchListener 触摸回调，手指在手机上落下、移动、离开都是触摸回调

对于View来说，通过 `View.OnTouchListener`拦截触摸事件并处理这个事件（onTouch函数返回true）会导致 `View.OnClickListener` 和 `View.OnLongClickListener` 失效。因为这个两个监听的判断是放在默认的 `onTouchEvent` 函数中实现的，处理这个监听会提前结束事件分发`onTouchEvent`将不会再被触发。 

对于ViewGroup来说，如上所见ViewGroup会优先处理子View的事件，设置上面的监听事件对ViewGroup的行为不会有太大影响，只有在子View没有处理触摸事件的情况下，ViewGroup才会触发它的`super.dispatchTouchEvent` 这个时候它的行为和一个普通View无异。

PS:`ViewParent.requestDisallowInterceptTouchEvent(boolean) `这个方法可以迫使实现ViewParent接口的父类无法通过`onInterceptTouchEvent` 方法拦截事件。 

##手势和滚动

真实的App 需要处理各种复杂的手势，比如下拉、上拉、拖动等等。但是如果依靠上面的知识来自己识别这些手势那也太麻烦了。Android中提供一个类`GestureDetector`来辅助识别这些手势操作，它会提供一个回调接口`GestureDetector.OnGestureListener`通过这个接口就可以直接处理各种手势事件了（这个回调接口包含很多方法，如果只需要处理个别手势的话，可以继承`GestureDetector.SimpleOnGestureListener`这个类，选择性的实现一些方法）。

```
//实现手势识别的回调接口，注意onDown方法必须返回true，否则系统认为放弃了余下手势的识别
class mListener extends GestureDetector.SimpleOnGestureListener {
   @Override
   public boolean onDown(MotionEvent e) {
       return true;
   }
}
mDetector = new GestureDetector(PieChart.this.getContext(), new mListener());
//在`onTouchEvent`方法中调用`mDetector.onTouchEvent(event)` 方法来获取触摸事件
@Override
public boolean onTouchEvent(MotionEvent event) {
   boolean result = mDetector.onTouchEvent(event);
   if (!result) {
       if (event.getAction() == MotionEvent.ACTION_UP) {
           stopScrolling();
           result = true;
       }
   }
   return result;
}
```

其实是非常简单的吧，如果是在ViewGroup中，需要在`onInterceptTouchEvent`方法中调用`mDetector.onTouchEvent(event)`原因不需要多说了吧。手势操作和UI效果永远是绑在一起的，这样才能有比较真实的物理效果。Android中的`Scroller`类提供了物理效果模拟的支持，这个类封装了滚动物理的计算。但是它只是负责做物理计算，计算的结果需要调用View的`View.scrollTo (int x, int y)` 或者`View.scrollBy (int x, int y)` 应用到View上面。`Scroller`类不会一直计算滚动的偏移位置，更新需要调用`computeScrollOffset` 方法。

```
if (mScroller.computeScrollOffset()) {
     // Get current x and y positions
     int currX = mScroller.getCurrX();
     int currY = mScroller.getCurrY();
    ...
 }
```

如此看来启动一个View的滚动效果，需要做如下几步：

1. 实例化一个Scroller对象，并启动滚动
2. 间接性的调用`mScroller.computeScrollOffset()方法，更新滚动位置，并重绘View
3. 通过View的`scrollTo`或者`scrollBy`方法，设置View的最新位置，新的位置可以通过 `mScroller.getCurrX()`和 `mScroller.getCurrY()`获取
4. 停止滚动


如何操作Scroller官方文档现在给出的两种方式：

1. 启动滚动之后(比如调用了`mScroller.fling`方法)立刻调用` postInvalidate()`方法刷新界面。然后每次在`onDraw`方法中计算滚动，并再次调用`postInvalidate()`方法，让整个动画循环下去
2. API11开始引入了一个类ValueAnimator,可以用它来循环计算滚动便宜，并通过`addUpdateListener()`回调来处理动画

对于第一种写法大概这样

```
    // 1.启动滚动
    public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY) {
        mScroller.fling(currentX, currentY, velocityX / SCALE, velocityY / SCALE, minX, minY, maxX, maxY);
        postInvalidate();
    }
    // 2.3.计算滚动，应用到View，并强制再次刷新(这样下次就可以再次回到这里计算滚动了)
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        ...
        if (!mScroller.isFinished()) {
                mScroller.computeScrollOffset();
                int currX = mScroller.getCurrX();
                int currY = mScroller.getCurrY();
                //TODO 把 currX 和 currY 应用到View上面
                postInvalidate();
        }
    }
    // 4. 再次触摸屏幕的时候停止滚动
    @Override
    public boolean onDown(MotionEvent e) {
        ..
        stopScrolling();
        ...
        return true;
     }
```

对于这种写法，我还见过一种是把onDraw中做的事，放在`View.computeScroll()`方法里面的，这个方法在源码中实际上会被`onDraw`方法调用到。

对于后一种，代码看起来就比较人性化了，但是必须是API11之后才可以使用

```
mScroller = new Scroller(getContext(), null, true);
       mScrollAnimator = ValueAnimator.ofFloat(0,1);
       mScrollAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
           @Override
           public void onAnimationUpdate(ValueAnimator valueAnimator) {
               if (!mScroller.isFinished()) {
                   mScroller.computeScrollOffset();
                   int currX = mScroller.getCurrX();
                   int currY = mScroller.getCurrY();
                   //TODO 把 currX 和 currY 应用到View上面
               } else {
                   mScrollAnimator.cancel();
                   onScrollFinished();
               }
           }
       });
```





内容来源:

1. [Input Events](http://developer.android.com/guide/topics/ui/ui-events.html)
2. [View](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.4_r1/android/view/View.java)
3. [ViewGroup](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.4_r1/android/view/ViewGroup.java)
4. [Making the View Interactive](http://developer.android.com/training/custom-views/making-interactive.html)
5. [ValueAnimator](http://developer.android.com/reference/android/animation/ValueAnimator.html)
6. [Scroller](http://developer.android.com/reference/android/widget/Scroller.html)
