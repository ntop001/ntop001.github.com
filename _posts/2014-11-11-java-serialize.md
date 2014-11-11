

有木有这样的需求，需要把一些数据保存在本地文件，但是这些数据本身是结构化的，所以保存的时候不是简单的写文件还需要有一定格式
这样下次还可以把这些结构化的数据读出来。`java.util.Properties` 可以实现这样的需求,把一对键值保存在文件中，再读出来。

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
但是 Properties 类只支持String类型的key-value 如果需要保存的数据很多而且有嵌套结构保存的过程就会变得异常复杂，这时候
大多人都会考虑java的序列化功能了。序列化功能可以把一个类序列化为一个byte[] 这样就不需要逐个保存类中的每个字段了。实现
系列化功能非常简单，只需要实现 Serializable 接口。

```
class SeriaTest implements Serializable {
	String a = "a";
	int b = 2;
	byte[] c = {3,4,5};
}
```
编译这段代码会报一个很重要的warning 







