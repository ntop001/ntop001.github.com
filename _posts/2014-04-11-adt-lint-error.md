---
layout: post
title: "Eclipse 导出 APK时候的lint错误"
description: ""
category: 
tags: []
---
{% include JB/setup %}

有时候使用Eclipse导出Apk的时候会一些 lint 错误，比如某个字符串在某个语言中没有翻译，这些错误都是ADT的lint工具做一些验证的时候报的错误。

如果想简单忽略这些错误，可以在Eclipse ` "Window" > "Preferences" > "Android" > "Lint Error Checking":` 忽略相关的配置。
