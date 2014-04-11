#Proguard 混淆选项

如果你还没有看过，那真是可惜了，糊里糊涂走了那么久 ...

1. Input/Output Options
2. Shrinking Options
3. Optimization Options
4. Obfuscation Options
5. Preverification Options
6. General Options


## Input/Output Options 

控制输入和输出选项，

## Shrinking Options

控制是否删除没有必要的类，默认情况下会删除所有的.class 文件(这时候无法导出jar包，因为jar包是空的)，只有被各种 `-keep` 命令保留的类和相关依赖会保存下来。
可以使用 `-printusage [filename]` 来打印哪些类(dead code)被删除了, 还可以使用 `-whyareyoukeeping class_spec` 来查看指定类被保留的原因。Shrink 选项可以删除
类文件中没有使用到的方法或者成员，功能异常强大。

禁用：-dontshrink (默认开启) 

## Optimization Options

控制如何对 `.class` 文件进行优化。优化步骤中预定义了许多优化策略，比如：

```
class/marking/final
	Marks classes as final, whenever possible.
class/unboxing/enum
	Simplifies enum types to integer constants, whenever possible.
class/merging/vertical
	Merges classes vertically in the class hierarchy, whenever possible.
class/merging/horizontal
	Merges classes horizontally in the class hierarchy, whenever possible.
field/removal/writeonly
	Removes write-only fields.
field/marking/private
	Marks fields as private, whenever possible.
...
```
可以使用 `-optimizations` 进行细粒度的控制，如果没有十足的把握，最好不要修改这里面的配置。`-optimizationpasses n` (优化次数) n表示迭代的次数，一般情况下
~2 次的时候已经会有明显的效果，~10次的时候优化会停止。所以设置 `-optimizationpasses 100` 并不会优化这么多次，大约10次左右的时候优化就会停止,优化是非常耗
时的过程，Android中默认是 5（据说是因为他们的Team成员老师抱怨build的时间太长了）。 Proguard的优化策略都是很难理解的，如果想吃快餐的话，还是不要琢磨这些
[策略](http://proguard.sourceforge.net/index.html#manual/optimizations.html)了。


禁用:-dontoptimize (默认开启)

## Obfuscation Options

终于到了我们关注的地方了，大部分开发者使用Proguard的原因可能都是为了混淆代码，前面的功能只是一些附加的选项。混淆的主要功能是把可读的包名和类名和成员，方法等
替换成a,b,c..等没有任何语义的字符。

* `-printmapping [filename]` 打印映射文件(com.example.foo -> a.b.c 类似),映射文件会保存所有的替换信息，这个文件在代码 Retrace 的时候有用到。
* `-applymapping filename` 指定映射文件，规则是如果指定了映射文件，就使用指定的规则否则Proguard会生成新的名字，一般是用在增量式混淆的场景，但是有时候也可以用来指定混淆规则
* `-obfuscationdictionary` 混淆字典，用来指定可以使用的混淆后的字符，默认的就是 a,b,c ... 一般情况没有必要修改它
* `-overloadaggressively` 绘声绘色的名字，它的作用的是积极的采用重载策略来减少代码，比如原来函数名不同的方法混淆后可以获得相同的名字，但是由于返回值或者参数不同他们还可以正常的调用。需要注意的是，这里的有些方式在 java 语法上是不被允许的比如这样：

```
class foo {
	public int getFoo()
	public long getFoo()
}
```
但是在java虚拟机上可以这样使用。

* `-keeppackagenames [package_filter]` 不要混淆包名
* `-flattenpackagehierarchy [package_name]`  把所有被混淆的包名里面的.class 转移到新的包名下面
* `-repackageclasses` 把所有被混淆的类文件转移到新的包名下面，和上面的区别是上面针对的是所有包名被改变的情况，而此针对的是类文件被混淆的情况
* `-keepattributes` 保留一些属性不被混淆，典型的有：

> Exceptions, Signature, Deprecated, SourceFile, SourceDir, LineNumberTable, LocalVariableTable, LocalVariableTypeTable, Synthetic, EnclosingMethod, RuntimeVisibleAnnotations, > RuntimeInvisibleAnnotations, RuntimeVisibleParameterAnnotations, RuntimeInvisibleParameterAnnotations, and AnnotationDefault  

典型用法，[保留堆栈信息](http://proguard.sourceforge.net/manual/examples.html#stacktrace)：

```
-printmapping out.map

-renamesourcefileattribute SourceFile 		# We're keeping all source file attributes, but we're replacing their values by the string "SourceFile"
-keepattributes SourceFile,LineNumberTable  # We're keeping the line number tables of all methods.
```

 处理类库的时候keep Exceptions, InnerClasses, and Signature attributes, 典型用法见[这里](http://proguard.sourceforge.net/manual/examples.html#stacktrace)


* `-keepparameternames` 保留参数信息，一般情况下被混淆的类库方法参数会变成 foo(int args), 使用此可以保留原来的参数命名foo(int key), 可以方便IDE做代码提示
* `-renamesourcefileattribute [string]` 替换源文件属性，用法可见 [保留堆栈信息](http://proguard.sourceforge.net/manual/examples.html#stacktrace) 信息部分
* `-adaptclassstrings [class_filter]` 适配类字符串，混淆后的类名被改变后，写在代码中的引用这些类名的字符也同样被混淆，这在反射时有用
* `-adaptresourcefilenames [file_filter] ` 同上，不过是针对打在jar包中的资源文件
* `-adaptresourcefilecontents [file_filter]` 同上，针对资源文件中的字符串

对于一般开发者来说，混淆是最重要的部分，务必理解这里面的每一个配置选项

禁用：-dontobfuscate(默认开启)

## Preverification Options

早期版本的jvm标准中对于class文件是运行时验证的，在java>= 6 之后为了加快运行时速度，改为编译时验证。所谓验证时对.class 文件的安全和合法性做一些检查。
可以通过 `-dontpreverify` 禁用验证。这样可以减少proguard处理时间。对于j2me 版本，需要加上 `-microedition` 处理上略有差异。

## General Options

Shrink,Optimization,Obfuse 是Proguard最重要的主题，General Options 部分是针对Proguard的一些通用配置。

* `-verbose` 所有Console工具都有的选项，打印详细Log信息
* `-dontnote [class_filter]` 不打印一些潜在错误或遗漏的配置信息
* `-dontwarn [class_filter]` 不再提示警告信息比如：无法解析的类或者其他的问题。> Only use this option if you know what you're doing!
* `-ignorewarnings`  同上，不过更友好一些，虽然忽略警告，但是依然会打印出这些警告
* `-dump [filename]` 如果你没有好用的反编译工具可以用这个选项查看jar内容和class 结构,比如这样的一个脚本：

```
-injars in.jar

-dontshrink
-dontoptimize
-dontobfuscate
-dontpreverify

-dump
```

至此，Proguard 中比较基础的配置已经介绍完毕，更多，更细节的内容还是看文档吧

*[Usage](http://proguard.sourceforge.net/index.html#manual/usage.html) 包含所以可配置属性的说明
*[Example](http://proguard.sourceforge.net/index.html#manual/examples.html) 示例脚本，一般性的问题都可以在这里找到现成可用的代码 