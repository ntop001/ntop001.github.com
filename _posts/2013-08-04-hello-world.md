---
layout: post
title: "Hello world"
description: ""
category: 
tags: []
---
{% include JB/setup %}

This is a **Hello world** , welcome .

## 使用 GCC 编译Helloworld

vi hello.c

```
#include<stdio.h>

int main(){
  printf("Helloworld");
  return 0;
}
```

编译成可执行文件并执行：gcc hello.c -o hello && ./hello

##  添加头文件

vi hello.h

```
void printHello();
```

vi hello.c

```
#include<stdio.h>
#include"hello.h"
int main(){
  printHello();
  return 0;
}
```

`#include<xxx.h>` gcc 会到系统的lib库下寻找对应的 xxx.h 文件<br>
`#include"xxx.h"` gcc 只会在本地寻找对应的 xxx.h 文件

## 编译多个文件

对于编译器来说，编译器会按照如下步骤编译源代码：
1. 预处理 把头文件的信息添加到`.c`文件 和一些预编译指令
2. 汇编 把单独的 `.c` 文件分别翻译成汇编代码
3. 转化成二进制的机器代码 （Object Code 所谓的 `.o` 文件）
4. 链接 把不同的代码碎片链接起来（把方法声明和方法实现建立引用关系） 

vi encrypt.h
```
void encrypt(char *message);
```

vi encrypt.c

```
void encrypt(char *message)
{
	char c;
	while(*message)
	{
		*message = *message ^ 31;
		message ++ ;
	}
}
}
```

vi message_hider.c

```
#include <stdio.h>
#include "encrypt.h"

int main()
{
	char msg[80];
	while (fgets(msg,80,stdin))
	{
		encrypt(msg);
		printf("%s", msg);
	}
}
```

编译： gcc message_hider.c encrypt.c -o message_hider

首先，编译器会分别编译 message_hider.c 和 encrypt.c 文件
然后，在链接阶段，编译会把在 message_hider.c 中对 encrypt(msg) 函数的引用链接到真实的在 encrypt.c 中的实现。
（估计保存了encrypt 函数的偏移地址）。 链接过程是通过查找方法名实现的，所以经常会有函数命名冲突的问题~

## 编译更多的文件

如果你有更多的`.c`文件，比如几千个，编译这这些文件会是非常耗时的操作，如果每次只是修改了一个文件的几行代码
就要重新编译所有文件这是不可接受的。通过上面对编译器步骤的了解，在编译器工作的 1，2，3阶段，是对每个文件单独
进行的（生成一个.o文件）。所以只要重新编译改动的 .c 文件，再链接结果是一样的。

所以更快的编译需要如下条件：

1. 保存编译出来的 .o 中间文件 
2. 重新编译改动了的 .c 文件到 .o 文件
3. 链接所有 .o 文件

编译出.o 文件： gcc -c *.c 这条命令告诉编译器，把当前目录下所有的 .c 文件都编译成 .o 文件，但是记着不要链接
链接所有 .o 文件: gcc *o -o xxx 这条命令告诉编译器，把目录下的所有 .o 文件链接到一起生成可执行文件

但是如果这些文件之间存在一些相互依赖，比如 a.o 依赖于 a.c 如果 a.c 变化了那么 a.o 也需要重新编译，那么手动完成上
面的部分编译依然是件很麻烦的事情，我们需要更自动化的方式。于是 make 诞生了。

make 基于一个很简单的规则：a.c 和 a.o ， 如果 a.c 变化了,那么 a.c 的最后修改日期一定比 a.c 新，所以 make 只要知道这些文件的依赖关系，就可以很容易的
的做到部分编译(依赖关系可能更为复杂)。保存依赖关系的文件叫 makefile。

make 规则： 1. 依赖关系 2. 处理依赖的指令（TAB开始一行）

对于上面的示例可以使用下面的make文件：
vi Makefile
```
encrypt.o: encrypt.c encrypt.h
	gcc -c encrypt.c
message_hider.o: message_hider.c encrypt.h
	gcc -c message_hider.c
message_hider: message_hider.o encrypt.o
	gcc message_hider.o encrypt.o -o message_hider
```

编译： make message_hider


## 结构和联合(Struct & Union)

结构体是类似这样的数据类型定义：

```
struct fish 
{
	const char*name;
	int length;
	int age;	
};
```
使用： 
```
struct fish tuna = {"tom", 10, 2}; 
printf("Fish name : %s ,length: %d, age: %d" ,tuna.name, tuna.length, tuna.age);
```

每次使用 `struct fish` 比较麻烦，其实可以像其他高级语言中定义类一样定义如下：

```
typedef struct fish 
{
	const char*name;
	int length;
	int age;	
}fish;
```
但是Struct 和 C 中的大部分基本类型一样是值传递，在函数中引用需要传递指针。

```
void foo(fish *tuna)
{
	printf("Fish name: %s", tuna->name);
}
```

注意: tuna->name 和 (*tuna).name 是一样的，表示， t 指向 name 的指针

用同一个变量表示不同的数据类型，在`JAVA`中这是一个很难完成的任务，你可能需要定义一个专门的类(Class)来表示这个数据类型，但是在 
C 中有专门的表示方式—— 联合(Union). 联合是可以用来在同一个结构中表示不同数据类型，比如：

```
typedef union 
{ 
	short count; 
	float weight; 
	float volume;
} quantity;
``` 

初始化可以： quantity q = {3};//默认初始化第一个 或者 q = {.weight=2.3f} (Struct 也可以使用这种方式赋值); 或者 q.volume = 2.0f;

## 枚举和比特位（BitField）

C 中枚举使用和struct一样：

```
typedef enum color 
{
	RED,GREEN,BLUE
}COLOR;
```

比特位可以用来精确控制内存使用，可以给每个struct的字段指定需要的存储空间比如：

```
typedef struct year
{
	unsigned int month:12; 12 < 2^4
	unsigned int day:31; 31 < 2^5
} year;
```
使用4个比特位存储月份，5个比特位存储天数。

## 动态内存申请

动态内存申请是在堆（Heap）上完成的，C 语言中的内存申请用 `malloc()` 函数，需要引入 `stdlib.h` 这个库。

```
fish *f = malloc(sizeof(fish));//申请内存

printf("Fish name is %s", f.name);

free(f);//释放内存
```

内存申请的时候常见的错误是在赋值前忘记释放内存，比如下面这段代码：

```
char *_abc = "abcd";
char *_123 = "1234";
char *str1 = strdup(_abc); // 没有释放对 "1234" 的拷贝
char *str1 = strdup(_123);
free(str1);
```

## 函数指针

声明：Return type(* Pointer variable)(Param types)

```
int add(int a, int b );

int main()
{
	// 函数指针说明
	int (* add_dup)(int,int);
	add_dup = add;

	printf("2 + 3 = %d", add_dup(2,3));
}

int add(int a, int b)
{
	return (a + b);	
}
```

函数指针有什么用处呢？ 如果在其他语言比如java中使用过接口，C#的委托，就会发现函数指针是类似的东西。
函数指针指向的是函数的地址，正常声明的函数名其实就是函数的内存地址。比如下面实现的排序函数：

```
void qsort(void *array, size_t length, size_t item_size, int (*compare)(const *void,const *void))
{
	void *a;
	void *b;
	...
	if( compare(a,b))
	...
} 

int compare(const void* a, const void *b)
{
	int a = *(int*)a;
	int b = *(int*)b;
	return (a - b);
}

int main()
{
	int array[] = {1,2,7,5,8,3,4};

	qsort(array, 7, sizeof(int), compare);
}
```

## 链接库

一般在Linux系统上 #include<XXX.h> ,编译器会在如下的两个目录下寻找头文件：

* /usr/local/include // for third-party lib's header files
* /usr/include       // for system header files

如果使用的是 MinGW 会在 C:\MinGW\inlcude\ 下寻找. #inclucde"xxx.h"会在当前目录下寻找头文件

同级的 lib 目录下会存放链接库文件 `.a` `.so`

*/usr/local/lib/
*/usr/lib/

那么如何使用自己编译出来的 .h 和 .o 文件 ？ 最简单直接的方法是使用编译命令,告诉头文件和.o 文件的全路径

```
gcc -I/my_header_files test_code.c /my_object_files/encrypt.o /my_object_files/checksum.o -o test_code
```

但是如果把所有的 `.o` 文件打包成 `.a` 文件(静态链接库)再使用(像系统做的那样)，其实可以做的更好。假如已经准备好了.a 文件，
分别把`.h` 和 `.a` 拷贝到上面的系统目录下，然后使用 `-l`命令告诉编译器寻找名为 `xxxx.a` 的静态链接库。

```
#inlcude<xxx.h>
...

gcc test_code.c -lxxxx -o test_code 
```

如果把 .h 文件 和 .a 文件分别呢放在了自有目录下，也可以使用编译命令：

```
gcc -I/my_header test_code.c -L/my_lib -lhfsecurity -o test_code
```

那么如何创建一个 `.a` 文件呢？使用 `ar` 命令，ar 是 archive command 的意思。

```
//The r means the .a file will be updated if it already exists
//The c means that the archive will be created without any feedback.
//The s tells ar to create an index at the start of the .a file.
ar -rcs libxxxx.a xxxx1.o xxxx2.o //
```
如何查看`.a`文件的内容？ 使用 `nm` 命令，

```
> nm libl.a
libl.a(libmain.o): 
00000000000003a8 s EH_frame0
				 U _exit 
0000000000000000 T _main     // Text 表示这是一个函数
00000000000003c0 S _main.eh 
				 U _yylex

libl.a(libyywrap.o): 
0000000000000350 s EH_frame0 
0000000000000000 T _yywrap 
0000000000000368 S _yywrap.eh >
``` 

更多的时候需要使用动态链接库(Linux 上.so 结尾的文件)，创建动态链接库有点小不同，首先是创建位置独立的 .o 文件

```
//-fPIC: This tells gcc that you want to create position-independent code
//Position-independent code can be moved around in memory.
gcc -I/includes -fPIC -c xxxx.c -o xxx.o
gcc -shared xxxx.o -o xxx.so // 使用 -shared 生成动态链接库
gcc *.c -lxxx -o xxx         // 像使用静态链接库一样使用动态链接库
```

虽然动态链接库使用的时候和静态链接库一样，但是使用动态链接库编译程序的时候，生成的可以执行文件中并没有包含链接库的代码，而是放了一些
占位符(PlaceHolder)，真正运行程序的时候，才会动态的加载.so 文件，并执行其中的代码，所以可以可以随时替换动态链接库文件，而不需要重新
编译程序。

## 进程与系统调用

系统调用可以直接调用系统的程序，比如下面的程序（MAC）:
vi sys.c
```
int main()
{
	system("say 'Something changes , something never'");
}
```
运行：gcc sys.c && ./a.out  会读出文本内容， system() 是一个系统调用。

和调用 printf() 函数不同的是，printf 函数的实现最终会被编译到文件中，但是 system() 函数不会，system 在调用的时候会到系统内核中
寻找实现。系统调用是生存在操作系统内核中的函数。[**这里**](http://www.ibm.com/developerworks/cn/linux/kernel/syscall/part1/appendix.html)
是网上有人总结的常见系统调用。

> ##What’s the kernel?##
> On most machines, system calls are functions that live inside the kernel of the operating system. But what is the kernel? You never actually see the kernel on the screen, but it’s always there, controlling your computer. The kernel is the most important program on your computer, and it’s in charge of three things:

> ###Processes###
>No program can run on the system without the kernel loading it into memory. The kernel creates processes and makes sure they get the resources they need. The kernel also watches for processes that become too greedy or crash.
> ###Memory###
Your machine has a limited supply of memory, so the kernel has to carefully ration the amount of memory each process can take. The kernel can increase the virtual memory size by quietly loading and unloading sections of memory to disk.
> ###Hardware###
> The kernel uses device drivers to talk to the equipment that’s plugged into the computer. Your program can use the keyboard and the screen and the graphics processor without knowing too much about them, because the kernel talks to them on your behalf.
> <br>System calls are the functions that your program uses to talk to the kernel.

总而言之，系统调用是程序和内核对话的通道。


C语言的进程在Linux平台的实现是通过 fork() 系统调用完成的。fork 函数执行后会复制进程，并且和父进程运行同一段程序（听起来有点绕 ？ 也就是在fork（） 函数执行的瞬间，当前进程被copy了一份一模一样的，这时候会有两个同样的进程在运行同样的代码）。fork 函数会返回一个数值，来告诉当前所处的进程, 如果是在子进程会返回 0 , 如果不是会返回一个非零值。 

好吧先看看下面这段代码打印什么 ？

vi fork.c

```
#include<unistd.h>
#include<sys/types.h>
#include<stdio.h>

int main()
{
	pid_t pid;
	pid = fork();

	printf("Hello process\n");
}
```
编译运行： gcc fork.c && ./a.out

结果：

```
Hello process
Hello process
```

为什么会打印两个 'Hello process' 呢 ？ 因为在执行 fork 时，程序被复制了一份（影分身之术），各自运行分别打印了自己的 'Hello process'。那么如何区分这个两个进程
究竟是谁fork了谁？ 可以通过 pid 来判断，pid 在子进程中是 0 ，在父进程中是 -1(出错) 或者 子进程的 pid（成功复制）。通过判断 pid 就可以给进程分配工作。

```
➜  system  cat fork.c 
#include<unistd.h>
#include<sys/types.h>
#include<stdio.h>

int main()
{
	pid_t pid;
	pid = fork();

	if(!pid){
		printf("Hello, I'm child  process-%d \n",getpid());
	}else{
		printf("Hello, I'm father  process-%d \n",getpid());
	}
}
➜  system  vi fork.c 
➜  system  gcc fork.c && ./a.out
Hello, I'm father  process-7486 
Hello, I'm child  process-7488 
```
因为是完全复制的进程，所他们执行相同的代码，只是通过pid区分了需要执行的代码段。
fork 并不会真的每次都完全复制整个进程，他会使用一种叫做 “copy on write” 的技术，先共享着同份内存，只有在内存发生变化的时候才会真正的copy内存。



## 进程通信

一个进程一般包含：代码，全局变量，常量，堆，栈等，但是有一些更重要的东西往往会被忽略——文件描述表（Descriptor Table）, 它是程序和外部通信的大门。

 file descriptor | Data Stream
 |:---:|:-----:|
 0 | The keyboard    // std input
￼1 | The screen      // std output
￼2 | The screen      // std error
￼3 | Database connection

数字表示文件描述符，后面的映射是各种流~

一些常用的进程操作函数：

1. eixt() 可以用来立刻结束进程
2. fileno() 获取当年数据流的文件描述符
3. dup2(a, b) 把 b 重定向到 a
4. waitpid(pid,pid_status,options) 等待某个进程结束 
5. pipe() 链接两个进程的标准输入和输出

进程运行时，经常会接收到一些来自操作系统的信号，比如按下 Ctr + C ， 就会结束当前的进程。这些信号只是一些int型的数值，每个数值对应一个系统函数来处理对应的信号.

| Signal | Handler|
|:----|:----|
|SIGURG |Do nothing|
|SIGINT |Call exit()|

有了信号表(signal table)就可以在进程接收到某个信号的时候，执行我们自己定义的函数，来替换系统的默认操作，C 语言中使用
`sigactions ` 来完成替换（或者也可以说是：信号捕获），sigactions 会告诉操作系统进程在接收信号后要执行那个函数。

```
void diediedie(int sig); 			//处理信号的函数， sig 是需要处理的信号
....
int catchsignal()
{
	struct sigaction action;		//Create a new action 
	action.sa_handler = diediedie;  //This is the name of the function you want the computer to call.
	sigemptyset(&action.sa_mask);   //The mask is a way of filtering the signals that the sigaction will handle
	action.sa_flags = 0;			//These are some additional flags. You can just set them to zero.
									//Tell the operating system about it
	return sigaction(SIGINT, &sigaction, NULL); 
}
```

下面这段代码会捕获 Ctr+C 的操作：

vi signal.c
```
#include <stdio.h> 
#include <signal.h> 
#include <stdlib.h>

void diediedie(int sig) 
{
	puts ("Goodbye cruel world....\n");
	exit(1); 
}

int catch_signal(int sig, void (*handler)(int)) 
{
	struct sigaction action; 
	action.sa_handler = handler; 
	sigemptyset(&action.sa_mask);
	 action.sa_flags = 0;
	return sigaction (sig, &action, NULL);
 }

int main() 
{
	if (catch_signal(SIGINT, diediedie) == -1)
	 { 
		fprintf(stderr, "Can't map the handler"); 
		exit(2);
	}
	char name[30]; 
	printf("Enter your name: "); 
	fgets(name, 30, stdin);
	printf("Hello %s\n", name); 
	return 0;
}
```
编译运行： gcc signal.c && ./a.out

信号表：

|Signal| Action |
|:---|:---|
|SIGINT|The process was interrupted|
|SIGQUIT|Someone asked the process to stop and dump the memory in a core dump file.|
|SIGFPE|Floating-point error|
|SIGTRAP|The debugger asks where the process is|
|SIGSEGV|The process tried to access illegal memory|
|SIGWINCH|The terminal window changed size|
|SIGTERM|Someone just asked the kernel to kill the process|
|SIGPIPE|The process wrote to a pipe that nothing’s reading|

我们经常使用 `KILL` 命令来杀死进程，但是事实上 kill 只是一个用来向进程发送信号的工具。比如：`kill -INT pid`.
程序也可以用 `raise(int)`来向自己发送信号，

## 网络交互 

C 语言中的网络交互是通过 Socket 来实现的， Socket 是一种特殊的数据流（two-way）,既可以写也可以读，和之前的所有数据流一样他也有一个文件描述符。
服务端程序使用的Socket需要如下几个步骤(BLAB)：

1. Bind 把一个文件描述符绑定到计算机的某个端口（>1024）
2. Listern 监听端口的数据
3. Accept 接受对话请求
4. Begin 开始对话

下面一个简单运行在本机的Server端程序示例：

vi socket.c
```
#include<stdio.h>
#include<string.h>
#include<errno.h>
#include<stdlib.h>
#include<sys/socket.h>
#include<arpa/inet.h>
#include<unistd.h>
#include<signal.h>


void error(char *msg);   		
void bind_to_port(int socket, int port);
int open_listener_socket();
int say(int socket, char *s);
int read_in(int socket, char *buf, int len);
int catch_signal(int sig, void (*handler)(int));
void handle_shutdown(int sig);

int listener_d;

int main()
{
	if(catch_signal(SIGINT, handle_shutdown) == -1) 
		error("Can't set the interrupt handler");
	listener_d = open_listener_socket();
	bind_to_port(listener_d, 3000);
	if(listen(listener_d, 10) == -1)
		error("Can't listen");
	
	struct sockaddr_storage client_addr;
	unsigned int address_size = sizeof(client_addr);
	puts("Waiting for connection");
	char buf[255];
	
	while(1)
	{
		int connect_d = accept(listener_d, (struct sockaddr *)&client_addr, &address_size);
		if( connect_d == -1)
			error("Can't open secondary socket");
		if( say(connect_d, 
		"Internet KK Protocol Server\r\n V1.0\r\nKnock!Knock!\r\n"
			) != -1 )
		{
			read_in( connect_d, buf, sizeof(buf));
			say(connect_d, "I'm a response!");
		}
		close(connect_d);
	}
	return 0;
}

void error(char *msg)
{
	fprintf(stderr, "%s: %s\n",msg, strerror(errno));
	exit(1);
}

void bind_to_port(int socket, int port)
{
	struct sockaddr_in name;
	name.sin_family = PF_INET;
	name.sin_port = (in_port_t)htons(3000);
	name.sin_addr.s_addr = htonl(INADDR_ANY);
	
	int reuse = 1;
	if(setsockopt(socket, SOL_SOCKET,SO_REUSEADDR, (char *)&reuse, sizeof(int)) == -1 )
	{
		error("Can't set the reuse option on the socket");
	}
	int c = bind(socket, (struct sockaddr *) &name, sizeof(name));
	if( c== 1)
	{
		error("Can't bind to socket");
	}
}

int open_listener_socket() 
{
	int s = socket(PF_INET, SOCK_STREAM, 0); 
	if (s == -1)
		error("Can't open socket");
	return s; 
}

int say(int socket, char *s)
{
	int result = send(socket, s, strlen(s), 0);
	if(result == -1)
	{
		fprintf(stderr, "%s: %s\n", "Error talking to the client", strerror(errno));
	}
	return result;
}

int read_in(int socket, char *buf, int len) {
	char *s = buf;
	int slen = len;
	int c = recv(socket, s, slen, 0); 
	
	while ((c > 0) && (s[c-1] != '\n')) 
	{
		s += c;
		slen -= c;
		c = recv(socket, s, slen, 0); 
	}
	if (c < 0) 
		return c;
	else if (c == 0) 
		buf[0] = '\0';
	else 
		s[c-1]='\0';
	return len - slen; 
}

void handle_shutdown(int sig)
{
	if(listener_d)  close(listener_d);
	fprintf(stderr, "Bye!\n");
	exit(0);
}


int catch_signal(int sig, void (*handler)(int)) 
{
	struct sigaction action; 
	action.sa_handler = handler; 
	sigemptyset(&action.sa_mask);
	 action.sa_flags = 0;
	return sigaction (sig, &action, NULL);
 }
```

## 线程

在C语言中有好多的线程库，用的最多的是  Posix（Portable Operating System Interface of Unix）Thread。和前面进程的使用比起来，pthread 可以说非常简单。

定义函数：

```
void* do_something(void *a) {
	int i = 0;
	for (i = 0; i < 5; i++) {
		sleep(1);
		puts("Does do!"); 
	}
	return NULL; 
}
``` 

启动线程：

```
#include<pthread.h>
....

pthread_t t;

if (pthread_create(&t, NULL, do_something, NULL) == -1)
{	
	error("Can't create thread t0");
}
```
线程锁也非常简单，pthread_mutex_t a_lock = PTHREAD_MUTEX_INITIALIZER;

使用：

```
pthread_mutex_lock(&beers_lock); 
/// some share varible
....
pthread_mutex_unlock(&beers_lock);
```


Congratulations! 这是本文结束的地方，本文中所有内容都来自 <Header First C> 一书，本文是此书的学习笔记。
C语言是大学期间学习的第一门机器语言，无论是上课还是教科书学的都很粗浅，从 <Header First C> 中学到了好多新的东西，所以
本文叫 “Hello  World” ， 重新认识 C 语言的世界。 
