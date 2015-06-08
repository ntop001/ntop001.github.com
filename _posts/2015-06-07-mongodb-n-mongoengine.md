---
layout: post
title: "mongodb and mongoengine"
description: ""
category: 
tags: []
---

记录一下mongodb和mongoengine的使用

## 安装MongoDB

MongoDB 的官方文档已经非常的详尽了，本来没有必要再抄录过来，所以仅作一些索引。安装可以参考[文档](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-os-x/?_ga=1.177060902.2092388320.1433676967)，在mac上用brew安装是比较方便的，只需要一行命令即可。

```
brew install mongodb
```

启动MongoDB只需要执行`mongod`命令即可，默认的情况下，它会在 `/data/db` 目录下创建数据库。如果要修改数据库的位置，可以使用 `mongod --dbpath <path to data directory>` 命令指定数据库位置。要注意的是要手动先创建 `/data/db` 这个目录

退出MongoDB只需要ctr+C终端当前mongod进程即可。

```
➜  ~  sudo mkdir -p /data/db
Password:
➜  ~  sudo mongod 
2015-06-07T20:13:11.932+0800 I JOURNAL  [initandlisten] journal dir=/data/db/journal
2015-06-07T20:13:11.933+0800 I JOURNAL  [initandlisten] recover : no journal files present, no recovery needed
....
2015-06-07T20:13:11.946+0800 I CONTROL  [initandlisten] MongoDB starting : pid=966 port=27017 dbpath=/data/db 64-bit host=ntopdeMacBook-Pro.local
....
2015-06-07T20:13:12.038+0800 I NETWORK  [initandlisten] waiting for connections on port 27017
```

"2015-06-07T20:13:11.946+0800 I CONTROL  [initandlisten] MongoDB starting : pid=966 port=27017 dbpath=/data/db 64-bit host=ntopdeMacBook-Pro.local" 这条log打印了数据库的位置，端口号，host地址等信息。


## 连接数据库

MongoDB提供了多种语言版本的驱动来连接到数据库，在之前使用Django的时候已经使用了通过pymongo驱动来连接数据库，这里为了方便，会使用MongoDB自带的shell版本驱动对数据做简单的操作。[mongo](http://docs.mongodb.org/manual/reference/mongo-shell/) 提供了简单的数据库操作的指令。

新开一个终端，输入`monggo` 会进入mongo控制台。

```
➜  ~  mongo
MongoDB shell version: 3.0.3
connecting to: test
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
	http://docs.mongodb.org/
Questions? Try the support group
	http://groups.google.com/group/mongodb-user
Server has startup warnings: 
2015-06-07T20:13:11.946+0800 I CONTROL  [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.
2015-06-07T20:13:11.946+0800 I CONTROL  [initandlisten] 
2015-06-07T20:13:11.946+0800 I CONTROL  [initandlisten] 
2015-06-07T20:13:11.946+0800 I CONTROL  [initandlisten] ** WARNING: soft rlimits too low. Number of files is 256, should be at least 1000
> 
```

这时候会发现mongod控制台打印了一条消息,表示已经有连接连到数据库。"connecting to: test" 这条标示连接到test数据库。

```
2015-06-07T20:22:50.321+0800 I NETWORK  [initandlisten] connection accepted from 127.0.0.1:52416 #1 (1 connection now open)
```

mongo 控制台是使用[js语法](http://docs.mongodb.org/manual/reference/method/)操作的，详细的API见文档，简单演示一下，如何操作。

```
//创建一个表 blogs
> db.createCollection('blogs')
{ "ok" : 1 }
> show collections
blogs
system.indexes
//在表中插入一条
> db.blogs.insert({title:'hello',content:'world'})
WriteResult({ "nInserted" : 1 })
//查询表的所有内容
> db.blogs.find()
{ "_id" : ObjectId("55743ce6b190d00add1e057e"), "title" : "hello", "content" : "world" }
//创建一个数据库 fb
> use fb
```

## MongoEngine

因为Web框架使用Django的缘故，但是又希望使用MongoDB作存储，Django本身是支持ORM的，但是ORM系统支持的数据库中并不包含MongoDB所以只能使用一个第三方的方案 ———— MongoEngine。

确切的说我现在做的工程是这样的：Django + MongoEngine + MongoDB

安装MongoEngine很简单，使用Python的包管理工具PIP只需要一条命令 `pip install mongoengine`。需要注意的是MongoEngine是依赖于PyMongo的，也要安装一下pymongo `pip install pymongo`。

> 注意版本问题，我在使用MongoEngine（0.9.0）的时候它还不能支持>=3.0版本的pymongo，[github 上的回答](https://github.com/MongoEngine/mongoengine/issues/935)要安装2.8版本的pymongo。

## MongoEngine基本使用

写了一小段代码 'test.py' ,演示一个hello world的使用。 

```
➜  mongoenginetest  cat test.py 
from mongoengine import *

// 连接到前面的fb数据库
connect('fb')
//定义一个ORM文档
class Blog(Document):
	title = StringField(required=True)
	content = StringField(required=True)
//实例化一个文档并保存
b = Blog()
b.title = 'deep into python'
b.content = 'python is not a good program lang'

b.save()
//从数据库读回数据
for blog in Blog.objects:
	print b.title
```

上面一段代码，连接到了fb数据库，然后定义一个Blog对象，实例化后保存到数据库。如果这时候打开mongo控制台，连接到我们的数据库，可以看到刚刚保存的结果。

```
➜  ~  mongo
MongoDB shell version: 3.0.3
connecting to: test
Server has startup warnings: 
2015-06-08T17:12:32.178+0800 I CONTROL  [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.
2015-06-08T17:12:32.178+0800 I CONTROL  [initandlisten] 
2015-06-08T17:12:32.178+0800 I CONTROL  [initandlisten] 
2015-06-08T17:12:32.178+0800 I CONTROL  [initandlisten] ** WARNING: soft rlimits too low. Number of files is 256, should be at least 1000
> show dbs
fb     0.078GB
local  0.078GB
test   0.078GB
> use fb
switched to db fb
> show collections
blog
op
system.indexes
> db.blog.find()
{ "_id" : ObjectId("55755c83984e3606ce87b488"), "title" : "deep into python", "content" : "python is not a good program lang" }
> 
```
