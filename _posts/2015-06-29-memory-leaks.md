---
layout: post
title: "Android内存泄漏"
description: ""
category: 
tags: []
---

老生常谈的话题，但是真正写代码的时候又很少有人能够认真对待。就拿[上篇](http://ntop001.github.io/2015/06/26/manager-tasks/)开始提到的分享SDK来说，各种内存泄漏，已经无语，所以写代码的时候还是要自己认真注意，而不是嘴上说说。

在考虑内存泄漏之前，先看一个问题，为什么Android码农爱持有Activity，在我们SDK中的绝大多数时候是用来显示对话框的（99.9%），其次有小部分是因为我们SDK依赖的第三方SDK需要。

## 被回收的Activity还可以做什么？

这里所指的回收是指Activity被从栈中移除，已经调用了 `onDestroy()` 方法。如果调用 `isFinishing()` 方法会返回 `true`。但是Activity的引用还被持有且不为空。

1. 用已经回收的Activity显示对话框会 leakwindow （runOnUiThread(Runnable{ new Dialog().show()})）但是程序不会崩溃，对话框也不会真的显示出来。
2. 可以用 leakActivity 显示一个Toast
3. 可以启动 启动一个Activity （startActivity(Intent)）
4. 可以操作UI,但是因为Activity已经回收了，导致没有效果。
5. 可以通过 leakActivity.getApplicationContext() 获取一个真正的Context

一个被回收的Activity还是可以做很多事情的，但是UI相关的东西基本上处于无用状态，所以对于大部分只是为了操作UI（比如显示对话框）而强引用了一个Activity其实并无卵用，这时使用WeakReference持有Activity已经足够，在Activity还在的时候做该做的事情，在它回收的时候也失去了UI操作的能力。

后来我在SDK加了很多这样的代码

```
//mActivity 是若引用
Activity activity = mActivity.get();
if( activity != null && activity.isFinishing() ) {
 //do something
}
```

## 典型的内存泄漏

在修bug的时候就不断的感慨我们已经把市面上的所有泄漏类型都涵盖了。。。sigh

```
Toast.makeText(this, "出错啦！", Toast.LENGTH_SHORT).show();
//Toast.makeText(MyActivity.this, "出错啦！", Toast.LENGTH_SHORT).show();
```

`Toast` 的第一个参数是 `Context`，大多数时候我们会像上面那样使用，但是很少会出现内存泄漏，但是如果在 `onDestroy()` 方法里面调用就会发现在很短的一段时间（几秒）Activity是泄漏的，原因是 `Toast.makeText` 方法持有了Activity（没有看源码，感觉Toast消失后就会释放Activity所以一般不会有太大的问题，为了安全起见最好还是传入 Applicaton Context）。


```
LocationManager mLocationManager = (LocationManager) activity.getSystemService(Context.LOCATION_SERVICE);
```

这算是系统bug，如果我们使用Activity来获取一个系统服务，那么这个服务会持有当前Activity，解决的办法还是 Application Context。

```
//直接敲的代码，有语法错误，当伪码看吧
public class MYActivity extends Activity {
    public static XXXListener listener;
    
    public void onCreate(..){
       ...
       listener = new XXXListener(){
           ....
       }
    }
}
```

这段代码也是我们SDK中的一个例子，XXXListener 的匿名实现引用了外部类 MYActivity ，然而它确是一个静态变量，这样就导致 `MYActivity.listener` 这个静态的变量持有了Activity实例, sigh...

```
//直接敲的代码，有语法错误，当伪码看吧
public class MYActivity extends Activity {
    public void onCreate(..){
       new Thread( new Runnable(){
           public void run(){
              //可以运行一万年的任务
             ...
           }
       }
       ).start();
    }
}
```

我们在使用AsyncTask、线程、Handler的时候经常犯这个错误，把一个内部类传给了一个需要运行好久的任务，然而这个任务因为是内部类的缘故会持有外部的Activity。大部分时候我们使用匿名内部类都会遇到上面的两种情况，并且出个bug也不自知。。。所以写代码的时候一定要小心内部类。


TODO：Fragment也会被泄漏的，但是虽然可以从 Fragment拿到Activity，但是Fragment泄露不会导致Activity被泄漏，可能内部有机制处理吧

## 最小需求

在解决SDK中内存泄漏的时候发现，大部分时候SDK需要的参数是Context而不是Activity，但是我们都传入了Activity实例。而且方法声明没有明确究竟是需要Activity还是Context作为参数，这样的函数写了很多，导致单纯的看代码很难发现问题。

所以规范很重要，方法声明是Context的函数，千万不要传入Activity，同时方法内部也不要直接持有传入的参数

```
/**
* 使用的时候也要 foo(context.getApplicationContext())
*/
public void foo(Context context ){
    Context ctx = context.getApplicationContext(); 
}
```

需要Activity的地方，把参数声明为Activity

```
/**
* 明确告诉调用者，这里需要的是Activity
*/
public void foo(Activity activity){
  //如果需要长期持有Activity，使用弱引用
}
```

## 工具

我们SDK之前一直是没有问题，最近出现一堆开发者给我们报内存泄漏的问题，我们的工程师调查完发现，原来他们正在使用一个NB的检测工具 [leakcanary](https://github.com/square/leakcanary),所以我们的问题也暴露了，现在我们的方案是把这款工具加入我们的测试。

另外一个工具是[MAT](http://www.eclipse.org/mat/),[之前](http://ntop001.github.io/2015/04/02/abcd/)有写过。使用leakcanary有时候会把短时间的内存泄漏报出来，比如运行一个3秒钟的线程并持有已经回收的Activity的引用，但是我们写代码的时候知道是可以忍受的。这时候用MAT分析更好一些。

## Activity 和 对话框

以上说的都是在 Android 5.0 上做的实验。

如果在当前Activity显示一个对话框，对话框上有个按钮可以直接调用 `finish()` 关闭Activity，这样是没有问题的，
Activity会先关闭对话框再关闭自己。

如果一个Activity正在显示一个对话框但是不再前台，而我们通过其他的方法调用了`finish`方法（或者系统回收或者是通过其他方式关闭了），这样会导致Activity的窗口泄漏。

```
06-29 05:02:35.563: E/WindowManager(2793): android.view.WindowLeaked: Activity   com.example.managetask.ActivityHello has leaked window com.android.internal.policy.impl.PhoneWindow$DecorView{fa91d70 V.E..... R......D 0,0-729,360} that was originally added here
```
所幸程序不会崩溃。

如果用WeakReference引用Activity，那么在没有GC前都是可以拿到Activity实例的，但是调用 isFinishing or isDestroyed 方法就会发现，实例已经回收。

如果Activity没有回收，那么是可以操作Activity的API，比如在这个Activity上显示对话框，虽然并看到不到这个Activity。但是跳转回来的时候，会发现他又对话框。

