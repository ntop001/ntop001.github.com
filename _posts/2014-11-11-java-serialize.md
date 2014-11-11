

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


