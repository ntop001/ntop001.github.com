---
layout: post
title: "Android’s HTTP Clients"
description: ""
category: "知识整理"
tags: [Android,Http]
---

这篇文档翻译自 [Android’s HTTP Clients](http://android-developers.blogspot.com/2011/09/androids-http-clients.html)

Android上的大部分网络请求都是HTTP请求，Android上包含两个HTTP实现，分别是HttpURLConnection和Apache HTTP Client。他们都支持HTTPS，流式的上传和下载接口，可以配置超时时间，支持IPV6和连接池。

## Apache HTTP Client

[DefaultHttpClient](http://developer.android.com/reference/org/apache/http/impl/client/DefaultHttpClient.html) 和 [ AndroidHttpClient](http://developer.android.com/reference/android/net/http/AndroidHttpClient.html)都是可以用于浏览器的扩展性比较好的实现。它们有很多灵活的API接口，比较稳定但是存在一些bug。

由于它的接口众多，修复这些bug同时保证兼容它的大量API很困难，所以Android小组打算抛弃这个实现了。

## HttpURLConnection

[HttpURLConnection](http://developer.android.com/reference/java/net/HttpURLConnection.html)对于大部分App来说是一个轻量级的，通用实现。这个类的实现比较简陋，但是它的API比较专，所以改进起来比较简单。

在冻酸奶(Froyo API8 2.2.x)之前，HttpURLConnection有几个bug，在一个没有读完的输入流(InputStream)上调用`close()`方法会[污染连接池](http://code.google.com/p/android/issues/detail?id=2939)。一个解决办法是关闭连接池：

```
private void disableConnectionReuseIfNecessary() {
    // HTTP connection reuse which was buggy pre-froyo
    if (Integer.parseInt(Build.VERSION.SDK) < Build.VERSION_CODES.FROYO) {
        System.setProperty("http.keepAlive", "false");
    }
}
```

在姜饼(Gingerbread API9 2.3)上，对HttpURLConnection请求返回默认添加了压缩，它会自动在请求上添加`Accept-Encoding: gzip`并且处理返回的数据。这样只要服务器也支持压缩，那么就可以很好的利用这一功能了。如果需要解除这个功能可以参考[文档](http://developer.android.com/reference/java/net/HttpURLConnection.html)。

需要注意的是HTTP的Content-Length字段返回的是压缩后的长度，直接调用`getContentLength() `方法来获取原始数据的长度是不对的，应该使用`InputStream.read()`方法读取指导返回-1为止。

在姜饼版本中对HTTPS的支持也有所改进。[HttpsURLConnection](http://developer.android.com/reference/javax/net/ssl/HttpsURLConnection.html)支持[Server Name Indication](http://en.wikipedia.org/wiki/Server_Name_Indication)技术，可以使多个服务器共享同一个IP地址。同时支持压缩和session tickets(因为HTTPS需要额外的握手来建立安全对话，在网络不好的情况这种技术可以复用上次的对话，避免再次握手)技术。如果连接失败的情况下，它会关闭上面的功能然后再次重试，这样就可以兼容旧的HTTPS服务器同时也可以支持最新的服务器特性。

在冰淇淋三明治版本（Ice Cream Sandwich API14 4.0.x）版本中，添加了对响应缓存的支持。启用缓存功能后，HTTP请求将满足下面三种请求方式之一：

1. 完全缓存的响应内容将直接从本地存储中读取。因为没有网络请求的缘故，这样的响应会变得非常快。
2. 条件缓存的响应内容必须让服务器来决定是否刷新缓存。客户端会发送一个请求给服务器"Give me /foo.png if it changed since yesterday" 如果请求内容变化的话，服务器可以返回新的内容，如果服务器内容没有变化的话就返回"304 Not Modified"。如果内从没有变化，不会导致下载。
3. 没有缓存的请求直接从服务器更新。这样的请求的结构会被存储在本地作为缓存。

为了兼容旧的系统版本，可以使用反射的方式打开HTTP缓存

```
private void enableHttpResponseCache() {
    try {
        long httpCacheSize = 10 * 1024 * 1024; // 10 MiB
        File httpCacheDir = new File(getCacheDir(), "http");
        Class.forName("android.net.http.HttpResponseCache")
            .getMethod("install", File.class, long.class)
            .invoke(null, httpCacheDir, httpCacheSize);
    } catch (Exception httpResponseCacheNotAvailable) {
    }
}
```

同时必需在服务器响应头里面配置缓存策略。


## Which client is best?

在Android的早期版本闪电泡芙(Eclair API5/API6/API7 2.0.x/2.1)和冻酸奶推荐使用ApacheHTTPClient。对于姜饼和后来的版本，推荐使用HttpURLConnection，API简单而且比较小更适合Android。默认的压缩和请求缓存的支持可以减少网络请求，提高速度并省电。对于新的应用来说最好都使用HttpURLConnection,因为我们一直在改进它。

PS：到目前为止Android2.2及以下的系统占有率不对1%。














