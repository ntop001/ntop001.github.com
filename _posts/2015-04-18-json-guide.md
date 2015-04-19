---
layout: post
title: "Gson 文档"
description: ""
category: 
tags: [Gson]
---

打算写一个可以把JSONObject直接转成Java对象的库，其实就是类似[Gson](https://code.google.com/p/google-gson/)提供的功能，但是Gson并不能满足需求，Gson 会把字符串先转成 `com.google.gson.JsonObject `
对象,然后再映射到自定义的Java类，我希望的库，可以把标准的 `org.json.JSONObject` 映射到Java类，或者可以通过一些接口支持更多的JSON java实现。
而最快的方式就是直接修改Gson的源码。所以先从文档下手，学习一下Gson的功能，以便更好的理解源码设计。

## Gson 使用

如何使用举例子比文字可能更直接，所以直接贴例子了

#### 基本类型(Primitives Examples)

```
(Serialization)
Gson gson = new Gson();
gson.toJson(1);            ==> prints 1
gson.toJson("abcd");       ==> prints "abcd"
gson.toJson(new Long(10)); ==> prints 10
int[] values = { 1 };
gson.toJson(values);       ==> prints [1]

(Deserialization)
int one = gson.fromJson("1", int.class);
Integer one = gson.fromJson("1", Integer.class);
Long one = gson.fromJson("1", Long.class);
Boolean false = gson.fromJson("false", Boolean.class);
String str = gson.fromJson("\"abc\"", String.class);
String anotherStr = gson.fromJson("[\"abc\"]", String.class);
```

#### 对象(Object Examples)

```
class BagOfPrimitives {
  private int value1 = 1;
  private String value2 = "abc";
  private transient int value3 = 3;
  BagOfPrimitives() {
    // no-args constructor
  }
}

(Serialization)
BagOfPrimitives obj = new BagOfPrimitives();
Gson gson = new Gson();
String json = gson.toJson(obj);  
==> json is {"value1":1,"value2":"abc"}

Note that you can not serialize objects with circular references since that will result in infinite recursion. 

(Deserialization)
BagOfPrimitives obj2 = gson.fromJson(json, BagOfPrimitives.class);   
==> obj2 is just like obj
```

#### 内嵌类(Nested Classes)

静态内嵌类是可以直接序列化和反序列化的，但是对于非静态内嵌类，是无法直接构造的，因为它的构造还需要外部类的引用，如果一定要序列化的话，可以把它改成静态内部类或者实现`InstanceCreator `接口。

```
public class A { 
  public String a; 

  class B { 

    public String b; 

    public B() {
      // No args constructor for B
    }
  } 
}
```

注意上面的B是无法直接序列化的。通过实现 `InstanceCreator` 的做法如下，对于`InstanceCreator`接口，还可以做到序列化没有0参数构造函数的功能。

```
public class InstanceCreatorForB implements InstanceCreator<A.B> {
  private final A a;
  public InstanceCreatorForB(A a)  {
    this.a = a;
  }
  public A.B createInstance(Type type) {
    return a.new B();
  }
}
```

但是官方文档并不推荐这样做，所以还是改成静态内部类吧。


#### 数组类型 (Array Examples)

支持多维数组

```
Gson gson = new Gson();
int[] ints = {1, 2, 3, 4, 5};
String[] strings = {"abc", "def", "ghi"};

(Serialization)
gson.toJson(ints);     ==> prints [1,2,3,4,5]
gson.toJson(strings);  ==> prints ["abc", "def", "ghi"]

(Deserialization)
int[] ints2 = gson.fromJson("[1,2,3,4,5]", int[].class); 
==> ints2 will be same as ints
```

#### 集合类型

```
Gson gson = new Gson();
Collection<Integer> ints = Lists.immutableList(1,2,3,4,5);

(Serialization)
String json = gson.toJson(ints); ==> json is [1,2,3,4,5]

(Deserialization)
Type collectionType = new TypeToken<Collection<Integer>>(){}.getType();
Collection<Integer> ints2 = gson.fromJson(json, collectionType);
ints2 is same as ints
```

注意上面集合类型的类型定义 `Type collectionType = new TypeToken<Collection<Integer>>(){}.getType()` 因为泛型的缘故无法获取到集合的类型，并且集合中应该是统一类型的，否则反序列化的时候无法确定类型。

#### 序列化和反序列化泛型 (Serializing and Deserializing Generic Types)

当调用 `toJson(obj)` 的时候Gson会调用 `obj.getClass()` 去获取序列化信息。同样把 `MyClass.class` 对象传入 `fromJson(json, MyClass.class)` 来反序列化。但是如果对象是泛型的时候由于java的类型擦除(Java Type Erasure)导致无法正确的序列化。

```
class Foo<T> {
  T value;
}
Gson gson = new Gson();
Foo<Bar> foo = new Foo<Bar>();
gson.toJson(foo); // May not serialize foo.value correctly

gson.fromJson(json, foo.getClass()); // Fails to deserialize foo.value as Bar
```

当Gson执行上面代码的时候获取到的类型信息是 `Foo.class` 而不是 `Foo<Bar>` 解决这个问题需要使用 `TypeToken` 来获取类型信息。

```
Type fooType = new TypeToken<Foo<Bar>>() {}.getType();
gson.toJson(foo, fooType);

gson.fromJson(json, fooType);
```

> The idiom used to get fooType actually defines an anonymous local inner class containing a method getType() that returns the fully parameterized type. 

#### 序列化和反序列化包含任意类型的集合类

比如下面这种类型：

```
//String
['hello',5,{name:'GREETINGS',source:'guest'}]
//Java Object
The equivalent Collection containing this is:
Collection collection = new ArrayList();
collection.add("hello");
collection.add(5);
collection.add(new Event("GREETINGS", "guest"));
//Event.java
class Event {
  private String name;
  private String source;
  private Event(String name, String source) {
    this.name = name;
    this.source = source;
  }
}
```

如果把java对象序列化成json字符串的话是没有问题的，但是如果把json串序反序列化成Java对象那么会出现很多问题，因为反序列化的过程丢失了很多类型信息。

文档中给出了三种方法，但是我认为最好不要这么做，容易出错

> Option 1: Use Gson's parser API (low-level streaming parser or the DOM parser JsonParser) to parse the array elements and then  use Gson.fromJson() on each of the array elements.This is the preferred approach. Here is an example that demonstrates how to do this.

> Option 2: Register a type adapter for Collection.class that looks at each of the array members and maps them to appropriate objects. The disadvantage of this approach is that it will screw up deserialization of other collection types in Gson.

> Option 3: Register a type adapter for MyCollectionMemberType and use fromJson with Collection<MyCollectionMemberType>
This approach is practical only if the array appears as a top-level element or if you can change the field type holding the collection to be of type Collection<MyCollectionMemberTyep>. 


#### 自定义序列化和反序列化类型 

Gson 是可以自己定义如何序列化和反序列化一个特定类型的，需要实现一个序列化处理器和一个反序列化处理器。

```
GsonBuilder gson = new GsonBuilder();
gson.registerTypeAdapter(MyType.class, new MySerializer());
gson.registerTypeAdapter(MyType.class, new MyDeserializer());
gson.registerTypeAdapter(MyType.class, new MyInstanceCreator());
```

下面一段代码演示如何实现一个序列化处理器 

```
private class DateTimeSerializer implements JsonSerializer<DateTime> {
  public JsonElement serialize(DateTime src, Type typeOfSrc, JsonSerializationContext context) {
    return new JsonPrimitive(src.toString());
  }
}
```

下面代码演示如何实现一个反序列化处理器 

```
private class DateTimeDeserializer implements JsonDeserializer<DateTime> {
  public DateTime deserialize(JsonElement json, Type typeOfT, JsonDeserializationContext context)
      throws JsonParseException {
    return new DateTime(json.getAsJsonPrimitive().getAsString());
  }
}
```

#### 实现实例构造器(Writing an Instance Creator)

这种需求往往是因为类本身没有提供没有参数的构造器，那么在反序列化的时候无法给这个类创造一个对象，所以需要实现Gson的 
`InstanceCreator`接口来提供一个可用的构造器。

```
private class MoneyInstanceCreator implements InstanceCreator<Money> {
  public Money createInstance(Type type) {
    return new Money("1000000", CurrencyCode.USD);
  }
}
```

对参数化类型(有了具体类型的泛型 [see](http://www.angelikalanger.com/GenericsFAQ/FAQSections/ParameterizedTypes.html#FAQ001)) 实现构造器和非泛型类是一样的，因为真正使用的时候类型信息是固定的。

```
class MyList<T> extends ArrayList<T> {
}

class MyListInstanceCreator implements InstanceCreator<MyList<?>> {
    @SuppressWarnings("unchecked")
  public MyList<?> createInstance(Type type) {
    // No need to use a parameterized list since the actual instance will have the raw type anyway.
    return new MyList();
  }
}
```

但是对于有些情况，比如下面的例子是需要知道参数化类型的具体参数类型的，需要这么做

```
public class Id<T> {
  private final Class<T> classOfId;
  private final long value;
  public Id(Class<T> classOfId, long value) {
    this.classOfId = classOfId;
    this.value = value;
  }
}

class IdInstanceCreator implements InstanceCreator<Id<?>> {
  public Id<?> createInstance(Type type) {
    Type[] typeParameters = ((ParameterizedType)type).getActualTypeArguments();
    Type idType = typeParameters[0]; // Id has only one parameterized type T
    return Id.get((Class)idType, 0L);
  }
}
```

这个例子特殊的地方是构造函数要把具体的泛型类型作为构造函数的参数，这里可以通过 `ParameterizedType` 类的 `getActualTypeArguments();` 方法读到具体的参数化类型。

#### 直接的打印JSON串还是格式化打印(Compact Vs. Pretty Printing for JSON Output Format)

如果要格式化打印，要使用 `GsonBuilder` 来构建Gson对象，在构造过程中设置 `setPrettyPrinting`。

```
Gson gson = new GsonBuilder().setPrettyPrinting().create();
String jsonOutput = gson.toJson(someObject);
```

#### Null对象支持

通常情况下null的对象是直接被忽略的，但是如果你想在json串中也打印出null，需要配置一下 `serializeNulls`

```
Gson gson = new GsonBuilder().serializeNulls().create();

NOTE: when serializing nulls with Gson, it will add a JsonNull element to the JsonElement structure.  Therefore, this object can be used in custom serialization/deserialization.

Here's an example:

public class Foo {
  private final String s;
  private final int i;

  public Foo() {
    this(null, 5);
  }

  public Foo(String s, int i) {
    this.s = s;
    this.i = i;
  }
}

Gson gson = new GsonBuilder().serializeNulls().create();
Foo foo = new Foo();
String json = gson.toJson(foo);
System.out.println(json);

json = gson.toJson(null);
System.out.println(json);

======== OUTPUT ========
{"s":null,"i":5}
null
```

#### 版本支持

这是一个非常人性化的功能，通过 @Since 注解，来实现不同协议中不同版本的支持。

```
public class VersionedClass {
  @Since(1.1) private final String newerField;
  @Since(1.0) private final String newField;
  private final String field;

  public VersionedClass() {
    this.newerField = "newer";
    this.newField = "new";
    this.field = "old";
  }
}

VersionedClass versionedObject = new VersionedClass();
Gson gson = new GsonBuilder().setVersion(1.0).create();
String jsonOutput = gson.toJson(someObject);
System.out.println(jsonOutput);
System.out.println();

gson = new Gson();
jsonOutput = gson.toJson(someObject);
System.out.println(jsonOutput);

======== OUTPUT ========
{"newField":"new","field":"old"}

{"newerField":"newer","newField":"new","field":"old"}
```

#### 排除不需要序列化的类(Excluding Fields From Serialization and Deserialization)

在Java中可以设置`transient`来标示某个字段不可以序列化， 也可以通过这种方式来设置 

```
import java.lang.reflect.Modifier;

Gson gson = new GsonBuilder()
    .excludeFieldsWithModifiers(Modifier.STATIC)
    .create();

NOTE: you can use any number of the Modifier constants to "excludeFieldsWithModifiers" method.  For example:
Gson gson = new GsonBuilder()
    .excludeFieldsWithModifiers(Modifier.STATIC, Modifier.TRANSIENT, Modifier.VOLATILE)
    .create();
```

再或者还以使用 `@Expose` 注解，注意要设置 `GsonBuilder().excludeFieldsWithoutExposeAnnotation().create()`

如果上面的方法都不能满足的话，还可以通过实现 `ExclusionStrategy` 接口实现自己的排除策略，下面的代码演示了这种功能。

```
@Retention(RetentionPolicy.RUNTIME)
  @Target({ElementType.FIELD})
  public @interface Foo {
    // Field tag only annotation
  }

  public class SampleObjectForTest {
    @Foo private final int annotatedField;
    private final String stringField;
    private final long longField;
    private final Class<?> clazzField;

    public SampleObjectForTest() {
      annotatedField = 5;
      stringField = "someDefaultValue";
      longField = 1234;
    }
  }

  public class MyExclusionStrategy implements ExclusionStrategy {
    private final Class<?> typeToSkip;

    private MyExclusionStrategy(Class<?> typeToSkip) {
      this.typeToSkip = typeToSkip;
    }

    public boolean shouldSkipClass(Class<?> clazz) {
      return (clazz == typeToSkip);
    }

    public boolean shouldSkipField(FieldAttributes f) {
      return f.getAnnotation(Foo.class) != null;
    }
  }

  public static void main(String[] args) {
    Gson gson = new GsonBuilder()
        .setExclusionStrategies(new MyExclusionStrategy(String.class))
        .serializeNulls()
        .create();
    SampleObjectForTest src = new SampleObjectForTest();
    String json = gson.toJson(src);
    System.out.println(json);
  }

======== OUTPUT ========
{"longField":1234}

```


#### JSON 字段命名支持

用法如下

```
private class SomeObject {
  @SerializedName("custom_naming") private final String someField;
  private final String someOtherField;

  public SomeObject(String a, String b) {
    this.someField = a;
    this.someOtherField = b;
  }
}

SomeObject someObject = new SomeObject("first", "second");
Gson gson = new GsonBuilder().setFieldNamingPolicy(FieldNamingPolicy.UPPER_CAMEL_CASE).create();
String jsonRepresentation = gson.toJson(someObject);
System.out.println(jsonRepresentation);

======== OUTPUT ========
{"custom_naming":"first","SomeOtherField":"second"}
```


[**Gson Design Document**](https://sites.google.com/site/gson/gson-design-document)
