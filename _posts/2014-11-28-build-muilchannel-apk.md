---
layout: post
title: "为友盟统计分析自动化打多渠道包"
description: ""
category: 
tags: [SDK]
---

在大部分统计分析服务中为了追踪App的渠道来源都需要在App中设下标示来标记App的分发来源。友盟的统计分析SDK的做法是在 AndroidManifest.xml
文件中配置一项`Meta-data` 来标记渠道的，如

```
        <meta-data
            android:name="UMENG_CHANNEL"
            android:value="Umeng" >
        </meta-data>
```

对于开发者而言，不同的渠道只要修改这个渠道标识就可以了，但是对于国内的Android市场，各种市场、广告和其他分发平台不下百家，如果每次
重新修改渠道标识再打包是一个巨大且无味的工程，所以各种自动化打包的方案就出来了。

## 基于Android工程中自带build脚本的打包方式

大家在开发App的时候如果是在Eclipse环境下，基本上是通过 `Android Tools -> Exports (Un)Signed Application Package ...` 来打包的，
但是实际上还可以通过配置ant脚本来打包。执行 `android update project –p XXXX –t XXXX` 就会发现工程的根目录下多了几个文件，其中
最重要的就是`build.xml` 这个文件中配置了输出apk的ant脚本。

* ant clean 清空bin和 obj目录
* ant release 打包生成未签名的 apk（你可以在build.xml中配置签名这样就可以直接输出签名的apk了）

(如果不知道ant是什么看下[这里](http://ant.apache.org/)) 现在有了可以
通过简单的ant命令就可以打包的方法了，那么下面只要写几行代码在每次打包之前替换渠道就可以了。

替换渠道是很简单的事，因为AndroidManifest.xml文件只是一个简单的文本文件，你可以写正则或者用xml解析的方式更新友盟渠道的值。
使用这种方式打包的缺点是需要使用 ant （不熟悉的同学可能觉得不方面），因为 ant 的许多配置本身也很麻烦。

## 用 Gradle 打包

Gradle的Android插件是Google发布 AndroidStudio的时候提供的，同Ant比起来，Gradle好用多了，可以看一下之前的[介绍](http://ntop001.github.io/%E7%9F%A5%E8%AF%86%E6%95%B4%E7%90%86/2014/11/07/gradle/)
因为Gradle的项目描述是使用 Groovy 语言做的，所以能够直接在打包脚本中写一些代码来完成渠道的替换.

利用 Android Gradle 的 ProductFlavor 功能添加多个渠道,每个渠道都是一个Flavors

```
//渠道
    productFlavors {
        wandoujia{
            //这里可以写一些更详细的配置，如果不需要的话，可以忽略
        }
        appchina{
        }
    }
```

在处理 AndroidManifest.xml 文件时添加一个钩子(hook)替换渠道,注意，这里面这是做了一个简单的字符串替换，需要在原来的
AndroidManifest.xml 文件中添加：<meta-data android:value="UMENG_CHANNEL_VALUE" android:name="UMENG_CHANNEL"/>
这样脚本会替换 UMENG_CHANNEL_VALUE 为当前 ProductFlavor 的名称。

```
android.applicationVariants.all{ variant -> 
    println "${variant.productFlavors[0].name}"
    variant.processManifest.doLast{
        copy{
            from("${buildDir}/manifests"){
                include "${variant.dirName}/AndroidManifest.xml"
            }
            into("${buildDir}/manifests/$variant.name")

            filter{
                String line -> line.replaceAll("UMENG_CHANNEL_VALUE", "${variant.productFlavors[0].name}")
            }

            variant.processResources.manifestFile = file("${buildDir}/manifests/${variant.name}/${variant.dirName}/AndroidManifest.xml")
        }    
   }
}
```

PS：目前看来这是最靠谱的打包方式，同Ant方式比配置简单，自定义能力强，同后面提到的反编译的方式比较，虽然速度上不及（每次重新打包资源）
但是稳定性可以保证，通过反编译的方式会对原有的apk进行修改，这些修改可能造成意外的问题。所以建议大家以后都通过这种方式打包。

## 通过反编译的方式打包

这种打包方式不需要源码，只要一个apk就可以了，操作起来比较简单，但是由于修改了原来apk的缘故，可能会出现一些莫名其妙的错误。
一种比较常见的做法是使用 apktool 这个工具，他有两个主要命令

* apktool d 123.apk xxx 将apk反编译到 xxx 文件夹
* apktool b xxx 123_out.apk  将 xxx 文件夹中内容重新打包成 123_out.apk

使用apktool打包的方式是反编译之后可以得到纯文本格式的AndroidManifest.xml文件，编辑这个文件修改其中的渠道然后再重新打包成apk。

但是apktool工具是一个破解工具稳定性无法保证，而且至今为止已经发现了很多问题，比如打包在 jar中的资源重新打包的时候会丢失等等，
但是对于apk而言只是要修改AndroidManifest.xml文件就够了。所以后来研究了一遍AndroidManifest.xml(AXML)文件二进制格式的结构之后我写了一
个专门用来解析AXML文件的[工具](https://github.com/ntop001/AXMLEditor). 并提供非常简单的编辑功能，可以用来替换AXML中的渠道标识。


## 其他打包方式

通过Eclipse插件打包，Eclipse 的插件功能非常好，包括ADT也只是Eclipse的一个插件而已，于是有人专门写了这样的一个插件,可以看看
[这里](https://github.com/ntop001/umeng-muti-channel-build-tool/tree/master/EclipsePlugin).

还有一种开发者经常提到的方式，用 aapt 命令往 apk 的assert目录插入一个文件作为渠道标示，然后在代码中读取这个渠道通过代码设置渠道。






