---
layout: post
title: "线程、线程池和Handler"
description: "总结SDK中用到的线程池知识"
category: "知识整理"
tags: [SDK]
---

在统计分析SDK里面提供的多数onXXXX() 开头的方法都是异步的，最开始的时候都是直接用线程实现的，用来处理琐碎的任务，然后在丢到
用HandlerThread构造的Handler里面的处理文件和网络这些比较重型的任务，后来觉得Handler还是有些局限的又回到线程池的处理方式。

## 线程

Java的线程实现实际上是一个[轻量级的进程(LWP)](http://en.wikipedia.org/wiki/Light-weight_process)，使用起来又是非常简单，有时候会
给人一种协程的感觉，几行代码就可以启动线程了

```
new Thread(){
			public void run(){
				System.out.println("hehe");
			}
		}.start();
```
但事实上启动线程是非常耗时且浪费性能的操作，在其他语言(go,lua..hehe 我知道的)中有些提供协程的概念,个人认为这种异步更适合于处理
这种不太复杂的异步任务。

`Thread` 类中有几个方法始终让我忘了又查查了又忘

1. wait() & notify() 继承自Object但是线程线程相关
2. resume() , suspend() & stop() 看上去很有诱惑力的self-explained 的名字，都是 Deprecated 的，哈哈
2. join() 阻塞当前线程直到调用线程执行完毕
3. yield() & sleep() 

#### wait() & notify()

这一组方法给人的感觉是 obj.wait() 当前线程停止，obj.notify() 当前线程继续执行,还是测试一下吧

```
    final Object obj = new Object();
		final Thread t = new Thread(){
			public void run(){
				try{
					Thread.sleep(2*1000);
				}catch(Exception ignore){}
				
				System.out.println("child Thread");
				try{
					obj.notify();
				}catch(Exception ignore){}
			}
		};

		t.start();
		try{
			obj.wait();
		}catch(Exception e){
			e.printStackTrace();
		}
		System.out.println("main Thread");
```
哎，出错了，在执行 obj.wait() 的时候发生了异常，搜索之后（notify的函数说明也有写）发现执行这个方法需要当前线程获得这个对象的监视
器(object's monitor)才行,那什么是 `object's monitor`呢？在Java中每个对象都有monitor功能，当调用 `synchronized(obj){ ... }`的时候
当前线程就获得了这个对象的监视器，我们更常用的词汇是对象锁。拥有同一个对象锁的两个代码块（分别在两个线程，同一个线程本来就不存在
竞争问题）是互斥的，不能同时执行。比如：

```
final Object o = new Object();
new Thread(){
  public void run(){
    synchronized(o){
      //Block A
    }
  }
}.start();

synchronized(o){
  //Block B
}
```
A 和 B 这两个代码块分别在两个线程中，但是是不能同时执行的，他们受o这个对象锁的保护。关于monitor见[这里](http://en.wikipedia.org/wiki/Monitor_%28synchronization%29#Implicit_condition_variable_monitors)更多解释.

修改后的代码如下：

```
final Object obj = new Object();
		final Thread t = new Thread(){
			public void run(){
				try{
					Thread.sleep(2*1000);
				}catch(Exception ignore){}
				
				System.out.println("child Thread");
				try{
					synchronized(obj){
						obj.notify();
					}
				}catch(Exception ignore){}
			}
		};

		t.start();
		try{
			synchronized(obj){
				obj.wait();
			}
		}catch(Exception e){
			e.printStackTrace();
		}
		System.out.println("main Thread");
```
打印结果：

```
child Thread
main Thread
```

首先线程启动，主线程执行到 `synchronized(obj){ obj.wait(); }` 获得对象锁，开始执行 `obj.wait()`开始等待，wait方法会主动释放对象锁
，过了2秒子线程睡眠结束，打印 `child Thread` 执行
到 `synchronized(obj){obj.notify();}` 子线程开始获得obj对象的锁，执行 `obj.notify()` 监视器(obj对象)通知主线程恢复执行，主线程
继续执行打印 `main Thread`。

#### resume() , suspend() & stop()

这组方法已经标记 @Deprecated 了，所以也再看了

#### join()

这个方法给人的感觉是让调用线程（调用join这个方法的线程）加入到当前线程，比较迷惑的哪个是调用线程哪个是当前线程。

```
public static void main(String[] args) throws Exception{
		final Thread t = new Thread(){
			public void run(){
				try{
					Thread.sleep(2*1000);
				}catch(Exception ignore){}
			
				System.out.println("child Thread");
			}
		};

		t.start();
		try{
			t.join();
		}catch(Exception ignore){}
		System.out.println("main Thread");
	}
```
正在执行 `main` 函数的线程就是当前线程（主线程），t 就是调用线程，所以结果就是 t 把当前的执行加入到了main中，在调用
join() 函数的那一刻起，主线就开始阻塞直到 t 执行完，才会继续执行。

####  yield() & sleep() & wait()

wait() 方法是 Object 类提供的，调用前提是当前线程已经获得了对象锁，并且调用之后会释放调用锁进入等待，sleep() 方法不会
对对象锁产生任何影响。所以一般wait方法用在多线程通信，而sleep方法用在让当前线程休眠一会，yield() 文档解释是放弃当前的CPU分配
的执行时间片，但是如果当前没有其他线程或者优先级更高的线程，那么当前线程还是会执行的。

[这里](http://javarevisited.blogspot.com/2011/12/difference-between-wait-sleep-yield.html)有个非常详细的解释.

## 线程池

在 `java.util.concurrent` 包下是并发和线程池的相关类，线程池相关的服务都是以接口的形式提供的

接口是最能体现一个类的行为的和特征的，并发包中提供如下依次继承的接口

1. Executor 只有一个 execute 方法，可以用来执行一个Runnable任务
2. ExecutorService 继承自 Executor，提供管理队列和调度任务管理的方法
3. ScheduledExecutorService 继承自 ExecutorService， 提供延迟和周期任务的支持

并发包中的 ThreadPoolExecutor 和 ScheduledThreadPoolExecutor 分别实现了2,3两个接口，提供了具体的实现，在使用的时候
一般使用 Executors 提供的工厂方法来构造一个类，它提供几种线程的配置选择

1. newCachedThreadPool() 有可用线程吗，没有创建新的，有则复用旧的
2. newFixedThreadPool(int nThreads) 创建固定数量的线程，如果没有足够可用线程会缓存任务等待知道有可用线程为止
3. newScheduledThreadPool(int corePoolSize) 创建拥有延迟或者周期任务的线程池

这些方法还提供了只使用一个线程的版本 `newSingleThreadExecutor()` 和 `newSingleThreadScheduledExecutor()` 我的SDK中
主要使用了这两类线程池。

#### Runnable VS Callable

Runnable 大家都比较熟悉，只有一个run接口是需要实现的。Callable 和Runnable 唯一的不同就是 Callable 可以返回一个结果。
注意接口 `void	run()` 和 ` V	call()` ,  使用 execute 接口执行的任务是没有返回值的，使用 `submit` 执行的任务都会有返回值
如  

1. Future<T> submit (Callable<T> task)
2. Future<?> submit (Runnable task)

Future 代表这个任务的执行结果，调用 Future 的 get 方法会阻塞当前线程直到任务执行完毕，注意的是使用 Callable 提交的任务
调用 get 会返回执行结果 `result = exec.submit(aCallable).get()` 使用 Runnable 执行的任务get方法返回null。

#### 任务调度

通常建立一个线程池之后比如:`ExecutorService executor = Executors.newSingleThreadExecutor();` 可以使用 execute 或者
submit（如果需要监控任务的执行状态） 来执行一个任务。关闭一个线程池可用用 `executor.shutdown` 来关闭, 已经关闭的线程池
是不能再次提交任务的。

```
public static void main(String[] args) throws Exception{
		ExecutorService executor = Executors.newSingleThreadExecutor();
		
		//异步执行
		executor.execute(new HeheRunnable());
		
		//阻塞执行
		System.out.println("runnable result: " + executor.submit(new HeheRunnable()).get() );
		System.out.println("callable result: " + executor.submit(new HeheCallable()).get() );
		//关闭线程池
		executor.awaitTermination(2, TimeUnit.SECONDS);
		System.out.println("main Thread");
	}
	
	static class HeheRunnable implements Runnable{

		@Override
		public void run() {
			System.out.println("runnable, hehe");
		}
	}
	
	static class HeheCallable implements Callable<String> {

		@Override
		public String call() throws Exception {
			System.out.println("callable, hehe");
			return "Task complete";
		}
	}
```

实现 `ScheduledExecutorService` 接口的线程池提供延迟任务和周期任务的功能 `schedule(Runnable command, long delay, TimeUnit unit)` 
延迟一定事件执行一个任务。在统计SDK中许多App拥有定时启动或者推送功能会导致App在同一时间启动，App启动后会向服务器发送日志导致
类似DDOS攻击的效果。SDK的一个策略就是随机一个时间让日志在发送。这样可以减缓这种DDOS的效果。

```
int delay = new Random().nextInt(10*60);
executor.schedule(aRunnable, delay , TimeUnit.SECONDS);
```
这样可以把请求平均分布在 10 分钟以内。

## Threadpool VS HandlerThread

相对于Java线程池 HandlerThread(一般配合Handler使用) 更像一个针对特殊场景(Android) 定制的一个线程池

1. 只有一个后天线程
2. 可以发送消息和处理消息
3. 可以执行 Runnable 任务和延迟任务

但是相对于线程池，后台线程数量无法定制，不能获取任务执行状态。如果你不需要太多的控制那么Handler是一个
不错的选择。

```
HandlerThread ht = new HandlerThread("hehe");
ht.start();
mHandler = new Handler(ht.getLooper());
```






