---
layout: post
title: "协议相关的东西"
description: ""
category: "知识整理"
tags: []
---
{% include JB/setup %}

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

后来我们还是抛弃了json为了减少带宽json还是太大了，对于友盟的一段普通log，thrift的是json的60%，分别压缩之后（deflate）是70%，也就是说原来的带宽使用可以一下降低到原来的70% 这个优化还是非常可观的。

二进制格式协议好多比如MessagePack、BJSON、ProtoBuffer、Thrift、Avro .. MessagePack的主要缺点是扩展性不好，BJSON其实只是把JSON二进制但是没有做任何优化，PB、Thrift、Avro 产生的协议大小都是一个量级的，但是Avro的依赖文件太多库太大（其实Avro与Thrift、PB的工作模式不一样，后两者依赖于自动生成的文件来读取协议，协议越多产生成文件越大，随着协议膨胀这也会变成一个很大的问题），PB在iOS平台上的库也很大，于是选择了Thrift，Thrift 和 PB都是同一帮人写的，原来在Google工作的时候写了PB后来跑到Facebook搞了Thrift，他们的序列化方式和使用方式都是一样的。

##编码

Thrift 和 PB 把协议变得更小，主要有两点：

1. 把json中的key映射成 1、2、3 这样的数字
2. 对于数值类型编码优化

比如对于上面的协议，PB 定义协议描述文件如下：

```
message root {
	required string name = 1;
	repeated list<string> items = 2;
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
2      | ？

规律自己找吧，小学智力题（答案见最后）

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

生成文件大概(好长的一段代码)

```
/**
 * Autogenerated by Thrift Compiler (@PACKAGE_VERSION@)
 *
 * DO NOT EDIT UNLESS YOU ARE SURE THAT YOU KNOW WHAT YOU ARE DOING
 *  @generated
 */
package com.foo;

import org.apache.thrift.scheme.IScheme;
import org.apache.thrift.scheme.SchemeFactory;
import org.apache.thrift.scheme.StandardScheme;

import org.apache.thrift.scheme.TupleScheme;
import org.apache.thrift.protocol.TTupleProtocol;
import org.apache.thrift.protocol.TProtocolException;
import org.apache.thrift.EncodingUtils;
import org.apache.thrift.TException;
import java.util.List;
import java.util.ArrayList;
import java.util.Map;
import java.util.HashMap;
import java.util.EnumMap;
import java.util.Set;
import java.util.HashSet;
import java.util.EnumSet;
import java.util.Collections;
import java.util.BitSet;
import java.nio.ByteBuffer;
import java.util.Arrays;

public class Log implements org.apache.thrift.TBase<Log, Log._Fields>, java.io.Serializable, Cloneable {
  private static final org.apache.thrift.protocol.TStruct STRUCT_DESC = new org.apache.thrift.protocol.TStruct("Log");

  private static final org.apache.thrift.protocol.TField ID_FIELD_DESC = new org.apache.thrift.protocol.TField("id", org.apache.thrift.protocol.TType.STRING, (short)1);
  private static final org.apache.thrift.protocol.TField TIMESTAMP_FIELD_DESC = new org.apache.thrift.protocol.TField("timestamp", org.apache.thrift.protocol.TType.I64, (short)2);

  private static final Map<Class<? extends IScheme>, SchemeFactory> schemes = new HashMap<Class<? extends IScheme>, SchemeFactory>();
  static {
    schemes.put(StandardScheme.class, new LogStandardSchemeFactory());
    schemes.put(TupleScheme.class, new LogTupleSchemeFactory());
  }

  public String id; // required
  public long timestamp; // optional

  /** The set of fields this struct contains, along with convenience methods for finding and manipulating them. */
  public enum _Fields implements org.apache.thrift.TFieldIdEnum {
    ID((short)1, "id"),
    TIMESTAMP((short)2, "timestamp");

    private static final Map<String, _Fields> byName = new HashMap<String, _Fields>();

    static {
      for (_Fields field : EnumSet.allOf(_Fields.class)) {
        byName.put(field.getFieldName(), field);
      }
    }

    /**
     * Find the _Fields constant that matches fieldId, or null if its not found.
     */
    public static _Fields findByThriftId(int fieldId) {
      switch(fieldId) {
        case 1: // ID
          return ID;
        case 2: // TIMESTAMP
          return TIMESTAMP;
        default:
          return null;
      }
    }

    /**
     * Find the _Fields constant that matches fieldId, throwing an exception
     * if it is not found.
     */
    public static _Fields findByThriftIdOrThrow(int fieldId) {
      _Fields fields = findByThriftId(fieldId);
      if (fields == null) throw new IllegalArgumentException("Field " + fieldId + " doesn't exist!");
      return fields;
    }

    /**
     * Find the _Fields constant that matches name, or null if its not found.
     */
    public static _Fields findByName(String name) {
      return byName.get(name);
    }

    private final short _thriftId;
    private final String _fieldName;

    _Fields(short thriftId, String fieldName) {
      _thriftId = thriftId;
      _fieldName = fieldName;
    }

    public short getThriftFieldId() {
      return _thriftId;
    }

    public String getFieldName() {
      return _fieldName;
    }
  }

  // isset id assignments
  private static final int __TIMESTAMP_ISSET_ID = 0;
  private byte __isset_bitfield = 0;
  private _Fields optionals[] = {_Fields.TIMESTAMP};
  public static final Map<_Fields, org.apache.thrift.meta_data.FieldMetaData> metaDataMap;
  static {
    Map<_Fields, org.apache.thrift.meta_data.FieldMetaData> tmpMap = new EnumMap<_Fields, org.apache.thrift.meta_data.FieldMetaData>(_Fields.class);
    tmpMap.put(_Fields.ID, new org.apache.thrift.meta_data.FieldMetaData("id", org.apache.thrift.TFieldRequirementType.REQUIRED, 
        new org.apache.thrift.meta_data.FieldValueMetaData(org.apache.thrift.protocol.TType.STRING)));
    tmpMap.put(_Fields.TIMESTAMP, new org.apache.thrift.meta_data.FieldMetaData("timestamp", org.apache.thrift.TFieldRequirementType.OPTIONAL, 
        new org.apache.thrift.meta_data.FieldValueMetaData(org.apache.thrift.protocol.TType.I64)));
    metaDataMap = Collections.unmodifiableMap(tmpMap);
    org.apache.thrift.meta_data.FieldMetaData.addStructMetaDataMap(Log.class, metaDataMap);
  }

  public Log() {
  }

  public Log(
    String id)
  {
    this();
    this.id = id;
  }

  /**
   * Performs a deep copy on <i>other</i>.
   */
  public Log(Log other) {
    __isset_bitfield = other.__isset_bitfield;
    if (other.isSetId()) {
      this.id = other.id;
    }
    this.timestamp = other.timestamp;
  }

  public Log deepCopy() {
    return new Log(this);
  }

  @Override
  public void clear() {
    this.id = null;
    setTimestampIsSet(false);
    this.timestamp = 0;
  }

  public String getId() {
    return this.id;
  }

  public Log setId(String id) {
    this.id = id;
    return this;
  }

  public void unsetId() {
    this.id = null;
  }

  /** Returns true if field id is set (has been assigned a value) and false otherwise */
  public boolean isSetId() {
    return this.id != null;
  }

  public void setIdIsSet(boolean value) {
    if (!value) {
      this.id = null;
    }
  }

  public long getTimestamp() {
    return this.timestamp;
  }

  public Log setTimestamp(long timestamp) {
    this.timestamp = timestamp;
    setTimestampIsSet(true);
    return this;
  }

  public void unsetTimestamp() {
    __isset_bitfield = EncodingUtils.clearBit(__isset_bitfield, __TIMESTAMP_ISSET_ID);
  }

  /** Returns true if field timestamp is set (has been assigned a value) and false otherwise */
  public boolean isSetTimestamp() {
    return EncodingUtils.testBit(__isset_bitfield, __TIMESTAMP_ISSET_ID);
  }

  public void setTimestampIsSet(boolean value) {
    __isset_bitfield = EncodingUtils.setBit(__isset_bitfield, __TIMESTAMP_ISSET_ID, value);
  }

  public _Fields fieldForId(int fieldId) {
    return _Fields.findByThriftId(fieldId);
  }

  public void read(org.apache.thrift.protocol.TProtocol iprot) throws org.apache.thrift.TException {
    schemes.get(iprot.getScheme()).getScheme().read(iprot, this);
  }

  public void write(org.apache.thrift.protocol.TProtocol oprot) throws org.apache.thrift.TException {
    schemes.get(oprot.getScheme()).getScheme().write(oprot, this);
  }

  @Override
  public String toString() {
    StringBuilder sb = new StringBuilder("Log(");
    boolean first = true;

    sb.append("id:");
    if (this.id == null) {
      sb.append("null");
    } else {
      sb.append(this.id);
    }
    first = false;
    if (isSetTimestamp()) {
      if (!first) sb.append(", ");
      sb.append("timestamp:");
      sb.append(this.timestamp);
      first = false;
    }
    sb.append(")");
    return sb.toString();
  }

  public void validate() throws org.apache.thrift.TException {
    // check for required fields
    if (id == null) {
      throw new org.apache.thrift.protocol.TProtocolException("Required field 'id' was not present! Struct: " + toString());
    }
    // check for sub-struct validity
  }

  private void writeObject(java.io.ObjectOutputStream out) throws java.io.IOException {
    try {
      write(new org.apache.thrift.protocol.TCompactProtocol(new org.apache.thrift.transport.TIOStreamTransport(out)));
    } catch (org.apache.thrift.TException te) {
      throw new java.io.IOException(te.getMessage());
    }
  }

  private void readObject(java.io.ObjectInputStream in) throws java.io.IOException, ClassNotFoundException {
    try {
      // it doesn't seem like you should have to do this, but java serialization is wacky, and doesn't call the default constructor.
      __isset_bitfield = 0;
      read(new org.apache.thrift.protocol.TCompactProtocol(new org.apache.thrift.transport.TIOStreamTransport(in)));
    } catch (org.apache.thrift.TException te) {
      throw new java.io.IOException(te.getMessage());
    }
  }

  private static class LogStandardSchemeFactory implements SchemeFactory {
    public LogStandardScheme getScheme() {
      return new LogStandardScheme();
    }
  }

  private static class LogStandardScheme extends StandardScheme<Log> {

    public void read(org.apache.thrift.protocol.TProtocol iprot, Log struct) throws org.apache.thrift.TException {
      org.apache.thrift.protocol.TField schemeField;
      iprot.readStructBegin();
      while (true)
      {
        schemeField = iprot.readFieldBegin();
        if (schemeField.type == org.apache.thrift.protocol.TType.STOP) { 
          break;
        }
        switch (schemeField.id) {
          case 1: // ID
            if (schemeField.type == org.apache.thrift.protocol.TType.STRING) {
              struct.id = iprot.readString();
              struct.setIdIsSet(true);
            } else { 
              org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
            }
            break;
          case 2: // TIMESTAMP
            if (schemeField.type == org.apache.thrift.protocol.TType.I64) {
              struct.timestamp = iprot.readI64();
              struct.setTimestampIsSet(true);
            } else { 
              org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
            }
            break;
          default:
            org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
        }
        iprot.readFieldEnd();
      }
      iprot.readStructEnd();

      // check for required fields of primitive type, which can't be checked in the validate method
      struct.validate();
    }

    public void write(org.apache.thrift.protocol.TProtocol oprot, Log struct) throws org.apache.thrift.TException {
      struct.validate();

      oprot.writeStructBegin(STRUCT_DESC);
      if (struct.id != null) {
        oprot.writeFieldBegin(ID_FIELD_DESC);
        oprot.writeString(struct.id);
        oprot.writeFieldEnd();
      }
      if (struct.isSetTimestamp()) {
        oprot.writeFieldBegin(TIMESTAMP_FIELD_DESC);
        oprot.writeI64(struct.timestamp);
        oprot.writeFieldEnd();
      }
      oprot.writeFieldStop();
      oprot.writeStructEnd();
    }

  }

  private static class LogTupleSchemeFactory implements SchemeFactory {
    public LogTupleScheme getScheme() {
      return new LogTupleScheme();
    }
  }

  private static class LogTupleScheme extends TupleScheme<Log> {

    @Override
    public void write(org.apache.thrift.protocol.TProtocol prot, Log struct) throws org.apache.thrift.TException {
      TTupleProtocol oprot = (TTupleProtocol) prot;
      oprot.writeString(struct.id);
      BitSet optionals = new BitSet();
      if (struct.isSetTimestamp()) {
        optionals.set(0);
      }
      oprot.writeBitSet(optionals, 1);
      if (struct.isSetTimestamp()) {
        oprot.writeI64(struct.timestamp);
      }
    }

    @Override
    public void read(org.apache.thrift.protocol.TProtocol prot, Log struct) throws org.apache.thrift.TException {
      TTupleProtocol iprot = (TTupleProtocol) prot;
      struct.id = iprot.readString();
      struct.setIdIsSet(true);
      BitSet incoming = iprot.readBitSet(1);
      if (incoming.get(0)) {
        struct.timestamp = iprot.readI64();
        struct.setTimestampIsSet(true);
      }
    }
  }

}
```

在一个最简单的Helloworld工程中，需要把上面的生成文件和基础库都添加到工程中，操作协议的例子大概如下吧：

```
Log log = new Log();
log.setId("123");
log.setTimestamp(456);
byte[] data = (new TSerializer()).serialize( log ); //序列化成byte array

Log log2 = new Log();
//解析字节,
log2 = new TDeserializer(new TCompactProtocol.Factory()).deserialize(response, data);
```

这样已经完成了一次最简单的Thrift之旅。

##Thrift 编译器

这个工具主要用来根据协议描述文件生成代码，目前为止已经支持了很多语言，但是java语言版本是针对服务器的，生成代码中包含一些移动平台不需要的依赖，并且代码量太大包含需要冗余的方法，所以可能需要自己修改一下生成代码的逻辑。这里记录一下编译Thrift编译器源码时候遇到的一些坑，需求如下（windows平台使用VS编译）：

1. 添加Android支持
2. 删除重复功能的和用不到的方法


按照 `README_Windows.txt`（下载Thrift Compiler源码之后根目录下有这个文件） 内容下载 `Flex` 和 `Bison` 并生成相关代码，
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