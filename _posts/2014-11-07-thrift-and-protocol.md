# Thrift

关于协议最开始接触的xml、后来是json，这些都是基于文本的，xml 长得像
这样

```
<root> 
	<name> foo </name>
	<list>
		<item>A</item>
		<item>B</item>
	</list>
</root>
```

xml 比较方面人工编辑和阅读但是作为传输协议会使用时会很大的带宽，如果和json比较一下

```
{
	root:
		{
			name:"foo", 
			list:["A","B"]
		} 
}
```
就会发现同样的信息使用json协议来表述会很小，所以现在很多移动平台的开发基本都是使用json作为传输协议。而且因为使用文本格式容易调试，这点在看到后面的Thrift和ProtoBuffer的时候就会感觉到了，这些二进制格式的协议都是不可读的，debug的时候非常麻烦。

后来我们还是抛弃了json为了减少带宽json还是太大了，对于友盟的一段普通log，如果json为1K，上面提到的二进制格式大约在0.6K 压缩（deflate）之后大约在 0.7K， 也就是说原来的带宽使用可以一下降低到原来的70% 这个优化还是非常可观的。

二进制格式协议好多比如MessagePack、BJSON、ProtoBuffer、Thrift、Avro .. MessagePack的主要缺点是扩展性不好，BJSON其实只是把JSON二进制但是没有做任何优化，PB、Thrift、Avro 产生的协议大小都是一个量级的，但是Avro的依赖文件太多库太大（其实Avro与Thrift、PB的工作模式不一样，后两者依赖于自动生成的文件来读取协议，协议越多产生成文件越大，随着协议膨胀这也会变成一个很大的问题），PB在iOS平台上的库也很大，于是选择了Thrift，Thrift 和 PB都是同一帮人写的，原来在Google工作的时候写了PB后来跑到Facebook搞了Thrift，他们的序列化方式和使用方式都是一样的。

##编码

Thrift 和 PB 把协议变得更小，主要有两点：

1. 把json中的key映射成 1、2、3 这样的数字
2. 对于数值类型编码优化

比如对于上面的协议，PB 定义协议描述文件如下：

```
message root {
	required string name = 1;
	optional list<string> items = 2;
}
```
这样原来必须在协议中传输的key（eg：name、list）在传输时候转变成 1、2，对于原来很长的key 比如 device_manuid device_manufacturer 这样很长的key优化效果很好。

其次就是对数值类型的优化，比如 300 会编码成：`1010 1100 0000 0010` 原理是这样的，每个byte的8位的最高位作为标志位，剩下7位用来存储数据，如果最高位是1表示下一个byte还有数据，0表示没有数据了，所以上面的编码这样解析的：

```
000 0010  010 1100
→  000 0010 ++ 010 1100
→  100101100
→  256 + 32 + 8 + 4 = 300
```

另外PB是携带value的类型信息的，类型信息会和key编码在一起，占用低三位（因为PB支持有限的类型共6中，使用3位可以存储最多8种类型信息已经够了）比如：`000 1000` 最后三位时类型信息，去除最高位和末三位，所以实际值为 `000 1` 就是1. 所以这里面存在一个优化技巧，如果协议描述文件中定义的key值小于16那么这key正好可以编码到一个byte里面，这样是节省空间的。PB的编码150

```
96 01 = 1001 0110  0000 0001
       → 000 0001  ++  001 0110 (drop the msb and reverse the groups of 7 bits)
       → 10010110
       → 2 + 4 + 16 + 128 = 150
```

处理有符号类型（负数情况），正常情况下一个int型的最高位是标志位，如果直接编码一个负数，那么会占用很长的字节，它实际上可以看成一个很大的无符号数。比如 `1000 0011` 既可以看成负数 -3 ，也可以看成无符号数 2^7+3, 这样编码的结果很长，PB使用 “zig-zags” 的方式编码，其实只是把正负数交叉重新排列了一下：

Signed | Encoded
-------|-------
0	   | 0
-1     | 1
1      | 2
-2     | 3
2	   | ？

规律自己找吧，小数智力题（答案见最后）

对应的算法就是 `(n << 1) ^ (n >> 31)` 

string是比较特殊的类型，它使用这样的方式编码：length + string

```
12 07 74 65 73 74 69 6e 67
```
12 是key的编码，07 是字符串长度， `74 65 73 74 69 6e 67` 是字符串值 "testing"

message 类型嵌套使用类似策略。

这里面的大部分文字都来自：https://developers.google.com/protocol-buffers/docs/encoding
原文写的更全面些，这里主要想说明PB或者Thrift编码之后的协议为什么比JSON小很多，但是使用这些编码方式生成的日志都是不可读的调试的时候比较麻烦，比如抓包的时候还需要再写单独的工具来解析。

其实Thrift是一个比较庞大的库，主要用来做服务通信的，协议编码只是其中一个模块而且设计之初就没有指定如何编码接口是抽象的，支持很多种编码方式比如json、binary、compact（和PB一样的方式）等。相反PB更接近于一个类似json的协议，PB中的B时buffer的意思，它在编码协议数据的时候会再内存中建立一个缓存区，控制内存占用，我再看到这部分源码的时候给友盟的统计分析添加了内存buffer功能。

##工作模式

Thrift 和 PB 都是通过协议描述文件生成代码 + 基本库的方式工作的，基本库提供基础的编码功能，生成代码用来编码或者解析，因为协议描述文件中有key的映射信息，这些信息会用来生成读写协议的代码。生成代码是使用PB或者Thrift提供的一个工具，Thrift的就非常简单，到[官网](https://thrift.apache.org/)下载一个工具就行了（用Thrift举例因为我现在在用）

1. 下载Thrift基础库和协议编译器
2. 编辑协议描述文件并生成代码
3. 调用代码来操作协议的读写

Thrift的描述文件大约就是这样子的

```
//log.thrift
struct Log {
	1: required string id;
	2: optional int64 timestamp;
}
```

这样就可以定义一个结构包含两个字段 id 和 timestamp 前者是string类型后者是 int64 类型，如果使用Thrift编译器编译这个协议在java语言中会生成一个`Log.java`的类,它会提供setId、getId、setTimestamp、getTimestamp 之类的方法。

编译：`thrift --gen <language> <Thrift filename>` 比如，

```
thrift --gen java log.thrift
```

生成文件大概：

在一个最简单的Helloworld工程中，需要把上面的生成文件和基础库都添加到工程中，操作协议的例子大概如下吧：

```
```

这样已经完成了一次最简单的Thrift之旅。

##Thrift 编译器

这个工具主要用来根据协议描述文件生成代码，目前为止已经支持了很多语言，但是java语言版本是针对服务器的，生成代码中包含一些移动平台不需要的依赖，并且代码量太大包含需要冗余的方法，所以可能需要自己修改一下生成代码的逻辑。这里记录一下编译Thrift编译器源码时候遇到的一些坑，需求如下（windows平台使用VS编译）：

1. 添加Android支持
2. 删除重复功能的和用不到的方法


按照 `README_Windows.txt` 内容下载 `Flex` 和 `Bison` 并生成相关代码，
上面工程中已经做好了这一步，可以直接到下一步。

######添加 Android 支持并删除冗余代码

1. 添加新的 generator(代码生成器) `t_android_generator.cc`
2. 修改 `android_legacy_` 永远为 True,以为在 Android 2.3 以下版本中没有 IOException(Exception e) 实现
3. 注释代码段：
```
  //generate_generic_field_getters_setters(out, tstruct);
  //generate_generic_isset_method(out, tstruct);

  //generate_java_struct_equality(out, tstruct);
  //generate_java_struct_compare_to(out, tstruct);
```
删除了如下函数实现：
```
 public void setFieldValue(_Fields field, Object value) 
 public Object getFieldValue(XXXField)
 public boolean isSet(XXXXFiled)
 public boolean equals(Object that) 
 public boolean equals(Work that)
 public int compareTo(Work other)
```

4. 修改生成文件夹目录 `out_dir_base_ = (bean_style_ ? "gen-javabean" : "gen-android")`
5. 删除对 `"import org.slf4j.Logger;\n" ` 和 ` "import org.slf4j.LoggerFactory;\n\n";` 的包引用
6. 修改预编译提示 `THRIFT_REGISTER_GENERATOR(android, "Android" ...`

######编译协议描述文件

```
thrift --gen android thrift.file
```

如果不使用上面的方法，那么在Android平台上需要添加Java5支持并且手动删除一些代码

```
thrift --gen java:java5 thrift.file
```



答：4
