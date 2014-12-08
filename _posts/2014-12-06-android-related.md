---
layout: post
title: "一次Android项目知识整理"
description: "知识整理"
category: 
tags: [Android]
---
{% include JB/setup %}

其实也算是老Androider了，但是在友盟的很长时间都是在写SDK而且是统计分析的SDK（木有UI），UI部分的功力日渐下降，现在整理一下最近做的一个项目用到的知识点。

## 网络访问

网络访问是比较耗时的任务，一般都是开线程来做的，以前都是用下面这些方式

1. 使用 AsyncTask 
2. 开线程配合 Handler

现在使用的是Google提供的一个库 [Volley](http://developer.android.com/training/volley/index.html) 来做的，Volley 封装了网络请求，
把每个请求放到内部实现的一个请求队列里面，底层使用线程池来异步操作，同时还封装了NetworkImageView控件，自带ImageLoader。目前看来以后可能是App标配组件了。

## 数据解析

数据解析部分使用的是 [Gson](https://code.google.com/p/google-gson/). Gson的好处是可以把JSON数据直接映射到定义好的类的各个字段，
方便数据的解析和序列化。这样JSON只作为传输协议使用而不用来在内部保存数据，一般写程序的时候最好不要直接用JSON来操作内存数据而用类来表示，JSON是可以认为是k-v型的集合数据，使用起来不方面，另外JSON内存占用非常大，基本是原始数据的3~5倍。

## UI

#### Tab效果实现

现在主流的Tab效果实现方式都是用Fragment配合TabHost实现的，早前的用Activity+TabHost的方式已经过时了。

#### ListView

ListView可以通过实现 `getViewTypeCount` 和 `getItemViewType` 来实现复用不同类型的Item。

其他开源的库

1. [UniversalImageLoader](https://github.com/nostra13/Android-Universal-Image-Loader) 和Volley中的ImageLoader 作用一样用来加载图片，并且实现了缓存控制的功能。但是UIL可以自带了图片圆角的功能，貌似挺好用的。
2. [PullToRefresh](https://github.com/chrisbanes/Android-PullToRefresh) 年久失修的下拉刷新组件
3. [DevSmartLib](https://github.com/dinocore1/DevsmartLib-Android) 一个横向的ListView（Gallery控件已经Deprecated了）
4. [RoundedImageView](https://github.com/vinc3m1/RoundedImageView) 作者说这是最快的 RoundedImageView
5. [CircleImageView](https://github.com/hdodenhof/CircleImageView) 基于上面RoundedImageView实现的


其他第三方服务：

1. 友盟的统计分析、社交分享和push全用了
2. 高德地图，使用高德地图是因为iOS上默认地图也是高德地图


