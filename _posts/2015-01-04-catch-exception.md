---
layout: post
title: "捕获Android中的崩溃信息"
description: ""
category: "知识整理"
tags: [SDK,Android]
---

App运行过程中突然崩溃了，崩溃时候的堆栈信息对于修复bug异常重要，在Android中有多种方式可以捕获这些错误信息。

## 通过java自身机制捕获错误

在 `java.lang.Thread` 下面提供了一个接口 `Thread.UncaughtExceptionHandler` 这个接口可以用来捕获线程异常结束时候的状态。Thread类
提供了静态方法 `setDefaultUncaughtExceptionHandler (Thread.UncaughtExceptionHandler handler)` 来为线程设置默认的处理异常的回调。

```
public class CrashHandler implements UncaughtExceptionHandler {
	public CrashHandler() {
		Thread.setDefaultUncaughtExceptionHandler(this);
	}
	
	@Override
	public void uncaughtException(Thread thread, Throwable ex) {
		//TODO
	}
}
```

在 `uncaughtException(Thread thread, Throwable ex)` 中就可以处理函数崩溃时候的异常信息了，使用这种方式获取异常信息只能获取
到java层面的崩溃信息，如果是由native代码（c/c++）产生的异常是无法捕获到的，在Android中每个程序都是运行在自己的虚拟机中的，对于Android的Linux内核而言，每个虚拟机只是一个Linux程序而已，我们写的C/C++代码和虚拟机是并列的存在。而java代码是运行在虚拟机里面的，它不知道外部的其他native代码的运行情况。


## 通过读取Logcat输出过滤错误

这种想法源自于我们在debug的时候，都是通过logcat来查看程序运行时信息的。那么完全可以在Android程序中运行logcat命令，然后过滤其中的错误信息，找到自己的程序输出的错误日志。

```
//进入android
sdk>adb shell
shell@android:/ $
//打开logcat
shell@android:/ $ logcat
```

这样就打开logcat了，但是这时候输出的是全部的日志（包含此时运行的所有程序）我们需要按照包名和错误类型对logcat中输出的全部日志进行过滤。

```
shell@android:/ $ logcat -h
logcat -h
unknown option -- hUnrecognized Option
Usage: logcat [options] [filterspecs]
options include:
  -s              Set default filter to silent.
                  Like specifying filterspec '*:s'
  -f <filename>   Log to file. Default to stdout
  -r [<kbytes>]   Rotate log every kbytes. (16 if unspecified). Requires -f
  -n <count>      Sets max number of rotated logs to <count>, default 4
  -v <format>     Sets the log print format, where <format> is one of:

                  brief process tag thread raw time threadtime long

  -c              clear (flush) the entire log and exit
  -d              dump the log and then exit (don't block)
  -t <count>      print only the most recent <count> lines (implies -d)
  -g              get the size of the log's ring buffer and exit
  -b <buffer>     Request alternate ring buffer, 'main', 'system', 'radio'
                  or 'events'. Multiple -b parameters are allowed and the
                  results are interleaved. The default is -b main -b system.
  -B              output the log in binary
filterspecs are a series of
  <tag>[:priority]

where <tag> is a log component tag (or * for all) and priority is:
  V    Verbose
  D    Debug
  I    Info
  W    Warn
  E    Error
  F    Fatal
  S    Silent (supress all output)

'*' means '*:d' and <tag> by itself means <tag>:v

If not specified on the commandline, filterspec is set from ANDROID_LOG_TAGS.
If no filterspec is found, filter defaults to '*:I'

If not specified with -v, format is set from ANDROID_PRINTF_LOG
or defaults to "brief"
```

这是logcat提供的所以可用命令，我们需要这样的命令

```
shell@android:/ $ logcat -d -v raw -s AndroidRuntime:E com.xxx.xxx
```

1. `-d` 输出log然后关闭
2. `-v` 设置输出格式 raw
3. `-s` 禁止其他输出，然后按照 `AndroidRuntime:E com.xxx.xxx`这些过滤条件过滤
4. `AndroidRuntime:E` 程序崩溃时候打印的log都会有这个标记，所以用它作为一个过滤条件

这样就可以过滤出我们需要的错误信息了，在java代码中如果需要运行这些命令需要使用Runtime类

```
Runtime.getRuntime().exec(...)
```

通过这种方式获取错误信息，需要在App中设置`READ_LOGS`权限，这种方式的好处是不仅仅可以拿到程序崩溃时候的信息，只要是打印在logcat中的信息都可以取到。缺点是实时性不好，可能会错过App某次崩溃时候打印的log。

## 捕获native(C/C++)代码的崩溃日志

一种方案是使用logcat过滤native代码的日志，但是logcat中打印的native代码产生的崩溃信息没有区分性，不容易过滤。另一种方案是在native层写代码捕获崩溃信息，具体的实现没有研究过，但是解铃还须系铃人这应该是可行的一种方案。

