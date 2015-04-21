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

##　使用

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
    
 //编辑Models
class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')


class Choice(models.Model):
    question = models.ForeignKey(Question)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)
```

Django里面有一个概念叫 `migration` 它记录了对于Model的变动（方便操作数据库），每次Model发生了变化都可以调用命令记录下来，比如上面 `python manage.py makemigrations polls` 这时候会生成一个文件记录了这次变动 `polls/migrations/0001_initial.py` 通过执行 `python manage.py migrate` 就可以把所有的变动对应的数据结构构造出来。这是一个非常牛逼的功能，这样就不需要每次变动Model的时候都重新建表建数据库。

## 其他常用的命令

1.python manage.py sqlmigrate polls 0001

这条命令会打印一些SQL语句，显示这次migrate会产生的SQL操作。

 ```
 BEGIN;
 CREATE TABLE "polls_choice" (
     "id" serial NOT NULL PRIMARY KEY,
     "choice_text" varchar(200) NOT NULL,
     "votes" integer NOT NULL
 );
 CREATE TABLE "polls_question" (
     "id" serial NOT NULL PRIMARY KEY,
     "question_text" varchar(200) NOT NULL,
     "pub_date" timestamp with time zone NOT NULL
 );
 ALTER TABLE "polls_choice" ADD COLUMN "question_id" integer NOT NULL;
 ALTER TABLE "polls_choice" ALTER COLUMN "question_id" DROP DEFAULT;
 CREATE INDEX "polls_choice_7aa0f6ee" ON "polls_choice" ("question_id");
 ALTER TABLE "polls_choice"
   ADD CONSTRAINT "polls_choice_question_id_246c99a640fbbd72_fk_polls_question_id"
     FOREIGN KEY ("question_id")
     REFERENCES "polls_question" ("id")
     DEFERRABLE INITIALLY DEFERRED;
 
 COMMIT;
 ```
 
2.python manage.py shell

打开Django控制台，这句话相当于打开Python控制台之后执行 `import django; django.setup()` 一旦进入Shell就可以随意操作了

```
>>> from polls.models import Question, Choice   # Import the model classes we just wrote.

# No questions are in the system yet.
>>> Question.objects.all()
[]

# Create a new Question.
# Support for time zones is enabled in the default settings file, so
# Django expects a datetime with tzinfo for pub_date. Use timezone.now()
# instead of datetime.datetime.now() and it will do the right thing.
>>> from django.utils import timezone
>>> q = Question(question_text="What's new?", pub_date=timezone.now())

# Save the object into the database. You have to call save() explicitly.
>>> q.save()

# Now it has an ID. Note that this might say "1L" instead of "1", depending
# on which database you're using. That's no biggie; it just means your
# database backend prefers to return integers as Python long integer
# objects.
>>> q.id
1

# Access model field values via Python attributes.
>>> q.question_text
"What's new?"
>>> q.pub_date
datetime.datetime(2012, 2, 26, 13, 0, 0, 775217, tzinfo=<UTC>)

# Change values by changing the attributes, then calling save().
>>> q.question_text = "What's up?"
>>> q.save()

# objects.all() displays all the questions in the database.
>>> Question.objects.all()
[<Question: Question object>]
```







