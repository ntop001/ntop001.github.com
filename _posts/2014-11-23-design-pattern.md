---
layout: post
title: "关于设计模式的想法"
description: ""
category: "知识整理"
tags: []
---

设计模式23种，我一般都叫做“剑二十三” 这是风云里面的一套剑路，恰好可以描述设计模式的意义，它是前人在编码过程中总结的
针可以复用的编码套路，所以如果你已经写了很长一时间代码了，即使不知道设计模式，那么你可能已经在不知不觉中自己重新发明
了这些模式。有时候重新回顾一下23中设计模式，有利于优化现有的代码，可能你糊里糊涂的绞尽脑汁做了一个复杂的业务逻辑，但是很可能这种逻辑前人早就遇到过而且有更好的实现方案，那么为什么不“抄”一个过来呢，设计模式就是答案啊。虽然不是所有的逻辑都可在设计模式中找到答案，但是针对大部分情况，已经够用了。下面记录一些自己用到的。


## 单例

几乎每次面试到设计模式的时候，对方的回答都是从单例开始，因为单例简单而且比较实用吧, 单例首先会隐藏构造方法，这样使
用者就无法自行构造此类，然后通过静态方法暴露实例，这样就可以在方法内部控制实例的数量，因为叫“单例”所以我们只有一个实例。

```
public class DesignPattern {
    private static DesignPattern Instance = new DesignPattern();
    private DesignPattern(){}
    
    public static DesignPattern getInstance(){
        return Instance;
    }
}
```

设计单例的时候我一般会考虑这个类是做什么用的，需要实例化多个吗，单例类的职责是否唯一等等，在写SDK的时候，SDK的实现就是一个单例，因为对于开发者来说无论统计还是广告SDK对外只要暴露最简单的接口就行了，开发者不需要自己实例一个对象，那样会增加集成的难度。

## 策略和状态模式

把这两种模式放在一起是因为从具体的实现和使用上来看这两者是无差的，但是在不同的使用场景下，你可能需要一个比较好的名字，这样代码更易于阅读所以有时候可以叫状态（State）模式，有时候叫策略（Strategy）模式。统计分析代码中判断是否发送log，最开始的时候用的是一个巨大的switch语句根据设置的策略（用一个int值表示）来判断，最后使用策略模式实现，代码的易读和可维护性提高了很多，而且容易添加新的策略。

```
public class AbstractStrategy {
        public boolean shouldSendLog(){
            return true;
        }
    }

    //实时发送
    public class SendImmediately extends  AbstractStrategy{
        public boolean shouldSendLog(){
            ....
        }
    }
    //启动时发送
    public class SendAtLaunch extends AbstractStrategy{
        public boolean shouldSendLog(){
            ....
        }
    }
    //设置发送策略
    AbstractStrategy aStrategy;
    public void setStrategy(int i){
        switch(i){
            case 0:
                aStrategy = new SendImmediately();
                break;
            case 1:
                aStrategy = new SendAtLaunch();
                break;
            default:
                aStrategy = new SendAtLaunch();

        }
    }
    
    if(aStrategy.shouldSendLog()){
      ...
    }
```

这段代码差不多展示策略模式的实现, 通过实例化一个抽象类，然后通过几个子类实现各自的业务逻辑，这种方法即使没有学习过设计
模式很多人都会自己想到，确实一种很普遍的用法。

## 模板方法

在父类中定义一组方法的执行顺序，在子类实现中实现具体的逻辑

```
public abstract class MyActivity {
        public abstract void onCreate();
        public abstract void onResume();
        public abstract void onPause();
        public abstract void onDestroy();

        public void lifeCircle(){
            onCreate();
            onResume();
            onPause();
            onDestroy();
        }
    }
```

写这个例子实在是希望做Android的人可以更容易理解一些，我们经常继承的父类Activity和实现它的那几个方法(`onCreate`,`onResume` ... )，模板方法就类似于那些方法的作用，提前定义好了执行顺序，就差你来实现具体逻辑了。（PS：不能说Android中那几个生命周期函数就是模板方法，只是说明用处）

## 装饰模式

Java 中最典型的例子就是 `InputStream` 和它的各种实现了比如 `FileInputStream` 、`ObjectInputStream` 、`BufferedInputStream` 等等，每种实现都在原来的实现上添加一点小小的功能.

```
 InputStream is = new FileInputStream("123.txt");
 InputStream bis = new BufferedInputStream(is);
```
这样原来的纯 `FileInputStream` 就拥有了`Buffer`功能。SDK中唯一使用到类似装饰模式的点是在提供H5统计的时候为了能够让
webview 拦截到js事件，SDK需要实现 `WebChromeClient` 接口，但是开发者也需要实现这个接口，所以采用装饰模式实现了这个封装，这样开发者对这个类的改变，在SDK中不会改变，而SDK
只添加一些自己需要的功能就够了。

## 访问者

这是让我非常喜欢的一种模式，因为它的解决了我很久以来都很困扰的一件事件——”协议更新了，怎么最小代价的把更变更新到发送的log上去“。 

```
    interface ProtocolAction {
        public void put(Log l);
    }

    Log aLog = null;
    public void addProtocol(ProtocolAction p ){
        p.put(aLog);
    }

    class Event implements ProtocolAction {
        public void put(Log l){
            ...
        }
    }
```
大概这样吧，通过 `ProtocolAction`接口提供了一个通向Log的快速访问渠道，这样每次更新协议的时候不需要更新其他的代码，只要再实现一个 `ProtocolAction` 就够了，比如添加Error字段

```
   class Error implements  ProtocolAction {
        public void put(Log l){
            ...
        }
    }
   
    addProtocol(new Error());
```
其余的代码都不会因为协议的变化而受到影响，关于这个模式的使用，还可以看下[这里](https://github.com/ntop001/AXMLEditor/blob/master/src/com/umeng/editor/decode/XMLVisitor.java)，我通过访问者模式遍历打印xml格式的结构，代码可以写的非常简洁。

## 观察者 

这种模式在有UI的编程中算是仅次于单例的吧，给按钮添加点击事件或者是开个线程去更新数据的时候显示进度等等都是观察者的实现。

## 外观模式(Facade)

与其说是一种模式不如说是一种写代码的情怀，如果见到类似 `XXXFacade` 的类那么他可能就是外观模式的一种实现了，外观模式解决的是代码整洁问题，让一个类负责它该做的且专一的事，而不是暴露很多无关的不相干的接口。

## 建造者模式

看看下面这段代码是不是很熟悉

```
      AlertDialog ad = new AlertDialog.Builder(context)
                    .setMessage("hehe")
                    .setTitle("title")
                    .setIcon(R.drawable.icon)
                    .create()；
```
对，这就是典型的建造者模式，它主要用来解决构造复杂对象的问题，比如AlertDialog就算的上，一个对话框需要设置标题（Title）、显示的内容（message）可能还会配一个图标（Icon）还有回调事件等等好多。如果通过一个构造方法来实现，那么这个方法的参数列表会很长，很难用。如果让Dialog直接提供这些方法，那么会让Diallog本身承载很多自己不是很关心的方法，另外可能会缺少某些参数构造步骤也变得复杂，所以把这部分功能剥离到一个Builder中既可以控制Dialog的单一职责还可以使整个构造过程更加简单。

## TODO
