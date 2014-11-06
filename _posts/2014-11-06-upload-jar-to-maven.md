---
layout: post
title: "上传自有jar包到maven central repo"
description: ""
category: 
tags: []
---
{% include JB/setup %}



# 上传Jar到maven central

## 什么是maven

[maven](http://maven.apache.org/index.html) 是一个做软件工程管理的软件，可以用来做工程的构建(build)和发布。对于开发者来说大部分情况下，我们是使用maven来构建系统，比如把工程代码打包发布。这是一个基础并且简单的功能，maven 通过一个配置文件（pom.xml）来配置需要执行的一些命令(比如编译文件、生成jar包、复制文件等等)，类似的工具有很多，现在Gradle也很流行。但是maven的配置文件是基于xml的，这样的语法用起来不是很灵活导致配置文件通常巨大无比。在maven工程中添加一些依赖的时候非常简单，比如：
```
 <dependency>
    <groupId>com.google.android</groupId>
    <artifactId>android</artifactId>
    <version>4.0.1.2</version>
    <scope>provided</scope>
 </dependency>
```
maven 会自动从 [CentralRepository](http://search.maven.org/) 寻找合适的依赖并下载到本地，CentralRepository 是maven的包管理中心，使用maven添加的依赖都可以从这里找到适当的包.

##上传Jar到Maven

Maven官网并没有提供任何可以直接上传到CentralRepo的方式，见[文档](http://maven.apache.org/guides/mini/guide-central-repository-upload.html)。 目前用的最多的是 [sonatype](http://www.sonatype.org/) sonatype 有两个主要功能：

* Maven repository hosting service: You can deploy snapshots, stage releases, and promote your releases so they will be published to Central Repository
* Manual upload of artifacts: The process is mostly automated with turnaround time averaging 1 business day.

注意网址是.org 的不是 .com 的那个。如何一步一步进行可以看这个[文档](http://central.sonatype.org/pages/ossrh-guide.html) 其实就两件事

1. [注册sonatype账号]( https://issues.sonatype.org/secure/Signup!default.jspa )
2. [创建一个 sonatype工程](https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134)

注册账号没有太多好说的，但是创建 sonatype 的工程师需要审核的，并且有许多填空可能你并不知道是什么意思。在sonatype上创建一个工程是从创建一个issue开始的，在issue中选择issue类型为 NewProject 提交审核并且被fix之后工程创建才会成功。

1. Project 一定要选择 Community Support - Open Source Project Repository Hosting
2. Issue Type 如果上一步完成正确可以看到下拉框可以选择 New Project 选项
3. Summary 工程名
4. Description 一段描述
5. Attachment 附件，不知道做什么用的，忽略
6. Group Id 这个和java的包名命名规范是一样的，比如：com.foo
7. Project URL 项目地址
8. scm 源码托管地址比如 github、googlecode
9. 其他默认

这样就可以提交一个issue了，这个issue过了一两天就可以审核通过，刚刚提交的时候是 unresolved 状态，审核通过之后就是 resolved 状态了。

接下来再把你的工程源码和jar包上传到sonatype就行了，sonatype 会把你的东西再次同步到Maven的central repo。有两种方式可以把jar包上传到sonatype

1. 打开 https://oss.sonatype.org/ 用之前的sonatype账号登陆，在网页上提交
2. 修改Maven配置使用 Maven 命令上传(这个比较方便程序员操作)

但是在这之前还需要一些准备

1. 准备 pom 文件
2. jar 包签名

一个最减的pom文件填写这些东西就足够centralrepo识别了
* modelVersion
* groupId
* artifactId
* version
* packaging (defaults to jar)
* name
* description
* url
* licenses
* scm/url
* scm/connection
* scm/developerConnection
* developers

一个简单的模板：

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">  
    
  <modelVersion>4.0.0</modelVersion>  
  <groupId>[groupId]</groupId>  
  <artifactId>[artifactId]</artifactId>  
  <version>1.0</version>  
  <name>[project name]</name>  
  <description>[project description]</description>  
  <url>[project url]</url>  
  <licenses>  
    <license>  
      <name>[license name]</name>  
      <url>[license url]</url>  
    </license>  
  </licenses>  
  <scm>  
    <url>http://fisheye.jboss.org/browse/[path to repo]</url>  
    <connection>http://anonsvn.jboss.org/repos/[path to repo]</connection>  
    <developerConnection>https://svn.jboss.org/repos/[path to repo]</developerConnection>  
  </scm>  
  <developers>  
    <developer>  
      <id>[jboss.org username]</id>  
      <name>[developer name]</name>  
      <organization>[developer organization]</organization>  
    </developer>  
  </developers>  
    
</project>  
```

其次对于jar文件sonatype（centralrepo）需要这4种文件，都是必须上传的：

1. ossrh-test-1.2.pom 上面介绍的pom文件
2. ossrh-test-1.2.jar 需要上传的jar包
3. ossrh-test-1.2-javadoc.jar java doc 可以伪造，上传一个空文件就行
4. ossrh-test-1.2-sources.jar 如果你不愿意共享代码也伪造一个吧

有了这四个文件再签了名就可以使用 sonatype 的网页上传了，如果缺少某个文件会有错误信息。

[签名](http://blog.sonatype.com/2010/01/how-to-generate-pgp-signatures-with-maven/#.VFhQhfmSyaQ)使用的是 [gpg](https://www.gnupg.org/index.html) 安装之后，可以验证一下：

```
>gpg --version
gpg (GnuPG) 1.4.7
Copyright (C) 2006 Free Software Foundation, Inc.
...
```

创建一对秘钥

```
>gpg --gen-key
```
创建秘钥过程中会问类型、大小、有效时间等等如果没有特别需求一路默认就可以了，接下来问名字、邮件、说明稍微注意一下也就行了，最后会让你输入秘钥口令，这个口令在签名的时候需要。gpg 和 java的签名机制差不多需要填写的内容页差不多，如果做过Android的打包签名应该很熟悉。

```
>gpg --list-keys
.../gnupg\pubring.gpg
--------------------------------------------------------
pub   1024D/33D460AE 2014-11-04 [expires: 2017-11-03]
uid                  liuzhaoyun (work@umeng) <liuzhaoyun@umeng.com>
sub   2048g/524CA703 2014-11-04 [expires: 2017-11-03]
```
会打印一下目前生成的key 包括公钥、私钥和刚刚填写的作者信息。对于Maven来说我们需要把公钥信息上传到Maven的服务器, 这样他就可以对签名的文件做验证。

```
>gpg --keyserver hkp://pool.sks-keyservers.n
et --send-keys  33D460AE
gpg: sending key 33D460AE to hkp server pool.sks-keyservers.net
```
我在上传的过程中老是报一个错误：

```
>gpg --keyserver hkp://pool.sks-keyservers.net --send-keys
  33D460AE
gpg: sending key 33D460AE to hkp server pool.sks-keyservers.net
gpg: system error while calling external program: No error
gpg: WARNING: unable to remove tempfile (out) `C:\Users\ADMINI~1\AppData\Local\T
emp\gpg-E59A64\tempout.txt': No such file or directory
gpg: no handler for keyserver scheme `hkp'
gpg: keyserver send failed: keyserver error
```

后来才发现git竟然也自带了一个gpg工具，自带的工具功能不全无法解析 hkp 协议，导致上传失败，我的解决方式是 cd 到gnu版本的gpg目录下操作的。

使用gpg签名非常简单，比如对 `temp.java` 签名 `gpg -ab temp.java` 会生成 `temp.java.asc` 的文件。

对于上面的四个文件分别签名之后就会多出4文件，把这个8个文件在sonatype网页上传就OK了。

现在回到[这个](https://oss.sonatype.org/)页面

![staging](http://central.sonatype.org/images/staging-upload.png)

点击左侧的 Staging Upload -> 选择 Artifact(s) with a POM ->先把 .pom 文件上传 ->再在下面的artifact(s) upload 表单中把剩余7个文件上传了，过一会点击左侧的 Staging Repositories 就可搜索刚刚上传的artifact 关键字就可以看到自己上传的东西了。这个[链接](http://central.sonatype.org/pages/manual-staging-bundle-creation-and-deployment.html) 有一些手动签名并上传的信息。 这个时候要看着工程的状态时 Open 的还是 Closed 的，如果是Open的那是不是有什么问题（正常情况下是Closed的，这个时候可以点击Release按钮发布工程）。如果是Closed的点击Release就可以发布了。（如果是第一次，还需要回到提交新工程的那个页面，找到issue下面的comments回复一个告诉帮你审核的人我的工程需要同步到maven central repo）这样jar包就可以同步到 central repo 了。其实关键点还是把工程先提交到 sonatype 接下来的事情就很简单了。

对于使用maven命令这种更接近程序员思考的方式，网上有许多资料可以参考，基本的思路还是透过 maven 的配置文件 把本地的东西打包好之后就直接上传到sonatype，可以参考[这个](http://www.trinea.cn/dev-tools/upload-java-jar-or-android-aar-to-maven-center-repository/ ) 

























