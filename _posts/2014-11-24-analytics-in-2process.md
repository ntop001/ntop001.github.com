---
layout: post
title: "如何在多个进程中调用统计分析代码"
description: "尝试解决统计分析不支持在多进程中调用的问题"
category: 
tags: [SDK]
---

在Android开发中，现在越来越多的APP会启动多个进程，有的是把Service作为一个单独进程来使用有的是把界面分割到不同的进程中去，
Android做这些是非常简单的，只要在 `AndroidManifest.xml` 中添加 `android:process=":xxxx"` 标签就行了，比如：

```
 <activity 
    android:name=".SecActivity"
    android:process=":abcd"
  ></activity>
```
这样这个Activity就会运行在新的进程中(:abcd 是新的进程的后缀名)。但是友盟的统计分析SDK在实现的时候并没有考虑到多进程的情况，
所以现在是无法支持多进程的，如果有两个进程中同时调用统计分析的代码，并且都去操作一个文件，那么这个文件就会因为两个进程的竞争
读写而损坏。而且两个进程中都会生成MobclickAgent的实例，白白浪费资源。我能想到的一种解决方式是这样的，让统计分析的SDK单独运行在
一个进程中（这样就可以解决多进程竞争读写文件的问题），然后在另外的进程中通过Intent将数据发送到统计分析所在的进程。

## 封装一个Service提供统计代码运行的空间

实现一个Service，这个Service独立运行在一个进程中，并且定义协议用来通过Intent和其他进程通信。这个Service接收三种类型的Action

1. resume 映射 onResume 方法 
2. pause  映射 onPause 方法
3. event  映射 onEvent 方法

每种类型需要的其它数据通过固定的key传递，大概代码如下：

```
public class AnalyticsService extends Service{
	public static String RESUME = "resume";
	public static String PAUSE = "pause";
	public static String EVENT = "event";
	
	@Override
	public void onCreate() {
		super.onCreate();
		MobclickAgent.setDebugMode(true);
		//自己传入页面数据，而不是使用自动的页面路径捕获
		MobclickAgent.openActivityDurationTrack(false);
		
		Log.i("--->", "start Analytics Service");
	}

	@Override
	public int onStartCommand(Intent intent, int flags, int startId) {
		
		String action = intent.getAction();
		
		if(RESUME.equals(action)){
			MobclickAgent.onResume(this);
			//页面数据通过 "page" 传递
			MobclickAgent.onPageStart(intent.getStringExtra("page"));
		}else if(PAUSE.equals(action)){
		  //页面数据通过 "page" 传递
			MobclickAgent.onPageEnd(intent.getStringExtra("page"));
			MobclickAgent.onPause(this);
		}else if(EVENT.equals(action)){
		  //事件Id 通过 "id" 传递
			String id = intent.getStringExtra("id");
			//key-value型的map数据通过 Bundle 传递
			Bundle b = intent.getBundleExtra("kv");
			Map<String,String> m = new HashMap<String,String>();
			
			for(String key: b.keySet()){
				m.put(key, b.getString(key));
			}
			//数值型数据通过 "ct" 传递
			if(intent.hasExtra("ct")){
				int value =  intent.getIntExtra("ct", 0);
				MobclickAgent.onEventValue(this, id, m, value);
			}else{
				MobclickAgent.onEvent(this, id, m);
			}
		}
		
		return super.onStartCommand(intent, flags, startId);
	}

	@Override
	public IBinder onBind(Intent intent) {
		// TODO Auto-generated method stub
		return null;
	}
}
```

在AndroidManifest.xml文件中配置让这个Service运行在单独进程中，并接收我们定义的Action

```
    <service 
		    android:name="com.example.hellobigworld.analytics.AnalyticsService"
		    android:process=":AnalytiServcsService"
		    >  
		    <intent-filter>
                <action android:name="resume" />
                <action android:name="pause" />
                <action android:name="event" />
            </intent-filter>
		</service>
```

## 定义一个工具类，提供进程间通信的通道

这个类主要是把原有的 `MobclickAgent.onResume` 、 `MobclickAgent.onPause` 还有各种 `MobclickAgent.onEvent` 方法替换成新的
版本，这个版本的方法不调用任何MobclickAgent提供的接口(注意一旦调用了MobclickAgent的方法，那么这个类就会在这个进程中加载这
是我们不希望看到的)。新的版本只会通过Intent把数据发送到上面的 `AnalytiServcsService` 由Service来处理统计事件。

```
public class Analytics {
	private static Analytics instance = null;
	private Context context;
	
	private Analytics(Context c){
		context = c.getApplicationContext();
	}
	
	public synchronized static Analytics getInstance(Context c){
		if(instance == null){
			Log.e("--->", "Analytics initialized..");
			instance = new Analytics(c);
		}
		
		return instance;
	}
	//新版的 onResume 方法，只要传入页面名称就可以了
	public void onResume(String page){
		Intent in = new Intent();
		
		in.setAction("resume");
		in.putExtra("page", page );
		
		context.startService(in);
	}
	//新版的 onPause 方法，只要传入页面名称就可以了
	public void onPause(String page){
		Intent in = new Intent();
		
		in.setAction("pause");
		in.putExtra("page", page );
		
		context.startService(in);
	}
	//新版的onEvent方法
  public void onEvent(String tag){
		Intent in = new Intent("event");
		in.putExtra("id", tag);
		in.putExtra(tag, "");
		
		context.startService(in);
	}
	
	public void onEvent(String tag, String label){
		Intent in = new Intent("event");
		in.putExtra("id", tag);
		in.putExtra(tag, label);
		
		context.startService(in);
	}
	
	public void onEvent(String id, Map<String,String> m){
		Intent in = new Intent("event");
		in.putExtra("id", id);
		
		Bundle map = new Bundle();
		for(Map.Entry<String, String> entry : m.entrySet()){
			map.putString(entry.getKey(), entry.getValue());
		}
		
		in.putExtras(map);
		
		context.startService(in);
	}
	
	public void onEventValue(String id, Map<String,String> m, int value){
		Intent in = new Intent("event");
		in.putExtra("id", id);
		in.putExtra("ct", value);
		
		Bundle map = new Bundle();
		
		for(Map.Entry<String, String> entry : m.entrySet()){
			map.putString(entry.getKey(), entry.getValue());
		}
		
		in.putExtras(map);
		
		context.startService(in);
	}
}
```
`onEvent(String tag)` 和 `onEvent(String tag, String label)` 的实现是比较hacky的，以为在SDK中实际上是以 `tag` 为事件id的，同
时把tag作为k-v型事件的一个key，label作为value，如果没有value则传 ""。所以我们这个工具类中定义了这些方法，并通过 `startService`
方法，把数据传入 `AnalytiServcsService`。

## 调用

分别用两个Activity来演示，并且这两个Activity生存在不同的进程中，他们的集成代码都是一样的

```
  public void onResume(){
		super.onResume();
		Analytics.getInstance(this).onResume("SecondActivity");
	}
	@Override
	protected void onPause() {
		super.onPause();
		Analytics.getInstance(this).onPause("SecondActivity");
	}
```
但是他们在 AndroidManifest.xml 文件中声明跑在不同进程中 

```
<activity
            android:name="com.example.hellobigworld.MainActivity"
            android:label="@string/app_name" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        
        <activity 
            android:name=".SecActivity"
            android:process=":abcd"
            ></activity>
```

这么做大概可以实现在不同进程中调用统计分析的代码了，但是如上所见并没有管理Service的声明周期，所以获取还需要适当的时候结束掉Service。




