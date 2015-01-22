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

##手势和滚动

