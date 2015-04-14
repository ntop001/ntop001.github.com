---
layout: post
title: "依赖注入"
description: ""
category: 
tags: []
---

[Ioc](http://en.wikipedia.org/wiki/Inversion_of_control)
[Dependency injection](http://en.wikipedia.org/wiki/Dependency_injection)

依赖注入和控制反转，这些概念每次看到都会有不同的理解，这篇文章部分翻译了上面的wiki链接，部分加入自己的理解。

## 控制反转（Ioc）

这个概念应该表达的问题处理的方式是这样的：在过去的编程中我们会写一些自有的代码，并调用一些可以复用的库来完成任务，但是Ioc中是通过复用的库来回调自有的代码。这种设计的方法一个常见的例子就是TUI和GUI中事件处理的区别，在学习C语言的时候经常会做一些简单的程序，在终端上输出一个菜单，等待用户做出选择阻塞执行。但是在后来的GUI（比如Windows）系统中，所有的菜单事件是通过系统框架的回调机制处理的。

在实践Ioc的程序中，需要先实现程序框架处理应用一般的行为（比如系统事件，菜单，鼠标等），开发者的代码只需要填写一些回调事件来处理真正的问题。在Ioc程序中可以复用的代码和处理问题的代码是严格分离的，常见的Ioc实现有

* 软件框架
* 回调接口
* 子例程
* 事件循环
* 依赖出入

设计的目的（或者说好处吧）

1. 解耦任务的执行和实现
2. 模块更独立
3. To free modules from assumptions about how other systems do what they do and instead rely on contracts. (不会译)
4. 避免替换新的模块的时候出问题


具体的实现技术参考

1. 使用工场模式 
2. 服务定位模式 (service locator pattern)
3. 依赖注入
  * 构造方法注入
  * 参数注入
  * setter 注入
  * 接口注入
4. 上下文查找 ( contextualized lookup ) 这种模式是通过在容器中查找对应服务
  ```
  public interface Orange {
    // methods
  }
 
  public class AppleImpl implements Apple, DependencyProvision {
    private Orange orange;
    public void doDependencyLookup(DependencyProvider dp) throws DependencyLookupExcpetion{
    this.orange = (Orange) dp.lookup("Orange");
  }
  // other methods
  }
  ```
5. 使用模板方法模式
6. 策略模式

## 依赖注入 (Dependency injection)

依赖注入主要用来解决模块或者服务之间过度耦合的问题（其实会带来一个额外的优点，容易测试），如果 AClient 依赖于BService ，那么可以通过一个外部容器（或者框架）来实例化B，并把它传递给A，而不是直接在A中new 一个B。这个模块原则把依赖项的创建和它自己的行为分离。依赖注入需要实现四个元素：

1. 服务（依赖）the implementation of a service object
2. 依赖于服务的客户端 the client object depending on the service;
3. 客户端和服务通信接口 the interface the client uses to communicate with the service;
4. 把服务注入客户端的容器。 the injector object, which is responsible for injecting the service into the client.

有三种基本的依赖注入类型：

1. setter-based injection
2. interface-based injection 
3. constructor-based injection

一个典型的相互耦合的程序如下, Client 中直接初始化了 ServiceExample 。

```
// An example without dependency injection
public class Client {
    // Internal reference to the service used by this client
    private Service service;
 
    // Constructor
    Client() {
        // Specify a specific implementation in the constructor instead of using dependency injection
        this.service = new ServiceExample();
    }
 
    // Method within this client that uses the services
    public String greet() {
        return "Hello " + service.getName();
    }
}
```

如果通过依赖注入的方式解决这个问题

构造方法注入 

```
// Constructor
Client(Service service) {
    // Save the reference to the passed-in service inside this client
    this.service = service;
}
```

使用 Setter 接口注入

```
// Setter method
public void setService(Service service) {
    // Save the reference to the passed-in service inside this client
    this.service = service;
}
```

使用接口注入 

```
// Service setter interface.
public interface ServiceSetter {
    public void setService(Service service);
}
 
// Client class
public class Client implements ServiceSetter {
    // Internal reference to the service used by this client.
    private Service service;
 
    // Set the service that this client is to use.
    @Override
    public void setService(Service service) {
        this.service = service;
    }
}
```



