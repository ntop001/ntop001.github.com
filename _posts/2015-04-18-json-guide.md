---
layout: post
title: "Gson 文档"
description: ""
category: 
tags: [Gson]
---

打算写一个可以把JSONObject直接转成Java对象的库，其实就是类似Gson提供的功能，但是Gson并不能满足需求，Gson 会把字符串先转成 `com.google.gson.JsonObject `
对象,然后再映射到自定义的Java类，我希望的库，可以把标准的 `org.json.JSONObject` 映射到Java类，或者可以通过一些接口支持更多的JSON java实现。
而最快的方式就是直接修改Gson的源码。所以先从文档下手，学习一下Gson的功能，以便更好的理解源码设计。


