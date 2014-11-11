---
layout: post
title: "Java中的序列化"
description: ""
category: "知识整理"
tags: [SDK]
---


有木有这样的需求，需要把一些数据保存在本地文件，但是这些数据本身是结构化的，所以保存的时候不是简单的写文件还需要有一定格式
这样下次还可以把这些结构化的数据读出来。

## Properties

`java.util.Properties` 可以实现这样的需求,把一对键值保存在文件中，再读出来。

```
	Properties p = new Properties();
	p.setProperty("abc", "123");
		
	p.store(new FileOutputStream("hehe"), "no comment");
		
	Properties p2 = new Properties();
	p2.load(new FileInputStream("hehe"));
		
	System.out.println("abc: " + p2.getProperty("abc"));
```

打开文件`hehe` 可以看到如下的保存格式 

```
#no comment
#Tue Nov 11 14:00:46 CST 2014
abc=123
```
但是 Properties 类只支持String类型的key-value 如果需要保存的数据很多而且有嵌套结构保存的过程就会变得异常复杂。

## Java 序列化

序列化功能可以把一个类序列化为一个byte[] 这样就不需要逐个保存类中的每个字段了。实现
系列化功能非常简单，只需要实现 Serializable 接口。

```
	public static void main(String[] args) throws Exception{
		
		SeriaTest st = new SeriaTest();
		st.a = "b";
		st.b= 3;
		
		ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("st.data"));
		oos.writeObject(st);
		oos.flush();
		oos.close();
		
		ObjectInputStream ois = new ObjectInputStream(new FileInputStream("st.data"));
		SeriaTest st2 = (SeriaTest)(ois.readObject());
		ois.close();
		
		System.out.println("A:" + st2.a);
		System.out.println("A:" + st2.b);
		
	}
	
	static class SeriaTest implements Serializable {
		String a ;
		int b ;
		byte[] c = {3,4,5};
	} 
	
```
这样就可以很容易的把 `SeriaTest.java` 这个类的实例的所有内容保存到文件中。编译这段代码的时候会报一个warning 忽视这个warning会
产生一个非常严重的后果。

> The serializable class SeriaTest does not declare a static final serialVersionUID field of type long

如果用Eclipse自动fix这个warning会添加一个字段并付一个随机值 `private static final long serialVersionUID = -2364974005302879154L;` 这个特殊的字段实际上是用来标示序列化版本的，比如这次你把一个类保存到本地，后来因为某些原因修改了
这个类，那么这个类已经变化了，你需要通过这个字段来标示这些变化。在没有这个字段的情况下，类只要发生变化，就无法再从字节流中
反序列化解析出这个类了。所以实现 `Serializable` 这个接口的时候一定要记得设置  `private static final long serialVersionUID = 0L;` 并初始化一个合理的值，每次有变化的时候增量更新，大的版本是可以兼容小的版本的这样就不会有问题。

有时候类中会引入一些不可或者不想序列化的字段可以用 `transient` 关键字标示，被标示的字段不会再被序列化。有木有觉得很奇怪实现
`Serializable` 这个接口没有实现任何的方法，其实这个接口是有定义方法的并标示为private 如果想自定义序列化行为可以这样覆盖

```
 private void writeObject(java.io.ObjectOutputStream out)
       throws IOException {
     // write 'this' to 'out'...
   

   private void readObject(java.io.ObjectInputStream in)
       throws IOException, ClassNotFoundException {
     // populate the fields of 'this' from the data in 'in'...
   }
```

使用Java自带的序列化经常会出现一些意想不到的bug，上面提到的没有添加序列化版本就是其一，另外一个经常遇到的就是混淆，在Java开发
中经常会把一个类混淆掉，混淆之后类中的各个字段就会变成abcb 这些没有任何意义的名称，这个潜在的问题就在于混淆修改了原来的类，如果每次序列化的结果不一样就这个类就无法之前保存的数据了。

所以一定要记得加上 `-keep class * implements java.io.Serializable { *; }`


## 其他替换方案

真实场景中使用Java自带的序列化方式，无论速度还是序列化数据大小都是非常差的，JSON是比较推荐的方案，基于本文格式易读而且精简
高效。或者是用GSON 它提供了类到json协议的映射，但是可维护性比java 自带的序列化好多了。

## Android

在Android开发中一般使用 `SharedPreferences` 来存储少量的键值数据，大量的重复的数据一般使用Sqlite数据库存储，少量的结构化
数据并没有太好的存储方式，最好用的还是使用JSON格式保存。曾经想过一种不太好的方案，先把一个类序列化并编码成字符串，然后再用
 `SharedPreferences` 来保存，这样考虑主要是因为数据格式嵌套太多，而且还有数组类型，但是真实的数据却很少不值得用文件保存。
 编码用到这几个方法, 这个方法是从 Apache 的某个库找到的。
 
 ```
 public static String serialize(Serializable obj) {
        if (obj == null) return "";
        try {
            ByteArrayOutputStream serialObj = new ByteArrayOutputStream();
            ObjectOutputStream objStream = new ObjectOutputStream(serialObj);
            objStream.writeObject(obj);
            objStream.close();
            return encodeBytes(serialObj.toByteArray());
        } catch (Exception ignore) {ignore.printStackTrace();
        }
        
        return null;
    }
    
    public static Object deserialize(String str) {
        if (str == null || str.length() == 0) return null;
        try {
            ByteArrayInputStream serialObj = new ByteArrayInputStream(decodeBytes(str));
            ObjectInputStream objStream = new ObjectInputStream(serialObj);
            return objStream.readObject();
        } catch (Exception ingore) {
        }
        
        return null;
    }
    
    public static String encodeBytes(byte[] bytes) {
        StringBuffer strBuf = new StringBuffer();
    
        for (int i = 0; i < bytes.length; i++) {
            strBuf.append((char) (((bytes[i] >> 4) & 0xF) + ((int) 'a')));
            strBuf.append((char) (((bytes[i]) & 0xF) + ((int) 'a')));
        }
        
        return strBuf.toString();
    }
    
    public static byte[] decodeBytes(String str) {
        byte[] bytes = new byte[str.length() / 2];
        for (int i = 0; i < str.length(); i+=2) {
            char c = str.charAt(i);
            bytes[i/2] = (byte) ((c - 'a') << 4);
            c = str.charAt(i+1);
            bytes[i/2] += (c - 'a');
        }
        return bytes;
    }

 ```
 
 现在还是使用json来保存了，简单可依赖。
