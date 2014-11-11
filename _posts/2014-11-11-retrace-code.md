经常有人发邮件问我们一些很奇怪的问题，还会附上一段堆栈代码：

> 为什么友盟统计到的错误堆栈都变成 a.b.c 这样的不可读字符了?

```
11-11 09:36:42.262: E/AndroidRuntime(2065): Caused by: java.lang.ArithmeticException: divide by zero
11-11 09:36:42.262: E/AndroidRuntime(2065): 	at com.example.hellobigworld.a.b(Unknown Source)
11-11 09:36:42.262: E/AndroidRuntime(2065): 	at com.example.hellobigworld.a.a(Unknown Source)
11-11 09:36:42.262: E/AndroidRuntime(2065): 	at com.example.hellobigworld.MainActivity.onClick(Unknown Source)
11-11 09:36:42.262: E/AndroidRuntime(2065): 	... 14 more
```

变成这样的原因是代码被混淆了。ADT的某个版本之后Android自动生成的工程(Eclipse工程)在导出APK的时候都会
混淆代码，混淆之后会发现工程根目录下面多了一个文件夹 `proguard` 里面包含四个文件：

1. dump.txt 存储class文件的内部结构
2. mapping.txt 源码到混淆之后代码的映射信息
3. seeds.txt 被各种 -keep 选项保留的类和成员变量信息
4. usage.txt 保存 unused or dead code 的信息

如果需要恢复原来的代码，需要用Proguard提供的retrace工具

```
D:\Tools\proguard4.10\bin>retrace mapping.txt log.txt
Caused by: java.lang.ArithmeticException: divide by zero

        at com.example.hellobigworld.ErrFactory.makeException(Unknown Source)

        at com.example.hellobigworld.ErrFactory.crash(Unknown Source)

        at com.example.hellobigworld.MainActivity.onClick(Unknown Source)

        ... 14 more
```

这样就可以恢复原来的代码了，但是还有一个情况 `(Unknown Source)` 是什么问题，好多开发者给友盟发support邮件，说错误统计
有问题，难道的错误log无法使用，误以为是友盟统计分析SDK的问题。实际上这个也是由于Proguard造成的，Proguard在混淆代码的
时候会做一些优化，比如行号、源文件这些信息其实VM是不需要的所以优化的时候会删除这些信息，导致SDK拿到的堆栈是没有这些信息的。

解决这个问题需要在 Proguard的配置文件中添加 

```
-keepattributes SourceFile,LineNumberTable
```

这样再次崩溃的时候就有源文件和行号的信息了

```
11-11 13:42:03.926: E/AndroidRuntime(4189): Caused by: java.lang.ArithmeticException: divide by zero
11-11 13:42:03.926: E/AndroidRuntime(4189): 	at com.example.hellobigworld.a.b(ErrFactory.java:13)
11-11 13:42:03.926: E/AndroidRuntime(4189): 	at com.example.hellobigworld.a.a(ErrFactory.java:6)
11-11 13:42:03.926: E/AndroidRuntime(4189): 	at com.example.hellobigworld.MainActivity.onClick(MainActivity.java:54)
11-11 13:42:03.926: E/AndroidRuntime(4189): 	... 14 more
```

就是这样，添加这个配置即使类和变量名丢了，行号和源文件还在。



