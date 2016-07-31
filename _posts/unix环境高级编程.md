#UNIX环境高级编程

##补一些基础的知识

* 口令文件`/etc/passwd`,用于存储登陆项，一般来说由7个冒号分割的字段组成，依次是：登陆名，加密口令，数字用户ID，数字组ID，注释，起始目录，以及shell程序，如：`xuziyan:x:205:500:xuziyan@lejent:/data/home/xuziyan:/usr/bin/zsh`
* 用户id为0的为root用户
* 用户组名与组id映射的文件一般为`/etc/group`
* UNIX系统维护3个进程时间值，*时钟时间*指进程运行的时间总量，与系统中同时运行的进程数有关；*用户CPU时间*指用户指令所用的时间量；*系统CPU时间*指内核程序所经历的时间

#####以下都是跳着看的，先从套接字开始


##网络进程通信IPC
正如使用文件描述符访问文件，应用程序使用套接字访问套接字，许多处理文件描述符的函数（read、write）也可以用于处理套接字描述符。创建一个套接字调用socket函数：

	#include<sys/socket.h>
	int socket(int domain, int type, int protocol);
	//出错返回-1，成功返回套接字描述符
	
 