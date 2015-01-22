---
layout: post
title: "java注解小记"
description: ""
category: "知识整理"
tags: [Java,注解]
---

有没有在写代码的时候见过这样的东西`@Override`很早的时候我一直以为是IDE提供的标示，后来才注意到这其实是Java提供的一种代码注释。本身对程序没有任何的影响，但是可以给予IDE和编译器更多的信息。

## 是什么

一个完整的注释可能是这样的

```
@Author(
   name = "Benjamin Franklin",
   date = "3/27/2003"
)
class MyClass() { ... }
```

或许写成这样更熟悉一些 

```
@Author(name = "Benjamin Franklin", date = "3/27/2003")
class MyClass() { ... }
```

这是包含两个属性的注释，包含一个属性的注释是这样的 

```
@SuppressWarnings(value = "unchecked")
void myMethod() { ... }
```

对于只有一个属性，而且叫`value`其实可以省略成这样`@SuppressWarnings("unchecked")`现在是不是很熟悉了~~


## 写一个

写注释太简单不过了~~~ 只是语法比较特殊而已

```
//ClassPreamble.java
@interface ClassPreamble {
   String author();
   String date();
   int currentRevision() default 1;
   String lastModified() default "N/A";
   String lastModifiedBy() default "N/A";
   // Note use of array
   String[] reviewers();
}
```

这样，我们再写一个类的时候可以给它加这样的注释了

```
@ClassPreamble(author = "", date = "", reviewers = { "" })
public class MainActivity extends Activity {
  ...
}
```

## 那这又能做什么用呢？

经常见到的一些

1. @Deprecated 告诉编译器这个方法或者类已经不被支持了
2. @Override 告诉编译器这个方法是重载的方法
3. @SuppressWarnings 把某个编译警告强制隐掉

还有一些注解是给注解添加注解的,比如

1. @Retention 告诉编译器这个注解使用的场景 三种选择：RetentionPolicy.SOURCE、RetentionPolicy.CLASS、RetentionPolicy.RUNTIME。
2. @Documented 这个可以告诉java文档工具把注解编入文档。
3. @Target 告诉编译器，这个注释的适用类型

```
@Target(ElementType.METHOD) @Retention(RetentionPolicy.RUNTIME)
public @interface MethodAnnotation { }
```

这个注解的注解告诉了编译器，`MethodAnnotation`这个注解只能作用于方法，并且在运行时的时候起作用。


好像还是没什么用处，对不对？有一个JSON的解析库叫gson，它有一个功能可以把一个JSON对象直接解析成一个Java的类，大概这样

```
//json数据
{
"name":"ntop",
"age":30
}
//.java
class Persion {
  @SerializedName("name")
  private String mName;
  @SerializedName("age")
  private int mAge;
}
//
Persion p = new Gson().from(jsonstring, Persion.class)
```

看上去是不是很神奇，其实这里面用到的就是注解的功能，可以自己写一个类似的函数，把json的字段映射到注解的名字

```
  private Object parse(String jsonstring, Object obj) throws Exception {
    	JSONObject json = new JSONObject(jsonstring);
    	Field[] fields = obj.getClass().getDeclaredFields();
    	for(Field field : fields){
    		SerializedName serianame= field.getAnnotation(SerializedName.class);
    		if(serianame != null){
    			field.set(obj, json.opt(serianame.value()));
    		}
    		
    	}
    	return obj;
  }
  
  Persion p = (Persion) parse("{\"name\":\"ntop\",\"age\":30}", new Persion() );
  System.out.println("name:" + p.mName);
  System.out.println("age:" + p.mAge);  
```

内容来源：

1. [Annotations](http://docs.oracle.com/javase/tutorial/java/annotations/index.html)
2. [How Do Annotations Work?](http://www.objectpartners.com/2010/08/06/how-do-annotations-work/)
