---
layout: post
title: "标示唯一设备"
description: "翻译：http://android-developers.blogspot.com/2011/03/identifying-app-installations.html"
category: 
tags: [SDK]
---


这是一篇翻译的文章，来自[Android Blog](http://android-developers.blogspot.com/2011/03/identifying-app-installations.html)
大概讲述了追踪唯一设备的那些事情。

在Android开发者中我们听到了很多抱怨，关于如何获取可依赖的、稳定的设备唯一标示。这让我们很蛋疼，以为追踪这样的设备标示
并不是一个很好的想法，并且有更好的做法。

## 追踪App安装量

对于开发者来说追踪App的安装量是很常见也可以理解的事情。但是呢，仅仅通过调用 ` TelephonyManager.getDeviceId() ` （获取IMEI）
这个方法来获取设备标示看上去有点不可信。

1. 这个标示并不可依赖
2. 这个标示在设备恢复出厂设置（Factory resets）的时候也不会改变，如果他把手机重置送人了，那么就是另外一个人了

PS：这里体现一个问题，目前国内对App安装的标示是针对设备的，但是Google的想法是针对人的，所以才存在第二个问题

如果要追踪App的安装，你可以自己生成一个UUID，只要在App 第一次启动的时候创建一个就行了。下面是一个简单的示例，你可以通过
一个静态的方法 `Installation.id(Context context)` 来获取Id值。

```
public class Installation {
    private static String sID = null;
    private static final String INSTALLATION = "INSTALLATION";

    public synchronized static String id(Context context) {
        if (sID == null) {  
            File installation = new File(context.getFilesDir(), INSTALLATION);
            try {
                if (!installation.exists())
                    writeInstallationFile(installation);
                sID = readInstallationFile(installation);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
        return sID;
    }

    private static String readInstallationFile(File installation) throws IOException {
        RandomAccessFile f = new RandomAccessFile(installation, "r");
        byte[] bytes = new byte[(int) f.length()];
        f.readFully(bytes);
        f.close();
        return new String(bytes);
    }

    private static void writeInstallationFile(File installation) throws IOException {
        FileOutputStream out = new FileOutputStream(installation);
        String id = UUID.randomUUID().toString();
        out.write(id.getBytes());
        out.close();
    }
}
```

## 标识设备

但是，如果你需要的是一个真实的硬件设备标识，这将会一个棘手的问题。在过去每一个Android设备还是一部手机的时候，可以通过
`TelephonyManager.getDeviceId()` 这个方法获取 `IMEI, MEID, or ESN ` 这些信息，这些标志是硬件唯一的。

但是使用这类设备id存在这些问题

* 非手机的Android设备： 一些仅支持wifi的设备或者音乐播放器没有通信模块，所以也不存在这个标识
* 持久性：这个标识即使设备清空数据或者恢复出厂设置的时候也不会改变，那么App可能会始终把这台手机当做同一个设备
* 权限：需要 `READ_PHONE_STATE` 这个权限才能读取，如果App本身没有使用电话相关的功能那么这个权限会让人觉得不爽
* Bugs：有些手机这个标识的实现有问题，比如返回“0”或者“*”等

## Mac 地址

从设备的Wifi或者蓝牙模块获取mac地址也是一种选择，但是这并不被推荐，因为不是所有的设备都有wifi，同时如果wifi没有打开，
硬件或许不会上报mac地址。

## 序列号

从Android 2.3（“Gingerbread”）开始，可以通过 `android.os.Build.SERIAL` 这个变量获取序列号。在没有电话模块的设备上要求
通过这个变量上报一个设备Id， 有些有手机可能也会这么做。

## Android ID

更具体的说是这个 `Settings.Secure.ANDROID_ID` , 这是一个 64-bit 的值会在设备第一次启动的时候生成，当设备被重置（清空数据）
的时候会重置。

Android看上去是一个很好用的标识，但是也有缺点：

1. 在Android 2.2 之前的设备上并不可靠
2. 有很多的设备厂商在实现功能的时候存有bug，导致大量的AndroidId重复

## 结论

对于大部分App而言，目的只是去标识一次安装而不是标识一个物理设备。但是这么做貌似很简单直接。

有好多理由应该避免获取设备的标识，但是如果执意这么做，最好在比较新的设备上用 AndroidId，对于老设备再稍微
考虑一下兼容方案。
（For those who want to try, the best approach is probably the use of ANDROID_ID on anything reasonably modern, with some fallback heuristics for legacy devices.
原话好难译）


