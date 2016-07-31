##Docker的概念 --by CoolShell

***

###1. Linux Namespace

>Linux Namespace是Linux提供的一种内核级别环境隔离的方法，它提供了对UTS、IPC、mount、PID、network、User等的隔离机制。

三个主要系统调用：

* *clone()* – 实现线程的系统调用，用来创建一个新的进程，并可以通过设计参数(CLONE_NEWUTS/CLONE_NEWIPC/CLONE_NEWNS/CLONE_NEWPID/CLONE_NEWNET/CLONE_NEWUSER)达到隔离。
* *unshare()* – 使某进程脱离某个namespace.
* *setns()* – 把某进程加入到某个namespace.

贴一段clone系统调用的代码：

	#define _GNU_SOURCE
	#include <sys/types.h>
	#include <sys/wait.h>
	#include <stdio.h>
	#include <sched.h>
	#include <signal.h>
	#include <unistd.h>
 
	/* 定义一个给 clone 用的栈，栈大小1M */
	#define STACK_SIZE (1024 * 1024)
	static char container_stack[STACK_SIZE];
 
	char* const container_args[] = {
    	"/bin/bash",
    	NULL
	};
 
	int container_main(void* arg)
	{
    	printf("Container - inside the container!\n");
    	/* 直接执行一个shell，以便我们观察这个进程空间里的资源是否被隔离了 */
    	execv(container_args[0], container_args); 
    	printf("Something's wrong!\n");
    	return 1;
	}
 
	int main()
	{
    	printf("Parent - start a container!\n");
    	/* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    	int container_pid = clone(container_main, container_stack+STACK_SIZE, SIGCHLD, NULL); ／／子进程死后父进程收到SIGCHLD
    	/* 等待子进程结束 */
    	waitpid(container_pid, NULL, 0);
    	printf("Parent - container stopped!\n");
    	return 0;
	}
	
####UTS Namespace

下面启用UTS Namespace隔离，程序需要使用root权限并且子进程的hostname变成了 container。

	int container_main(void* arg)
	{
    	printf("Container - inside the container!\n");
    	sethostname("container",10); /* 设置hostname */
    	execv(container_args[0], container_args);
    	printf("Something's wrong!\n");
    	return 1;
	}
 
	int main()
	{
    	printf("Parent - start a container!\n");
    	int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWUTS | SIGCHLD, NULL); /*启用CLONE_NEWUTS Namespace隔离 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
	}


	vic@ubuntu:~$ sudo ./uts
	Parent - start a container!
	Container - inside the container!
	root@container:~# hostname
	container
	root@container:~# uname -n
	container
	
####IPC Namespace

启动IPC隔离，我们只需要在调用clone时加上CLONE_NEWIPC参数就可以了,目的是只有在同一个Namespace下的进程才能相互通信。但IPC需要有一个*全局的ID*，所以我们需要对这个全局ID进行隔离。

	// 创建一个ipc Queue
	vic@ubuntu:~$ ipcmk -Q 
	Message queue id: 0
 
	vic@ubuntu:~$ ipcs -q
	------ Message Queues --------
	key        msqid      owner      perms      used-bytes   messages    
	0xd0d56eb2 0          hchen      644        0            0
	
	// 如果我们运行没有CLONE_NEWIPC的程序，我们会看到，在子进程中还是能看到这个全启的IPC Queue。
	hchen@ubuntu:~$ sudo ./uts
	Parent - start a container!
	Container - inside the container!
 
	root@container:~# ipcs -q
	------ Message Queues --------
	key        msqid      owner      perms      used-bytes   messages    
	0xd0d56eb2 0          hchen      644        0            0
	
	//但是，如果我们运行加上了CLONE_NEWIPC的程序，我们就会下面的结果：
	root@ubuntu:~$ sudo./ipc
	Parent - start a container!
	Container - inside the container!
 
	root@container:~/linux_namespace# ipcs -q
 
	------ Message Queues --------
	key        msqid      owner      perms      used-bytes   messages
	

####PID Namespace

继续修改，在PID Namespace的隔离下，子进程的父进程号变成了1！

	int container_main(void* arg)
	{
    	/* 查看子进程的PID，我们可以看到其输出子进程的 pid 为 1 */
    	printf("Container [%5d] - inside the container!\n", getpid());
    	sethostname("container",10);
    	execv(container_args[0], container_args);
    	printf("Something's wrong!\n");
    	return 1;
	}
 
	int main()
	{
    	printf("Parent [%5d] - start a container!\n", getpid());
    	/*启用PID namespace - CLONE_NEWPID*/
    	int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWUTS | CLONE_NEWPID | SIGCHLD, NULL); 
    	waitpid(container_pid, NULL, 0);
    	printf("Parent - container stopped!\n");
    	return 0;
	}
	
	
	vic@ubuntu:~$ sudo ./pid
	Parent [ 3474] - start a container!
	Container [    1] - inside the container!
	root@container:~# echo $$
	1
	
子进程的父进程为init，作为所有进程的父进程，init有很多特权（比如：屏蔽信号等），另外，其还会检查所有进程的状态，我们知道，如果某个子进程脱离了父进程（父进程没有wait它），那么init就会负责回收资源并结束这个子进程。但是，**在子进程的shell里输入ps,top等命令，我们还是可以看得到所有进程。说明并没有完全隔离。**这是因为像ps, top这些命令会去读/proc文件系统，因为/proc文件系统在父进程和子进程都是一样的，所以这些命令显示的东西都是一样的。

所以，我们还需要对文件系统进行隔离。

####Mount Namespace

在下面面的实例中，我们在启用了mount namespace并在子进程中重新mount了/proc文件系统。

	int container_main(void* arg)
	{
    	printf("Container [%5d] - inside the container!\n", getpid());
    	sethostname("container",10);
    	/* 重新mount proc文件系统到 /proc下 */
    	system("mount -t proc proc /proc");
    	execv(container_args[0], container_args);
    	printf("Something's wrong!\n");
    	return 1;
	}
 
	int main()
	{
    	printf("Parent [%5d] - start a container!\n", getpid());
    	/* 启用Mount Namespace - 增加CLONE_NEWNS参数 */
    	int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, NULL);
    	waitpid(container_pid, NULL, 0);
    	printf("Parent - container stopped!\n");
    	return 0;
	}
	
	//ps中只剩下了两个进程，并且查看/proc文件夹，会发现干净了许多
	vic@ubuntu:~$ sudo ./pid.mnt
	Parent [ 3502] - start a container!
	Container [    1] - inside the container!
	root@container:~# ps -elf 
	F S UID  PID  PPID C PRI  NI ADDR SZ WCHAN  STIME TTY TIME CMD
	4 S root  1     0  0  80  0  -  6917 wait   19:55 pts/2 00:00:00 /bin/bash
	0 R root 14     1  0  80  0  -  5671 -      19:56 pts/2 00:00:00 ps -elf
	
在通过CLONE_NEWNS创建mount namespace后，父进程会把自己的文件结构复制给子进程中。而子进程中新的namespace中的所有mount操作都只影响自身的文件系统，而不对外界产生任何影响。这样可以做到比较严格地隔离。

####User Namespace

使用了CLONE_NEWUSER这个参数后，内部看到的UID和GID已经与外部不同了，默认显示为65534。那是因为容器找不到其真正的UID所以，设置上了最大的UID()其设置定义在/proc/sys/kernel/overflowuid)。要把容器中的uid和真实系统的uid给映射在一起，需要修改 /proc/<pid>/uid_map 和 /proc/<pid>/gid_map 这两个文件。这两个文件的格式为：

	ID-inside-ns（容器内id） ID-outside-ns（真实id） length
	
其中：

* 第一个字段ID-inside-ns表示在容器显示的UID或GID。
* 第二个字段ID-outside-ns表示容器外映射的真实的UID或GID。
* 第三个字段表示映射的范围，一般填1，表示一一对应。

另外，需要注意的是，

* 写这两个文件的进程需要这个namespace中的CAP_SETUID (CAP_SETGID)权限.
* 写入的进程必须是此user namespace的父或子的user namespace进程。
* 另外需要满如下条件之一：
  1. 父进程将effective uid/gid映射到子进程的user namespace中.
  2. 父进程如果有CAP_SETUID/CAP_SETGID权限，那么它将可以映射到父进程中的任一uid/gid。
  
下面用一段程序来实战：

	#define _GNU_SOURCE
	#include <stdio.h>
	#include <stdlib.h>
	#include <sys/types.h>
	#include <sys/wait.h>
	#include <sys/mount.h>
	#include <sys/capability.h>	
	#include <stdio.h>
	#include <sched.h>
	#include <signal.h>
	#include <unistd.h>
 
	#define STACK_SIZE (1024 * 1024)
 
	static char container_stack[STACK_SIZE];
	char* const container_args[] = {
    	"/bin/bash",
    	NULL
	};
 
	int pipefd[2];
 
	void set_map(char* file, int inside_id, int outside_id, int len) {
    	FILE* mapfd = fopen(file, "w");
    	if (NULL == mapfd) {
        	perror("open file error");
        	return;
    	}
    	fprintf(mapfd, "%d %d %d", inside_id, outside_id, len);
    	fclose(mapfd);
	}
 
	void set_uid_map(pid_t pid, int inside_id, int outside_id, int len) 	{
    	char file[256];
    	sprintf(file, "/proc/%d/uid_map", pid);
    	set_map(file, inside_id, outside_id, len);
	}
 
	void set_gid_map(pid_t pid, int inside_id, int outside_id, int len) 	{
		char file[256];
    	sprintf(file, "/proc/%d/gid_map", pid);
    	set_map(file, inside_id, outside_id, len);
	}
 
	int container_main(void* arg)
	{
    	printf("Container [%5d] - inside the container!\n", getpid());
    	printf("Container: eUID = %ld;  eGID = %ld, UID=%ld, GID=%ld\n",(long) geteuid(), (long) getegid(), (long) getuid(), (long)getgid());
 
    	/* 等待父进程通知后再往下执行（进程间的同步）在execv之前就做好user namespace的uid/gid的映射，这样，execv运行的/bin/bash就会因为我们设置了uid为0的inside-uid而变成#号的提示符。 */
    	char ch;
    	close(pipefd[1]);
    	read(pipefd[0], &ch, 1);
 
    	printf("Container [%5d] - setup hostname!\n", getpid());
    	//set hostname
    	sethostname("container",10);
 
    	//remount "/proc" to make sure the "top" and "ps" show container's information
    	mount("proc", "/proc", "proc", 0, NULL);
 
    	execv(container_args[0], container_args);
    	printf("Something's wrong!\n");
    	return 1;
	}
 
	int main()
	{
    	const int gid=getgid(), uid=getuid();
    	printf("Parent: eUID = %ld;  eGID = %ld, UID=%ld, GID=%ld\n",(long) geteuid(), (long) getegid(), (long) getuid(), (long) getgid());
    	pipe(pipefd);
    	printf("Parent [%5d] - start a container!\n", getpid());
 
    	int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWUSER | SIGCHLD, NULL);
    	printf("Parent [%5d] - Container [%5d]!\n", getpid(), container_pid);
 
    	//To map the uid/gid, 
    	//   we need edit the /proc/PID/uid_map (or /proc/PID/gid_map) in parent
    	//The file format is
    	//   ID-inside-ns   ID-outside-ns   length
    	//if no mapping, 
    	//   the uid will be taken from /proc/sys/kernel/overflowuid
    	//   the gid will be taken from /proc/sys/kernel/overflowgid
    	set_uid_map(container_pid, 0, uid, 1);
    	set_gid_map(container_pid, 0, gid, 1);
    	printf("Parent [%5d] - user/group mapping done!\n", getpid());
    	/* 通知子进程 */
    	close(pipefd[1]);
 
    	waitpid(container_pid, NULL, 0);
    	printf("Parent - container stopped!\n");
    	return 0;
	}
	
	vic@ubuntu:~$ id
	uid=1000(vic) gid=1000(vic) groups=1000(vic)
	vic@ubuntu:~$ ./user #<--以vic用户运行
	Parent: eUID = 1000;  eGID = 1000, UID=1000, GID=1000 
	Parent [ 3262] - start a container!
	Parent [ 3262] - Container [ 3263]!
	Parent [ 3262] - user/group mapping done!
	Container [    1] - inside the container!
	Container: eUID = 0;  eGID = 0, UID=0, GID=0 #<---Container里的UID/GID都为0了
	Container [    1] - setup hostname!
 
	root@container:~# id #<----我们可以看到容器里的用户和命令行提示符是root用户了
	uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
	
User Namespace是以普通用户运行，但是别的Namespace需要root权限，那么，如果我要同时使用多个Namespace，该怎么办呢？一般来说，我们先用一般用户创建User Namespace，然后把这个一般用户映射成root，在容器内用root来创建其它的Namesapce。

####Network Namespace

首先，我们先看个图，下面这个图基本上就是Docker在宿主机上的网络示意图（其中的物理网卡并不准确，因为docker可能会运行在一个VM中，所以，这里所谓的“物理网卡”其实也就是一个有可以路由的IP的网卡):

![Docker Network Namespace](http://coolshell.cn//wp-content/uploads/2015/04/network.namespace.jpg)

上图中，Docker使用了一个私有网段，172.40.1.0，docker还可能会使用10.0.0.0和192.168.0.0这两个私有网段，关键看你的路由表中是否配置了。

当你启动一个Docker容器后，你可以使用ip link show或ip addr show来查看当前宿主机的网络情况（我们可以看到有一个docker0，还有一个veth22a38e6的虚拟网卡——给容器用的）：

	vic@ubuntu:~$ ip link show
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state ... 
    	link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc ...
    	link/ether 00:0c:29:b7:67:7d brd ff:ff:ff:ff:ff:ff
	3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    	link/ether 56:84:7a:fe:97:99 brd ff:ff:ff:ff:ff:ff
	5: veth22a38e6: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc ...
    	link/ether 8e:30:2a:ac:8c:d1 brd ff:ff:ff:ff:ff:ff

下面用ip命令来模拟Docker中的网络原理：

	## 首先，我们先增加一个网桥lxcbr0，模仿docker0
	brctl addbr lxcbr0
	brctl stp lxcbr0 off
	ifconfig lxcbr0 192.168.10.1/24 up #为网桥设置IP地址
 
	## 接下来，我们要创建一个network namespace - ns1
 
	# 增加一个namesapce 命令为 ns1 （使用ip netns add命令）
	ip netns add ns1 
	# 激活namespace中的loopback，即127.0.0.1（使用ip netns exec ns1来操作ns1中的命令）
	ip netns exec ns1   ip link set dev lo up 
 
	## 然后，我们需要增加一对虚拟网卡
 
	# 增加一个pair虚拟网卡，注意其中的veth类型，其中一个网卡要按进容器中
	ip link add veth-ns1 type veth peer name lxcbr0.1 
	# 把 veth-ns1 按到namespace ns1中，这样容器中就会有一个新的网卡了
	ip link set veth-ns1 netns ns1 
	# 把容器里的 veth-ns1改名为 eth0 （容器外会冲突，容器内就不会了）
	ip netns exec ns1  ip link set dev veth-ns1 name eth0 
	# 为容器中的网卡分配一个IP地址，并激活它
	ip netns exec ns1 ifconfig eth0 192.168.10.11/24 up
	# 上面我们把veth-ns1这个网卡按到了容器中，然后我们要把lxcbr0.1添加上网桥上
	brctl addif lxcbr0
	
	 lxcbr0.1
	# 为容器增加一个路由规则，让容器可以访问外面的网络
	ip netns exec ns1     ip route add default via 192.168.10.1
	# 在/etc/netns下创建network namespce名称为ns1的目录，
	# 然后为这个namespace设置resolv.conf，这样，容器内就可以访问域名了
	mkdir -p /etc/netns/ns1
	echo "nameserver 8.8.8.8" > /etc/netns/ns1/resolv.conf
	
有几点不同的是：

* Docker没有用ip命令，而是自己实现了ip命令内的一些功能——是用了Raw Socket发些“奇怪”的数据。
* Docker的resolv.conf没有用这样的方式，而是用了Mount Namesapce的那种方式。
* Docker是用进程的PID来做Network Namespace的名称的。

无论是Docker的NAT方式，还是上文中把”物理网卡“设置为”混杂模式都会有性能上的问题，NAT不用说了，存在一个转发的开销，混杂模式呢，网卡上收到的负载都会完全交给所有的虚拟网卡上，于是就算一个网卡上没有数据，但也会被其它网卡上的数据所影响。

这两种方式都不够完美，我们知道，真正解决这种网络问题需要使用VLAN技术，于是Google的同学们为Linux内核实现了一个[IPVLAN](https://lwn.net/Articles/620087/)的驱动，这基本上就是为Docker量身定制的。

###2. Linux CGroup

> Linux Namespace解决了环境隔离的问题，我们还需要解决对计算机资源使用上的隔离，这就是Linux CGroup出来的原因。

Linux CGroup全称Linux Control Group，是Linux内核的一个功能，用来限制、控制与分离一个进程组群的资源（如CPU、内存、磁盘输入输出等），它主要提供了以下几个功能：

* Resource limitation: 限制资源使用，比如内存使用上限以及文件系统的缓存限制。
* Prioritization: 优先级控制，比如：CPU利用和磁盘IO吞吐。
* Accounting: 一些审计或一些统计，主要目的是为了计费。
* Control: 挂起进程，恢复执行进程。

在实际应用中，我们通常会利用CGroup做以下几件事情：

* 隔离一个进程集合（比如：nginx的所有进程），并限制他们所消费的资源，比如绑定CPU的核。
* 为这组进程分配其足够使用的内存
* 为这组进程分配相应的网络带宽和磁盘存储限制
* 限制访问某些设备（通过设置设备的白名单）

Linux把CGroup实现成了一个file system，我们可以自己mount：

	mkdir cgroup
	mount -t tmpfs cgroup_root ./cgroup
	mkdir cgroup/cpuset
	mount -t cgroup -ocpuset cpuset ./cgroup/cpuset/
	mkdir cgroup/cpu
	mount -t cgroup -ocpu cpu ./cgroup/cpu/
	mkdir cgroup/memory
	mount -t cgroup -omemory memory ./cgroup/memory/

一旦mount成功，你就会看到这些目录下就有好文件了，比如，如下所示的cpu和cpuset的子系统：

	vic@ubuntu:~$ ls /home/vic/cgroup/cpu /home/vic/cgroup/cpuset/
	/home/vic/cgroup/cpu:
	cgroup.clone_children  cgroup.sane_behavior  cpu.shares         	release_agent
	cgroup.event_control   cpu.cfs_period_us     cpu.stat           tasks
	cgroup.procs           cpu.cfs_quota_us      notify_on_release  user
 
	/home/vic/cgroup/cpuset/:
	cgroup.clone_children  cpuset.mem_hardwall             	cpuset.sched_load_balance
	cgroup.event_control   cpuset.memory_migrate           	cpuset.sched_relax_domain_level
	cgroup.procs           cpuset.memory_pressure          notify_on_release
	cgroup.sane_behavior   cpuset.memory_pressure_enabled  release_agent
	cpuset.cpu_exclusive   cpuset.memory_spread_page       tasks
	cpuset.cpus            cpuset.memory_spread_slab       user
	cpuset.mem_exclusive   cpuset.mems

一旦到cgroup下的各个子目录下面去mkdir，那么文件夹中会出现很多受保护的文件：

	vic@ubuntu$ sudo mkdir test
	[sudo] password for vic: 
	hchen@ubuntu$ ls ./test
	cgroup.clone_children  cgroup.procs       cpu.cfs_quota_us  cpu.stat           tasks
	cgroup.event_control   cpu.cfs_period_us  cpu.shares        notify_on_release
	
####CPU限制	

在我们之前创建的test文件夹中，修改其cpu.cfs_quota_us文件，为程序指定配额的CPU资源（20000代表20%），再将某个CPU占用很大进程的PID加入task文件中:

	root@ubuntu:~# echo 20000 > /home/vic/cgroup/cpu/test/cpu.cfs_quota_us
	root@ubuntu:~# echo 3529 >> /home/vic/cgroup/cpu/test/tasks

很快看到，这个进程的CPU占用会立马降成20%。

####内存限制

假设PID＝3530的程序存在内存泄漏，执行一下命令为该程序制定64k的内存使用限制，则：

	$ echo 64k > /home/vic/cgroup/memory/test/memory.limit_in_bytes
	$ echo 3530 > /home/vic/cgroup/memory/test/tasks
	
很快，这个问题就会因为内存问题被kill掉。

####磁盘IO限制

我们从/dev/sda读入文件写入/dev/null中：

	sudo dd if=/dev/sda1 of=/dev/null
	
通过iotop，可以看到磁盘的读速度约为55.74M/s:

	TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND          
	8128 be/4 root       55.74 M/s    0.00 B/s  0.00 % 85.65 % dd if=/de~=/dev/null...
 	
 把dd命令的PID放入tasks中并且将设备号（即8:0）以及相对应的限制读字节数写入blkio.throttle.read_bps_device中，再次执行iotop，可以发现读速度立马被限制到了1MB左右。

	root@ubuntu:~# echo '8:0 1048576'  > /home/vic/cgroup/blkio/haoel/blkio.throttle.read_bps_device 
	root@ubuntu:~# echo 8128 > /home/vic/cgroup/blkio/haoel/tasks
	
	root@ubuntu:~# iotop
	TID  PRIO  USER  DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND          
	8128 be/4 root   973.20 K/s    0.00 B/s  0.00 % 94.41 % dd if=/de~=/dev/null...

####CGroup子系统

control group具有以下子系统：

* blkio — 这​​​个​​​子​​​系​​​统​​​为​​​块​​​设​​​备​​​设​​​定​​​输​​​入​​​/输​​​出​​​限​​​制​​​，比​​​如​​​物​​​理​​​设​​​备​​​（磁​​​盘​​​，固​​​态​​​硬​​​盘​​​，USB 等​​​等​​​）。
* cpu — 这​​​个​​​子​​​系​​​统​​​使​​​用​​​调​​​度​​​程​​​序​​​提​​​供​​​对​​​ CPU 的​​​ cgroup 任​​​务​​​访​​​问​​​。​​​
* cpuacct — 这​​​个​​​子​​​系​​​统​​​自​​​动​​​生​​​成​​​ cgroup 中​​​任​​​务​​​所​​​使​​​用​​​的​​​ CPU 报​​​告​​​。​​​
* cpuset — 这​​​个​​​子​​​系​​​统​​​为​​​ cgroup 中​​​的​​​任​​​务​​​分​​​配​​​独​​​立​​​ CPU（在​​​多​​​核​​​系​​​统​​​）和​​​内​​​存​​​节​​​点​​​。​​​
* devices — 这​​​个​​​子​​​系​​​统​​​可​​​允​​​许​​​或​​​者​​​拒​​​绝​​​ cgroup 中​​​的​​​任​​​务​​​访​​​问​​​设​​​备​​​。​​​
* freezer — 这​​​个​​​子​​​系​​​统​​​挂​​​起​​​或​​​者​​​恢​​​复​​​ cgroup 中​​​的​​​任​​​务​​​。​​​
* memory — 这​​​个​​​子​​​系​​​统​​​设​​​定​​​ cgroup 中​​​任​​​务​​​使​​​用​​​的​​​内​​​存​​​限​​​制​​​，并​​​自​​​动​​​生​​​成​​​​​内​​​存​​​资​​​源使用​​​报​​​告​​​。​​​
* net_cls — 这​​​个​​​子​​​系​​​统​​​使​​​用​​​等​​​级​​​识​​​别​​​符​​​（classid）标​​​记​​​网​​​络​​​数​​​据​​​包​​​，可​​​允​​​许​​​ Linux 流​​​量​​​控​​​制​​​程​​​序​​​（tc）识​​​别​​​从​​​具​​​体​​​ cgroup 中​​​生​​​成​​​的​​​数​​​据​​​包​​​。​​​
* net_prio — 这个子系统用来设计网络流量的优先级
* hugetlb — 这个子系统主要针对于HugeTLB系统进行限制，这是一个大页文件系统。

关于各个子系统的参数细节，可以参考[Redhat 官方文档](https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux/6/html-single/Resource_Management_Guide/index.html#ch-Subsystems_and_Tunable_Parameters)

####CGroup术语 & 下一代CGroup

CGroup术语：

* 任务（Tasks）：就是系统的一个进程。
* 控制组（Control Group）：一组按照某种标准划分的进程，比如官方文档中的Professor和Student，或是WWW和System之类的，其表示了某进程组。CGroups中的资源控制都是以控制组为单位实现。一个进程可以加入到某个控制组。而资源的限制是定义在这个组上，就像上面示例中我们用的test一样。简单点说，cgroup的呈现就是一个目录带一系列的可配置文件。
* 层级（Hierarchy）：控制组可以组织成hierarchical的形式，即一颗控制组的树（目录结构）。控制组树上的子节点继承父结点的属性。简单点说，hierarchy就是在一个或多个子系统上的cgroups目录树。
* 子系统（Subsystem）：一个子系统就是一个资源控制器，比如CPU子系统就是控制CPU时间分配的一个控制器。子系统必须附加到一个层级上才能起作用，一个子系统附加到某个层级以后，这个层级上的所有控制族群都受到这个子系统的控制。CGroup的子系统可以有很多，也在不断增加中。

由于CGroup有多种对进程的分类方式，所以就很有可能会出现多层级正交的情况，另外一种情况是每个层级绑定了不同的子系统，那么也会造成管理的困难，所以在Kernel3.16之后，引入了[unified hierarchy](http://lwn.net/Articles/601840/)的设计，**__DEVEL_sane_behavior**的特性可以把所有子系统都挂载到根层级下，只有叶子节点可以存在tasks，非叶子节点只进行资源控制。

	$ sudo mount -t cgroup -o __DEVEL__sane_behavior cgroup ./cgroup
 
	$ ls ./cgroup
	cgroup.controllers  cgroup.procs  cgroup.sane_behavior  	cgroup.subtree_control 
 
	$ cat ./cgroup/cgroup.controllers
	cpuset cpu cpuacct memory devices freezer net_cls blkio perf_event net_prio hugetlb

mount了__DEVEL_sane_behavior之后，会生成4个文件,在这里mkdir一个子目录，里面也会有这四个文件,**上级的cgroup.subtree_control控制下级的cgroup.controllers**。

假设我们有以下目录结构：

	# A(b,m) - B(b,m) - C (b)
	#               \ - D (b) - E
 
	# 下面的命令中， +表示enable， -表示disable
	# 在B上的enable blkio
	echo +blkio > A/cgroup.subtree_control
	# 在C和D上enable blkio 
	echo +blkio > A/B/cgroup.subtree_control
	# 在B上enable memory  
	echo +memory > A/cgroup.subtree_control

这里cgroup只有上级控制下级，但无法传递到下下级，所以，C和D中没有memory的限制，E中没有blkio和memory的限制。而**本层的cgroup.controllers文件是个只读的，其中的内容就看上级的subtree_control里有什么了**。

根据unified hierarchy的设计理念，**任何被配置过subtree_control的目录都不能绑定进程，根结点除外**。所以，A,C,D,E可以绑上进程，但是B不行。

可以看到，这种方式干净的区分开了两个事，一个是*进程的分组*，一个是*对分组的资源控制*（以前这两个事完全混在一起），在目录继承上增加了些限制，这样可以避免一些模棱两可的情况。

###3. AUFS

>AUFS是一种Union File System，所谓UnionFS就是把不同物理位置的目录合并mount到同一个目录中。AUFS又叫Advance UnionFS,其主要目的是为了可靠性和性能，并且引入了一些新的功能，并且在使用上全兼容UnionFS。

AUFS可以把用户在新mount的文件下的更改保存在本地磁盘中，而保证另外的模版磁盘的完整性，这一特性也正好被Docker利用并实现了Docker中的分层镜像：

![Docker Layer](http://coolshell.cn//wp-content/uploads/2015/04/docker-filesystems-multilayer.png)

关于docker的分层镜像，除了aufs，docker还支持btrfs, devicemapper和vfs，你可以使用 -s 或 –storage-driver= 选项来指定相关的镜像存储。在Ubuntu 14.04下，docker默认Ubuntu的 aufs（在CentOS7下，用的是devicemapper）我们可以在下面的目录中查看相关的每个层的镜像：

	/var/lib/docker/aufs/diff/<id>
	
AUFS有所有Union FS的特性，把多个目录，合并成同一个目录，并可以为每个需要合并的目录指定相应的权限，实时的添加、删除、修改已经被mount好的目录。而且，它还能在多个可写的branch/dir间进行负载均衡。

下面介绍几个AUFS的术语：

* Branch:  就是各个要被union起来的目录
  1. Branch根据被union的顺序形成一个stack，一般来说最上面的是可写的，下面的都是只读的。
  2. Branch的stack可以在被mount后进行修改，比如：修改顺序，加入新的branch，或是删除其中的branch，或是直接修改branch的权限
* Whiteout 
  1. 如果UnionFS中的某个目录被删除了，那么就应该不可见了，就算是在底层的branch中还有这个目录，那也应该不可见了。Whiteout就是某个上层目录覆盖了下层的相同名字的目录。用于隐藏低层分支的文件，也用于阻止readdir进入低层分支。
  2. 在隐藏低层档的情况下，whiteout的名字是’.wh.<filename>’。
  3. 在阻止readdir的情况下，名字是’.wh..wh..opq’或者 ’.wh.__dir_opaque’。
* Opaque: 允许任何下层的某个目录显示出来。

AUFS要遍历所有的branch去查找一个inode，这是个O(n)的算法（很明显，这个算法有很大的改进空间的）所以，branch越多，查找文件的性能也就越慢。但是，一旦AUFS找到了这个文件的inode，那后以后的读写和操作原文件基本上是一样的。

### 4.DeviceMapper

>DeviceMapper自Linux 2.6被引入成为了Linux最重要的一个技术。它在内核中支持逻辑卷管理的通用设备映射机制，它为实现用于存储资源管理的块设备驱动提供了一个高度模块化的内核架构.

因为Docker首选的AUFS并不在Linux的内核主干里，所以，对于非Ubuntu的Linux分发包，比如CentOS，就无法使用AUFS作为Docker的文件系统了。于是作为第二优先级的DeviceMapper就被拿出来做分层镜像的一个实现。

DeviceMapper自Linux 2.6被引入成为了Linux最重要的一个技术。它在内核中支持逻辑卷管理的通用设备映射机制，它为实现用于存储资源管理的块设备驱动提供了一个高度模块化的内核架构，它包含三个重要的对象概念，**Mapped Device、Mapping Table、Target device**。

Mapped Device 是一个逻辑抽象，可以理解成为内核向外提供的逻辑设备，它通过Mapping Table描述的映射关系和 Target Device 建立映射。Target device 表示的是 Mapped Device 所映射的物理空间段，对 Mapped Device 所表示的逻辑设备来说，就是该逻辑设备映射到的一个物理设备。

Mapping Table里有 Mapped Device 逻辑的起始地址、范围、和表示在 Target Device 所在物理设备的地址偏移量以及Target 类型等信息（注：这些地址和偏移量都是以磁盘的扇区为单位的，即 512 个字节大小，所以，当你看到128的时候，其实表示的是128*512=64K）。

DeviceMapper在内核中通过一个一个模块化的 Target Driver 插件实现对 IO 请求的过滤或者重新定向等工作，DeviceMapper只是一个框架，在这个框架上，我们可以插入各种各样的策略.在这诸多“插件”中，有一个东西叫**Thin Provisioning Snapshot**，这是Docker使用DeviceMapper中最重要的模块。

![Device Mapper Kernel Architecture](http://coolshell.cn//wp-content/uploads/2015/08/device.mapper.2.gif)

####Thin Provisioning

Thin Provisioning和我们计算机中的内存管理中用到的“虚拟内存技术”有这异曲同工之处，简单的以两张图片来说明：

![Fat Provisioning](http://coolshell.cn//wp-content/uploads/2015/08/thin-provisioning-1.jpg)

![Thin Provisioning](http://coolshell.cn//wp-content/uploads/2015/08/thin-provisioning-2.jpg)

####Thin Provisioning Snapshot

* Snapshot来自LVM（Logic Volumn Manager），它可以在不中断服务的情况下为某个device打一个快照。
* Snapshot是Copy-On-Write的，也就是说，只有发生了修改，才会对对应的内存进行拷贝。
* Thin Provisioning还处理实验阶段，不要上Production。






