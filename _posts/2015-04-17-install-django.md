---
layout: post
title: "Hello Django"
description: ""
category: 
tags: []
---

记录安装Django的过程和遇到的一些坑

## Python 2.x VS Python 3.x

语言自身的变迁会导致很多兼容问题，Python 2.x 到 3.x 同样如此，很多变化让人觉得3.x是新的语言了~~ 选择哪个版本是入门的一大难题。

因为我们过去的项目都是基于2.x开发的，所以直接选择2.x。关于如何选择，搜索这个主题可以获得很多信息。

## 安装Python

前往 https://www.python.org/downloads/ 下载安装，注意选择合适的版本开发。安装完之后需要把 `C:\Python27\Scripts` 和 `C:\Python27` 添加到环境变量，这样后续就可以在命令行中直接使用一些命令了(比如: `> python xxx` ), 后面安装Python模块不如PIP和
Django的命令会被添加到 `C:\Python27\Scripts` 里面。

## 安装 PIP

PIP是Python下的一个包管理工具，Django 需要用它来安装，所以需要先安装它。前往页面 `https://pip.pypa.io/en/latest/installing.html` 下载一个.py 文件执行就可以安装了。

## 安装 Django

如果没有把 `C:\Python27\Scripts` 添加到环境变量，需要执行 `python -m pip install Django==1.8` （这句话的意思是安装1.8版本的Django）来安装，但是如果已经添加了环境变量，直接 `pip install Django==1.8` 就行了。

安装过程中可能会报这个错误 `SSL InsecurePlatform error when using Requests package` 搜索一下StackOverflow里面找到[**解决方法**](http://stackoverflow.com/questions/29099404/ssl-insecureplatform-error-when-using-requests-package). 

> Use the somewhat hidden security feature:

>  pip install requests[security]

>  This installs following extra packages:
 
>  pyOpenSSL
>  ndg-httpsclient
>  pyasn1

>  Please note that this is not required for python-2.7.9+


验证Django安装是否成功，可以打印一下Django的版本

```
>>> import django
>>> print(django.get_version())
1.8
```

Tips:

-C 执行字符串中的命令

```
python -c "import django; print(django.get_version())"
```

使用Django新建一个工程 `>django-admin startproject mysite` 注意当安装了Django之后，会在 `C:\Python27\Scripts` 里面添加 `django-admin` 的命令。执行这个命令会新建一个工程名字叫 `mysite` 工程目录如下：

```
//project
mysite/
    manage.py
    mysite/
        __init__.py
        settings.py
        urls.py
        wsgi.py
```

新建一个APP `>python manage.py startapp polls` 在Django中每个App都是一个工程的模块，分别可以负责工程的不同功能。

```
//app
polls/
    __init__.py
    admin.py
    migrations/
        __init__.py
    models.py
    tests.py
    views.py
```


