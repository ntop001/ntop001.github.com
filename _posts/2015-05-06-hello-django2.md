---
layout: post
title: "Hello Django 2"
description: "model/view/controller"
category: 
tags: [Django]
---


## 模板

在App的目录下，可以新建一个文件夹 `templates` 可以用来存放App使用的模板文件，Django 默认会在这文件夹下寻找模板，同样，Django也会默认在 `static` 文件夹下查找使用的css，js等文件。

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
