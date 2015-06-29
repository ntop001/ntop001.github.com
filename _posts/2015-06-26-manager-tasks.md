http://www.slideshare.net/RanNachmany/manipulating-android-tasks-and-back-stack


两个问题

半透明Activity出现在Recent列表，用户体验差




通过Back/空白区域关闭ShareBoard的时候需要关闭Activity
点击Home键的时候需要关闭Activity取消操作过程，以免Activity出现在Recent列表中

dumpsys activity recents 打印最近任务
dumpsys activity activities 当前Activity task 
dumpsys activity processes


测试点

 Q1.startActivityForResult 之后，子Activity是新的任务还是和父Activity在一个任务中

使用 FLAG_ACTIVITY_NEW_TASK 和 taskAffinity 标记的Activity还会成为新的Task，但是使用SingleInstance标记的Activity 和父Activity在同一个Task中。如果使用  SingleTask 标记的话，会发现所有Activity都在一个Task中。


同时，如果被启动的sub-Activity真的运行在新的Task中了，那么这也会产生问题，onActivityResult 方法会立即执行（在sub-Activity运行之前）。

父 Activity 调用 startActivityForResult 然后finish， sub-activity 不会回调 onActivityResult 方法。
父 Activity 调用 startActivityForResult，然后调用 moveTaskToBack， 子Activity回调onActivityResult的同时会把父Activity带回前台。（栈的顺序变化）
如果 sub-activity 是新的任务，那么父Activity调用 moveTaskToBack 后也不会执行 onActivityResult。
如果父Activity自己是单例，那么它用 startActivityForResult 启动的Activity会嵌入到自己的Task中，如果是 startActivity 方式，那么会建立新的Task



 Q2. 如果一个App有两个任务，Recent 中查看到的是一个还是两个 ？可否把新的任务中的Task从Recent重移除，excludeRecent 怎么使用 ？

Android 中的 FLAG_ACTIVITY_NEW_TASK 并不会启动新的Task，除非添加了 taskAffinity 

 标签。两个Task都会出现在Android的Recent中(必须添加taskAffinity才能出现在Recent中，否则还是会和原App绑定一起

 )，可以使用 excludeFromRecents

 隐藏起来。



使用 launchMode = “singleInstance”  标记的Activity会启动一个新的Task，而这个Task中只有一个Activity，就是自己。而且会发生奇妙的事情。

A, B( singleInstance，excludeFromRecents= true ),C 	启动顺序 ： A -> B -> C

启动B的时候查看Recents会发现Recent为空。。。然后启动C，查看Task列表会发现

C(task#0),A(task#0),B(task#1),  队列顺序发生变化，B 进入新的Task，C 回到原来的Task，并且把这个Task的其他Activity拍到了B的前面。这时候Back键后退，会发现顺序是：  C -> A -> B .

避免上面奇怪现象的发生，可以同时给B设置 ( singleInstance，excludeFromRecents= true, taskAffinity ) 这样，会先分离出来B为独立Task，然后通过 excludeFromRecents 可以隐藏掉。



dumpsys 使用


正常的3各：Running activities (most recent first):
      TaskRecord{2c048081 #4 A=com.example.managetask U=0 sz=3}

        Run #2: ActivityRecord{3a7134b0 u0 com.example.managetask/.ActivityWorld t4}

        Run #1: ActivityRecord{257fac0a u0 com.example.managetask/.ActivityHello t4}

        Run #0: ActivityRecord{6f882dc u0 com.example.managetask/.MainActivity t4}







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



