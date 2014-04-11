---
layout: post
title: "使用github pages的小技巧"
description: ""
category: 
tags: []
---
{% include JB/setup %}

至今为止对于GithubPages最令人沮丧的就是安装 Jekyll ，Jekyll 其实非常难用，而且需要安装ruby，还存在一些版本问题。
发现一些小技巧可以不必下载 Jekyll , 也能够发文档。

其实非常简单的，在 _posts 目录下按照 `2014-04-11-xxxx-xxx.md` 的格式简历文档，然后在文档的开头加上这段代码：

```
---
layout: post
title: "使用github pages的小技巧"
description: ""
category: 
tags: []
---
{% include JB/setup %}

正文部分
```

就可以了，title 部分填写文档标题 , 其他的category,tags 的配置也都顾名思义，不做介绍。

文档发布之后，直接push到github, github 会自动解析这些配置，生成合适的文档。
