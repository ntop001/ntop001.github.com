---
layout: post
title: "在第三方jar包中查找资源"
description: ""
category: "知识整理"
tags: [SDK, Android]
---

在提供SDK给开发者的时候有时候不仅仅需要一个jar包，还需要把相关的图片、xml资源一起打包，对于开发者来说集成第三方SDK变成两步

1. 集成jar包
2. 把资源文件考入自己的工程

对SDK的提供者来说，在开发jar的时候在jar包中引用资源一般这样做  `R.id.xxxx` 就可以引用到对应的资源，但是如果这样的jar包提供给开发者，开发者在使用的时候是会找不到这个资源，原因是这样造成的，对于SDK提供者
来说他实际上是这样引用资源（以友盟统计分析SDK为例）的 `com.umeng.analytics.R.id.xxx`, R.java 其实是当前工程下的一个类，但是当开发者把资源考到自己工程中的时候(假设开发者的工程包名是`com.foo.bar`) 那么这些资源文件重新生成的R文件是 `com.foo.bar.R.id.xxx` 所以虽然都是 `R.id.xxx` 但是包名变了，所以这时候Jar包里面会报错，找不到
对应的资源，解决这个问题一个直观的想法是用反射的方式查找对应资源。

## 用反射查找资源

先看一下R.java 文件的布局

```
public final class R {
    public static final class anim {
        public static final int umeng_fb_slide_in_from_left=0x7f040000;
    }
    public static final class attr {
    }
    public static final class drawable {
        public static final int author_avatar=0x7f020000;
        public static final int default_bg=0x7f02001b;
        public static final int home_normal=0x7f020001;
        public static final int ic_launcher=0x7f020002;
    }
    public static final class id {
        public static final int action_settings=0x7f09001b;
        public static final int android_id=0x7f090009;
        public static final int button1=0x7f09000b;
    }
    public static final class layout {
        public static final int main=0x7f030001;
    }
    public static final class menu {
        public static final int main=0x7f080000;
    }
    public static final class string {
        public static final int action_settings=0x7f060001;
        public static final int app_name=0x7f060000;
        public static final int hello_world=0x7f060002;
        public static final int version_name=0x7f060004;
    }
}
```
大概是这样，每种类型的资源都是R类的一个静态子类，资源都定义成静态字段,需要反射查找资源。

```
 int getId(String pkg, String type, String name){
		try{
			Class<?> clz = Class.forName(pkg + ".R$" + type);
			return clz.getField(name).getInt(null);
		}catch(Exception igore){}
		
		return -1;
	}
```

在SDK的代码中每次查找资源id的时候不能再使用 `R.id.xxx` 的方式而是使用 `getId("com.foo.bar", "id", "xxx")` 对于图片类型就是 
`getId("com.foo.bar", "drawable", "xxx")` 第一个参数包名可以由开发者传入或者通过 `context.getPackageName()` 获取，当然 `context` 也需要开发者传入。但是如果通过 `context.getPackageName()` 获取包名可能会有一个很大的问题，因为在Android中，可以在
打包的时候修改 AndroidManifest.xml 中的包名，通过`context.getPackageName()`方式获取的包名和R文件的实际包名并不一致，会导致
错误。那么就需要看下面这种方式了。


## 使用 Resources 类查找资源

其实Android中已经提供了通过资源文件名索引资源的方法，`android.content.res.Resources` 类提供了一系列查找资源的方法 其中

> public int getIdentifier (String name, String defType, String defPackage)

> Added in API level 1
> Return a resource identifier for the given resource name. A fully qualified resource name is of the form "package:type/entry". The first two components (package and type) are optional if defType and defPackage, respectively, are specified here.

> **Note:** use of this function is discouraged. It is much more efficient to retrieve resources by identifier than by name.

> **Parameters**
> name	The name of the desired resource.
> defType	Optional default resource type to find, if "type/" is not included in the name. Can be null to require an explicit > type.
> defPackage	Optional default package to find, if "package:" is not included in the name. Can be null to require an explicit > package.
> **Returns**
> int The associated resource identifier. Returns 0 if no such resource was found. (0 is not a valid resource ID.)

这个方法，可以通过资源名，资源类型，包名索引资源id，但是它的实现并不是上面那种反射实现的方式，而是直接解析了底层的资源
所以即使在开发者打包的时候修改了包名，也能正确的索引资源。


