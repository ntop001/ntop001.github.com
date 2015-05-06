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

启用用户模块只需要做很少的配置，概括一下主要是下面4点

1. 数据模型
2. 路由配置
3. 请求处理
4. 模板渲染

我们可以通过一个配置一个路由比如:http://xxx/login/ 来把用户导向登陆页面，然后再处理请求，这个过程中需要做的事情

1. 提供一个用户登录的界面
2. 在请求中验证用户登陆（或者注册）
3. 登陆成功后重定向到一个合适的界面
4. 对于其他页面如果用户没有登陆都无法访问并重定向到登陆页面


#### 数据模型

`django.contrib.auth.models` 包中提供了 `User` 对象，代表一个用户数据模型，它有几个主要的属性

* username
* password
* email
* first_name
* last_name

这个类具有ORM功能的所以只有创建一个对象并调用它的`save()`方法就可以在数据库中添加一条用户的记录。

下面直接通过在数据库中插入一个用户（正常情况应该提供一个注册页面来获取注册信息，然后把注册信息保存在数据库中）

```
>>> from django.contrib.auth.models import User
>>> user = User.objects.create_user('john', 'lennon@thebeatles.com', 'johnpassword')

# At this point, user is a User object that has already been saved
# to the database. You can continue to change its attributes
# if you want to change other fields.
>>> user.last_name = 'Lennon'
>>> user.save()
```

这样数据库中就有了一条数据记录。

## 路由配置

现在配置的路由是这样的

```
url(r'^login/$', views.login, name= "login"),
url(r'^logout/$', views.logout, name= "logout")
```

这里通过访问 `http://xxx/login/` 和 `http://xxx/logout/` 就可以触发在views中定义好的处理方法。


## 请求处理

在views.py中提供了两个方法来做请求处理

```
from django.contrib.auth import views as auth_views

def login(request):
	login_tpl = auth_views.login(request)
	return login_tpl
	
def logout(request):
	logout_tpl = auth_views.logout_then_login(request)
	return logout_tpl
```

在路由配置中的路由会分别触发这两个方法，第一个是登陆方法，第二个是退出，但是这里面并没有写自己的逻辑，而是直接调用了 Django 提供的登陆处理功能 `auth_views.login(request)` 如果是GET请求，这个方法会返回登陆表单页面，如果是POST请求会处理登陆请求验证登陆信息。

但是这个方法依赖一个模板，需要我们自己提供，这就是模板渲染。

## 模板渲染

默认情况下Django会寻找 `templates/registration/` 目录下的 `login.html` 文件，这个文件大概这么写就行了（约定，细节看[*这里*](https://docs.djangoproject.com/en/1.8/topics/auth/default/#all-authentication-views)）

```
{% block content %}

  {% if form.errors %}
    <p class="error">Sorry, that's not a valid username or password</p>
  {% endif %}

  <form action="" method="post">
    {% csrf_token %}
    <label for="username">User name:</label>
    <input type="text" name="username" value="" id="username">
    <label for="password">Password:</label>
    <input type="password" name="password" value="" id="password">

    <input type="submit" value="login" />
    <input type="hidden" name="next" value="/op/" />
  </form>

{% endblock %}
```

配置完这4项之后在浏览器中输入 `http://xxx/login/` 就会弹出登陆页面

    <form action="" method="post">
        <input type="hidden" name="csrfmiddlewaretoken" value="7hLMThuc3l8ek8ECNrcRntJBRSCLjKmc">
        <label for="username">User name:</label>
        <input type="text" name="username" value="" id="username">
        <label for="password">Password:</label>
        <input type="password" name="password" value="" id="password">
        <input type="submit" value="login">
        <input type="hidden" name="next" value="">
    </form>


如果输入用户名`john` 密码 `johnpassword` (之前已经在数据库中插入这个用户) 就会发现页面跳转到404页面，那是因为我们还没有配置登陆完成之后的要跳转到那个页面，这时候可以看下表单中的 `<input type="hidden" name="next" value="">` 这里可以配置登陆成功后的跳转页面比如配置 `<input type="hidden" name="next" value="/next/">` 将会跳转到 `http://xxxx/next/`。

登出的话，也差不多，但是上面的实现中是调用了 `auth_views.logout_then_login(request, login_url= '/login/')` 方法，这个方法会在登出之后重定向到登陆界面，可以通过配置 `login_url= '/login/'`来设置登陆界面的地址。

还有一个问题 "4. 对于其他页面如果用户没有登陆都无法访问并重定向到登陆页面" 没有解决,对于其他的界面，还需要在处理请求的时候需要先看看用户是否已经登陆，一般的做法是在每个请求方法中写一段代码来做，但是在Django提供了一种非常简单的处理方式: 添加 `@login_required` 注释

```
@login_required
def index(request):
	return HttpResponse('Hello world')
```

只要给每个方法前面添加这个注释，那么当请求路由这个方法时都会先检查用户是否登陆，如果没有登陆就会重定向到登陆页面，默认的登陆地址是 `http://xxx/accounts/login/?next=xxx` 但是我们之前登陆地址是 `http://xxxx/login/` 所以还需要告诉 `@login_required` 登陆地址在哪里，需要设置 

```
@login_required(login_url='/login/')
def index(request):
	return HttpResponse('Hello world')
```

## End

还有很多中方式来利用Django的用户登陆模块的功能，详细还是看文档吧

https://docs.djangoproject.com/en/1.8/topics/auth/default/

