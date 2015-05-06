---
layout: post
title: "Hello Django 3"
description: "用户系统"
category: 
tags: [Django]
---

现在大部分的网站都会有用户系统，比如需要个注册、登陆之类的功能，有了用户系统就可以对不同的用户设置权限，从而限制对网站的访问。
在Django中有一套默认的工具提供了用户系统的支持。

官网的文档[**在此**](https://docs.djangoproject.com/en/1.8/topics/auth/)

下面做一些简单的笔记。

## 配置

[原文](https://docs.djangoproject.com/en/1.8/topics/auth/)

用户模块依赖Django的两个功能，Session 和 Authentication ，这两个功能在Django中是通过中间件(Middleware)的形式提供的，在默认情况下，这两个功能都是开启的，如果没有开启的话可以查看setting.py中是不是如下配置

```
INSTALLED_APPS = (
    ...
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    ...
)

MIDDLEWARE_CLASSES = (
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.auth.middleware.SessionAuthenticationMiddleware',
    ...
)
```

如果已经配置好了，需要执行一次 `manage.py migrate` 这样会创建数据库和相关存储用户信息的表，否则用户数据无法存储。

## 使用

启用用户模块只需要做很少的配置，概括一下主要是下面X点

1. 数据模型
2. 路由配置
3. 





