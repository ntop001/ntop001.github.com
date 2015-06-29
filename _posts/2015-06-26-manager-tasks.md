---
layout: post
title: "Android Task 管理"
description: ""
category: 
tags: [Android,Task]
---

一般情况下，程序员很少会关心Task的运行，最近也是业务需求才多看了一些。主要内容来自这个[PPT](http://www.slideshare.net/RanNachmany/manipulating-android-tasks-and-back-stack). 这个项目的AndroidSDK，里面存在太多的内存泄露，泄露点非常多，而且代码比较冗杂，改起来比较麻烦所以想了一个取巧的办法，在后台隐藏一个透明的Activity做载体，每次需要用到Activity的时候都把这个Activity供出来，同时把这个Activity做成 “单例” 这样即使泄露也只会泄露一个Activity。

基于上面的需求我的代理Activity需要有如下功能，

1. 透明 不能让用户觉察到个Activity的存在。
2. 单例 不能被Back或者其他方式被回收
3. 可以隐藏在后台 这样就不会覆盖在其他页面前面，影响用户操作

我是这样解决的，把Activity的启动模式设置为`launchMode="singleInstance"` 这样这个Activity就会唯一的存在于一个单独的Task中，然后利用 `moveTaskToBack` 方法把这个Task移到后台（注意如果这个Activity不是独立的Task而是和App的Task在一起，那么这个方法会把整个App移到后台）。在需要使用Activity的时候再把Activity挪到前台，但是要覆盖 `finish` 方法 

```
public void finish(){
 moveTaskToBack(true);
}
```

这样Activity就不会被回收，需要的时候再挪到前台。看似“聪明“的方案，实际上存在很多问题，所以有了下面的探索。

## 如何查看Activity所在的Task信息

进入模拟器的shell(>adb shell),Android SDK 提供了 `dumpsys activity` 命令来打印Activity的状态。敲入这个命令会打印很多信息，也可以通过下面的命令打印需要的信息。

1. dumpsys activity recents 打印最近任务
2. dumpsys activity activities 当前Activity 状态
3. dumpsys activity processes 打印进程状态

eg：

```
//正常 App 的所有Activity生存在一个Task中
Running activities (most recent first):
      TaskRecord{2c048081 #4 A=com.example.managetask U=0 sz=3}

        Run #2: ActivityRecord{3a7134b0 u0 com.example.managetask/.ActivityWorld t4}

        Run #1: ActivityRecord{257fac0a u0 com.example.managetask/.ActivityHello t4}

        Run #0: ActivityRecord{6f882dc u0 com.example.managetask/.MainActivity t4}
```

## Android中如何启动新的任务

经常使用的方法是在Intent中添加 `FLAG_ACTIVITY_NEW_TASK`  标签。但是真实的情况是这样是无法启动新的Task的，还需要给对应的Activity添加taskAffinity属性才能把这个Activity启动到新的额Task中。

还有两种方法也能启动新的Task，给Activity配置`launchMode`属性，可选值为 

* `signleInstance`  新的Task中只会有这一个Activity
* `singleTask` 新的Task中此Activity是最底层，而且整个Task只会有一个实例

上面两种模式只会产生一个Task实例，如果再次启动这个Activity的话，会触发`onNewIntent`方法。

## 如果一个App有两个任务（Task），Recents 中查看到的是一个还是两个 ？可否把新的任务中的Task从Recent重移除，excludeRecent 怎么使用 ？

> Recents 就是在Android手机上按下最右的虚拟按键时，显示的最近打开过的App

如果一个App产生了两个（或多个）Task，那么默认情况下Recents中只会出现一个App的快照。但是如果其中一个Task有不同的 taskAffinity（Task 的 taskAffinity 来自于它的根Activity的taskAffinity属性）， 否则还是会和原App绑定一起。可以使用 `excludeFromRecents="true"` 属性把Task从Recent中隐藏起来。


## 奇妙的事情

使用 launchMode = “singleInstance”  标记的Activity会启动一个新的Task，而这个Task中只有一个Activity，就是自己。
如果有三个Activity，A, B( singleInstance ),C 	启动顺序 ： A -> B -> C

启动B的时候查看Recents会发现Recent为空 （既找不到A也找不到B）。。。然后启动C，查看Task列表会发现

C(task#0),A(task#0),B(task#1),  队列顺序发生变化，B 进入新的Task，C 回到原来的Task，并且把这个Task的其他Activity拍到了B的前面。这时候Back键后退，会发现顺序是：  C -> A -> B .

避免上面奇怪现象的发生，可以同时给B设置 ( singleInstance，excludeFromRecents= true, taskAffinity ) 这样，会先分离出来B为独立Task，然后通过 excludeFromRecents 可以隐藏掉。

## startActivityForResult 

之前所有的启动Activity的方式默认都是用的startActivity方法。但是如果使用 startActivityForResult 方法又会出现不一样的结果。

使用 FLAG_ACTIVITY_NEW_TASK 和 taskAffinity 标记的Activity还会成为新的Task。但是使用SingleInstance标记的Activity 和父Activity在同一个Task中（使用  SingleTask 标记的话，会发现所有Activity都在一个Task中）。

同时，如果被启动的sub-Activity真的运行在新的Task中了，那么这也会产生问题，onActivityResult 方法会立即执行（在sub-Activity运行之前）。

下面记录几种情况

1. 父 Activity 调用 startActivityForResult 然后finish， sub-activity 不会回调 onActivityResult 方法。
2. 父 Activity 调用 startActivityForResult，然后调用 moveTaskToBack， 子Activity回调onActivityResult的同时会把父Activity带回前台。（栈的顺序变化）
如果 sub-activity 是新的任务，那么父Activity调用 moveTaskToBack 后也不会执行 onActivityResult。
如果父Activity自己是单例，那么它用 startActivityForResult 启动的Activity会嵌入到自己的Task中，如果是 startActivity 方式，那么会建立新的Task








Q3

测试 Activity 被回收的情况下，对话框显示是否正常，被回收的Activity是否还可以使用

在有显示对话框的Activity中直接关闭Activity是没有问题的，Activity会先关闭对话框再关闭自己。但是如果Activity被系统回收或者是通过其他方式关闭了，会导致Activity的窗口泄漏。

06-29 05:02:35.563: E/WindowManager(2793): android.view.WindowLeaked: Activity com.example.managetask.ActivityHello has leaked window com.android.internal.policy.impl.PhoneWindow$DecorView{fa91d70 V.E..... R......D 0,0-729,360} that was originally added here

所幸程序不会崩溃。

如果用WeakReference引用Activity，那么在没有GC前都是可以拿到Activity实例的，但是调用 isFinishing or isDestroyed 方法就会发现，实例已经回收。

如果Activity没有回收，那么是可以操作Activity的API，比如在这个Activity上显示对话框，虽然并看到不到这个Activity。但是跳转回来的时候，会发现他又对话框。

是否可以利用回收的Activity做事情？

用已经回收的Activity显示对话框会 leakwindow （runOnUiThread(Runnable{ new Dialog().show()})）但是程序不会崩溃。
可以用 leakActivity显示一个Toast
可以启动 启动一个Activity （startActivity(Intent)）
可以操作UI




Q4 

静态引用
AsyncTask 线程引用
activity.getSystemService(“”) cause memory leaks 系统服务
Toast 也会 https://code.google.com/p/android/issues/detail?id=1770

Q5


PS：Android 5.1 系统
