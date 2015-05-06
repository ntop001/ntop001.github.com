---
layout: post
title: "Hello Django 2"
description: "model/view/controller"
category: 
tags: [Django]
---

Django的工程架构非常清晰，分为MVC三个模块，一个典型的**Django**工程如下

```
//app
polls/
    __init__.py
    templates/
        **.html
    models.py
    views.py
    urls.py
```

1. `models.py` 定义各种数据模型的类
2. `views.py` 实际上是一个 Controller 用来处理HTTP请求和一些自己的逻辑
3. `templates/` 文件夹中定义各种View的模板
4. `urls.py` 定义每个url到Controller中处理函数的映射（路由）

## Model

Django提供了[**ORM**](http://en.wikipedia.org/wiki/Object-relational_mapping)支持，只要定义好了数据模型，继承自 `models.Model` ,那么这个类变具有了ORM能力。这样就不需要再手动的操作数据库，把数据中的记录解析成数据模型。

```
from django.db import models


class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')
```

这个例子中定义了类 `Question` 包含两个属性 `question_text` 和 `pub_date`。定义好模型之后可以调用 `python manage.py migrate` 命令，这样Django就会根据定义好的模型，创建合适的数据库和表。通过ORM对象操作数据库非常简单，比如 `Question.objects.all()` 就可以读出数据库中所有的数据单元,下面的简单语法就可以把一条记录存到数据库。

```
>>> from django.utils import timezone
>>> q = Question(question_text="What's new?", pub_date=timezone.now())

# Save the object into the database. You have to call save() explicitly.
>>> q.save()
```

## Controller

在Django中views.py中定义的方法实际上是起到了Controller的作用，如果已经在urls.py中定义好了路由，那么每个请求都会被路由到views.py中定义的方法中。比如定义 `url(r'^$', views.index, name="index")` 可以把 `http://xxxx/` 路由到views中定义的index方法。这样如果在浏览器中输入`http://xxxx/`就可以出发到`views.index` 方法，这样就可以再这个方法中做适当的处理。

```
def index(request):
	return HttpResponse('hello world')
```

views中定义的请求处理方法，都会有一个输入参数 `request` 包含了请求信息，返回的内容都是 `HttpResponse` 实例包含了返回信息。比如上面的返回会最终在浏览器的网页中显示 `hello world` 字样。

但是一般情况下，需要返回给浏览器的是一个复杂的网页而不几个字符，这时候就需要返回一个复杂的HTML网页比如

```
<!DOCTYPE html>
<html>
<title>HTML Tutorial</title>
<body>

<h1>This is a heading</h1>
<p>This is a paragraph.</p>

</body>
</html>
```

更多的时候还需要把数据库中的数据展示在网页中，这是一个复杂的工程，在Django（大部分Web框架）中提供了模板功能，可以方便的把数据填充到网页中。


## 模板

模板在Django中属于View层，它定义了网页的展示效果，在App的目录下，可以新建一个文件夹 `templates`用来存放App使用的模板文件，Django 默认会在这文件夹下寻找模板，同样，Django也会默认在 `static` 文件夹下查找使用的css，js等文件。

所以上面的目录大约是这样

1. polls/templates/polls/index.html 使用： render(request, 'polls/index.html', {'latest_question_list': latest_question_list})
 
 ```
 {% if latest_question_list %}
    <ul>
    {% for question in latest_question_list %}
        <li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>
    {% endfor %}
    </ul>
 {% else %}
    <p>No polls are available.</p>
 {% endif %}
 ```
2. polls/static/polls/style.css 

 ```
 li a {
    color: green;
 }
 
 //use: polls/templates/polls/index.html
 {% load staticfiles %}
 <link rel="stylesheet" type="text/css" href="{% static 'polls/style.css' %}" />
 ```

## 表单
