---
layout: post
title: "迁移Eclipse工程到AndroidStudio"
description: ""
category: 
tags: []
---

最近在接手一些旧的Eclipse工程，打算迁移到Android Studio，照目前的形势来看，Google一直努力的开发环境是Android Studio+Gradle
的方式，以后可能会彻底放弃Eclipse+ADT，所以没有必要继续在Eclipse上工作。

迁移其实简单，在 Android Studio 里面 “New” -> "Import Project" 选择Eclipse工程，就可以把Eclipse工程导入 Android Studio 了。
但是这样做，有个小问题，原来的Eclipse工程结构是这样的

```
- Helloworld
  - src
  - res
  - AndroidManifest.xml
```

导入 Android Studio 之后变成 Android Studio 风格的工程结构了

```
- Helloworld
  - src
    - main
      - java
      - res
      - AndroidManifest.xml
  build.gradle
```

这样的结构，对于版本管理工具来说会产生很大的变化（比如git会产生很多diff），同时不兼容旧的Eclipse工程结构。

一个简单的办法是，不用 Android Studio 导入，而是在原来的Eclipse工程根目录下面创建一个build.gradle文件，在这个文件中配置
工程结构。

```
apply plugin: 'com.android.application'

android {
    compileSdkVersion 22
    buildToolsVersion '22.0.1'
    defaultConfig {
        //配置 applicationId
        applicationId 'com.github.mobile'
        minSdkVersion 15
        targetSdkVersion 22
        versionCode 1900
        versionName '1.9.0'
    }

    lintOptions {
        warning 'MissingTranslation'
        abortOnError false
    }

    sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            res.srcDirs = ['res']
            aidl.srcDirs = ['aidl']      
            assets.srcDirs = ['assets']
        }
    }
}

```

相对于一般的 Android Studio 工程，它只多了 sourceSets 的配置, sourceSets 中指定了工程结构。

```
sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            res.srcDirs = ['res']
            aidl.srcDirs = ['aidl']      
            assets.srcDirs = ['assets']
        }
    }
```

对于Android Studio来说，它最关注的其实是 build.gradle 文件。有了这个文件，他就可以识别工程结构，正常编译了。在Eclipse
工程下面添加了 build.gradle  文件之后，再次导入 Android Studio，会发现可以成功导入，而且工程结构也没有变化。
