## 概述

熟悉linux的应该知道，`init`是linux用户空间的第一个进程，而android是基于linux内核的，所以`init`也是android用户空间的第一个进程。   

作为`"天字第一号进程"`，它的进程号是`1`，担负了很多重要的工作，本章节主要介绍了它的两个工作：

• `如何创建zygote：zygote是整个java世界的开创者，将在下一个章节重点介绍。`   
• `如何创建属性服务：属性服务property service支撑起了android的属性机制。`

## init分析

本章节有大量的源码分析。`注意`：看代码看的算法、流程、设计、思路、实现，不要纠结代码于本身。

```c
// init.c

// 创建一些文件夹，并挂载设备.
mkdir("/dev/socket", 0755);
mount("proc", "/proc", "proc", 0, NULL);
mount("sysfs", "/sysfs", "sysfs", 0, NULL);

// 重定向标准输入/输出/错误输出到/dev/__null__
open_devnull_stdio()

// 设置init的日志输出设备为/dev/__kmsg__
log_init()

//  解析init.rc配置文件
parse_config_file("/init.rc");

// 通过该函数读取/proc/cpuinfo获取手机的hardware名
get_hardware_name();
snprintf(tmp, sizeof(tmp), "/init.%s.rc", hardware);
// 解析和手机相关的配置文件parese_config_file(tmp);

/*
  rc配置的解析是init工作的重点之一，rc文件中包含各种Action(动作)，不同Action可以设置不同的执行阶段。
  之所以分阶段，是因为动作之间有先后顺序，有些动作需要其他动作执行完成后，才能执行。
  哪些Action在什么时候（阶段）完成，由配置文件决定。
  init将动作执行的事件划分为四个阶段：early-init、init、early-boot、boot。
*/

// 执行处于early-init阶段的action
action_for_each_trigger("early-init", action_add_queue_tail);
drain_action_queue();

// 创建利用Uevent与linux内核交互的socket，用于监听来自内核的Uevent事件？
device_fd = device_init();

// 初始化和属性相关的资源
property_init();

// 初始化/dev/keychord设备，与调试有关
keychord_fd = open_keychord();

// 加载系统的开机画面，貌似和开机动画控制程序bootanimation加载的开机动画文件不一样。
if (load_565rle_image(INIT_IMAGE_FILE)) {
  // 如果加载失败，则会打开/dev/ty0设备,并输出"ANDROID"字样作为开机画面
}

// 调用property_set函数设置属性项 - 这里属性服务就已经启动啦？没启动的话，这里的属性设置能生效吗？
propertt_set("ro.bootloader", bootloader[0] ? bootloader : "unknown");

// 执行位于init阶段的动作
action_for_each_trigger("init", action_add_queue_tail);
drain_action_queue();

// 启动属性服务，注意返回是一个文件描述符
property_set_fd = start_property_service();

// 调用socketpair创建两个已经connect好的socket？干嘛的呢？
if (socketpari(AF_UNIX, SOCK_STREAM, 0, s) == 0) {
	signal_fd = s[0];             // client？
	signal_recv_fd = s[1];   // server？
}

// 执行位于early-boot和boot阶段的动作
action_for_each_trigger("earlty-boot", action_add_queue_tail);
action_for_each_trigger("boot", action_add_queue_tail);
drain_action_queue();

// init监听四个方面的事件

ufds[0].fd = device_fd;  // 监听来自内核的Uevent事件
ufds[0].events = POLLIN;
ufds[1].fd = property_set_fd; // 监听来自属性服务器的事件
ufds[1].events = POLLIN;
ufds[2].fd = signal_recv_fd;  // 监听socketpair创建的socket事件
ufds[2].events = POLLIN;
fd_count = 3;
if (keychord_fd > 0) {
	// 如果keychord_fd设备初始化成功，则init也会关注来自这个设备的事件
	fd_count++;
}

#if BOOTCHART
// bootchart 是一个小工具，用于对系统的性能进行分析，并生成系统启动过程的图标。
// 最大的用途就是帮助提升系统的启动速度
# endif

// 做完上述工作后，init将进入一个无限循环
for (;;) {
	for (i = 0; I < fd_count; ++i)
		ufds[i].revents = 0;    // events一般表示监听的事件，revent(return?realy?)表示实际发生/返回的事件。
	// 执行动作
	drain_action_queue();
	// 重启已经死去的进程 - 守护进程就是这里复活的？
	restart_processes();
	
	// 调用poll等待事件的发生
	nr = poll(ufds, fd_count, timeout);
	
	// 接收事件处理
	if (ufds[2].revents == POLLIN) { .. }
	if (ufds[0].revents == POLLIN) { handle_device_fd(device_fd); }
	if (ufds[1].revents == POLLIN) { handle_property_set_fd(property_set_fd); }
	if (ufds[3].revents == POLLIN) { handle_keychord(keychord_fd); }
}
```
init的代码很长，书中将其工作流程**精简**为以下几点：
+ 解析rc配置文件（int.rc/xxx/rc）
+ 执行rc配置文件中各个阶段的Action，zygote就是在其中某个阶段完成的
+ 初始化属性相关资源，并通过property_start_service启动属性服务
+ init进入死循环，并监听来自各个方面的事件，并处理

>PS：看本书的目的是为了长见识，增加知识的广度，所以init的其他工作多少了解点（开机动画、系统优化、关闭标准in/out/err、复活进程），也是有裨益的。

### 解析rc配置文件

接着回到书中rc配置文件的解析流程，类似rc文件等android配置文件，解析的流程无非是格式化文本，然后将配置内容转换为各种数据结构。
>对于具体的文本解析方法，确实比较有趣，但不是阅读本书的重点，直接跳过书中的相关内容，重点看看拿到配置后的相关处理逻辑。

与配置文件解析相关的一些接口：
```c
parse_config_file         //  加载配置文件
parse_config              
loopup_keyword         // 关键字
parse_new_section    // 解析section
parse_line
```
>解析配置的文件原理挺简单，首先找到配置文件的一个section，然后针对不同的section使用不同的解析函数来解析

如何解析到section呢？书中说法是通过`关键字`来找到`section`，关键字的定义在`keywords.h`，它的部分内容如下：
```c
// 处理函数声明？
int do_chroot(int nargs, char **args);
..
int do_class_start(int nargs, char **args);
int do_mkdir(int nargs, char **args);
int do_setprop(int nargs, char **args);
int do_start(int nargs, char **args);
int do_chmod(int nargs, char **args);
…

// 没看懂这个宏定义？
#define KEYWORD(symbol, flags, nargs, func) K_##symbol,

// 关键字声明？
KEYWORD(chdir,       COMMAND, 1, do_chdir)               // 注意第二个参数，它是一个COMMAND 
KEYWORD(class_start, COMMAND, 1, do_class_start)  
KEYWORD(mkdir,       COMMAND, 1, do_mkdir)                
KEYWORD(restart,     COMMAND, 1, do_restart)           
…
KEYWORD(disabled,    OPTION,  0, 0)                             // 注意第二个参数，它是一个OPTION
KEYWORD(group,       OPTION,  0, 0)
KEYWORD(oneshot,     OPTION,  0, 0)
KEYWORD(onrestart,   OPTION,  0, 0)
KEYWORD(socket,      OPTION,  0, 0)

…
KEYWORD(on,          SECTION, 0, 0)                              // 注意第二个参数，它是一个SECTION
KEYWORD(service,     SECTION, 0, 0)                             
…
```
>从`keywords.h`可以看出`rc`配置文件支持哪些`SECTION`，哪些是`SECTION`，支持哪些`COMMAND`、哪些`OPTION`
>>有时候看文档看不懂，看代码一眼懂，`keywords.h`和`init/parser.c`的部分实现，还是比较精巧的，书中也特别介绍了

解析rc文件，最起码应该对rc文件的格式/语法有清晰的了解，否则rc文件都看不懂，解析rc的代码就更加看不懂了。
[官方的rc语法参考](https://android.googlesource.com/platform/system/core/+/master/init/README.md)
[What is inside the init.rc and what is it used for.](https://community.nxp.com/t5/i-MX-Processors-Knowledge-Base/What-is-inside-the-init-rc-and-what-is-it-used-for/ta-p/1106360)

>这里最好对各种SECTION、OPTION、COMMAND的含义有一定的了解
>>可以参考：https://www.cnblogs.com/kongbursi-2292702937/p/17215180.html


书中展示了`init.rc`的部分内容：
```sh
on property:persist.service.adb.enable=1  # SECTION  后面跟的一般叫做触发器trigger，可以理解为执行条件.
	start adbd # COMMAND

on boot  # boot是一个触发器，android定义了很多触发器，例如：early-init、init等等
	…

service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
	socket zygote stream 666
	onrestart write /sys/android_power/request_state wake
	onrestart write /sys/power/state on
	onrestart restart media
	
service media /system/bin/mediaserver # SECTION  服务名 服务路径 服务参数
	user media # OPTION
	group system audio camera graphics inet net_bt net_bt_admin net_raw # OPTION
	ioprio rt 4 # OPTION

service dumpstate /system/bin/dumpstate -s # SECTION  服务名 服务路径 服务参数
	socket dumpstate stream 0660 shell log # OPTION
	disabled # OPTION
	oneshot # OPTION
```

前面提到过，`init`的核心工作之一是`启动zygote`,而该工作就在`init.rc`的解析流程中，`zygote`对应的`service section`内容是：
```rc
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
	socket zygote stream 666
	onrestart write /sys/android_power/request_state wake
	onrestart write /sys/power/state on
	onrestart restart media
```
`section`的处理函数为`parse_new_section`：
```c
void parse_new_section(…)
{
	switch(kw) {
	// service类型SECTION
	case K_service:
		// 内部使用parse_service和parse_line_service解析services
	// on类型SECTION
	case K_on:
		// …
	}
}
```
在介绍`parse_service`和`parse_line_service`之前，先介绍两个关键的数据结构：

+ service结构体
```c
// service结构体可以理解为init.rc中service SECTION的抽象？，貌似包含了相当多的信息.
// init在解析init.rc后，会将解析service保存在全局的service_list中，然后挨个处理

struct listnode slist;   // listnode，非常常见的链表数据结构，用于将多个service链表链接成一个.
const char* name;    //  service名字，例如："zygote"
const char* classname; // service所属class的名字，默认是"default"，我的理解是类似组的概念，一组一组的服务批量启动？
..
// service的属性，例如：SVC_DISABLED(不随class自启动)、SVC_ONESHOT(退出后不需要重启，只启动一次就可以了)
unsigned flags; 
..
pid_t pid; // 进程号
time_t time_started;   // 上一次启动时间
struct socketinfo* sockets;  // socket相关
..
struct action onrestart;   // action的抽象.
..
int nargs; // 参数个数
char *args[1]; // 参数
```

+ action结构体
```c
// 是否可以理解为service中结构OPTION字段/COMMAND字段的抽象？
// 同样有链表保存action，不过这里有三个action链表

struct listnode alist;         // 保存所有action的列表，
struct listnode qlist;  // 保存等待执行的action？？ -- 感觉不是特别好理解.
struct listnode  tlist; // 保存待某些条件满足后需执行的action.

unsigned hash; // 干嘛的？
const char* name; // action name？

// 执行的命令列表.
struct listnode commands;
struct command *current;
```

回到`parse_service`的内部处理流程：
```c
// 整体逻辑还是比较简单的，就是根据rc文件，构建一个service结构体.
static void parse_service(..)
{
	struct service* svc;
	// 判断service_list链表是否已经有了该service
	svc = service_find_by_name(args[1]);
	if (svc) return; // 如果已经有了，直接返回对应的节点service
	
	// 没有的话要重新构建
	svc = calloc(..);
	..
	svc->name = args[1];
	svc->onrestart.name = "onrestart"; // ？？
	
	list_init(&sav->onrestart.commands);
	
	// 把zygote这个service添加到全局的service_list中
	list_add_tail(&service_list, &svc->slist);
	return svc;
}
```

接着是`parse_line_service`的内部处理流程：
```c
// 接着解析service SECTION后面的OPTION了，这样一个SECTION就解析完了.
static void parse_line_service(..)
{
	struct service *svc = state->context;
	struct command *cmd;
	…
	// 根据关键字来做各种处理
	kw = lookup_keyword(args[0]);
	switch (kw) {
		case K_capanility:
		case K_class:
		case K_oneshot:
			// 设置svc属性
			svc->flags |= SVC_ONESHOT;
			break;
		case K_onrestart:
			// 创建command结构体
			cmd = malloc(..);
			cmd->func = kw_func(kw); // COMMAND处理函数.
			..
			list_add_tail(&svc->onerestart.commands, &cmd->clist);
		case K_socket:
			svc->sockets = …;
	}
}
```
执行完成`parse_service`和`parse_line_service`之后，`rc文件`中的SECTION信息，将结构化为`service`数据结构。
>前前后后那么多逻辑，我的理解，本质上其实就是rc文件数据的反序列化？

### zygote启动

init.rc的解析的目的在于告知init进程需要启动哪些服务和初始化操作，要启动的服务中，最关键的就是zygote.

书中从`zygote`的`Service SECTION`的解析，到`Service`和`Action`的两个结构体的构建，分别作了详细的说明。

接着就是重头戏，zygote service如何启动的。在`init.rc`中有这样一句话：
```sh
# class_start是一个COMMAND，对应的函数为do_class_start，在`keywords.h`中有声明
class_start default
```
该命令位于`boot section`范围内，还记得文章开头init.c的代码解析吗，init会执行到下面几句代码：
```c
// 将boot section节的command加入到执行队列
action_for_each_trigger("boot", action_add_queue_tail);
// 执行队列里的命令，class是一个COMMAND，所有它对应的do_class_start会被执行
drain_action_queue();
```
下面是枯燥的代码分析阶段，首先是`do_class_start`函数：
```c
// builtins.c
int do_class_start()
{
	// args为class_start命令对应的参数，执行命令为：`class_start default`，参数自然是`default`
	// 该函数将从service_list(分析rc文件的小节提到过)中，找到`classname`为`default`的service。有点过滤的意思.
	// 接着将这些service通过service_start_if_not_disable函数拉起.
	service_for_each_class(args[1], service_start_if_not_disable);
}
```
接着是`service_start_if_not_disable`函数：
```c
static void service_start_if_not_disable()
{
	// serice结构体创建时的flags在这里被用到
	if (!(svc->flags & SVC_DISABLED)) {
		service_start(svc, NULL);
	}
}
```
再看看`service_start`函数：
```c
void service_start()
{
	// 更新service的一些字段
	svc->flags &= (~(SVC_DISABLE|SVC_RESTARTING));
	svc->time_started = 0;
	
	// 如果service已经在运行，则不用处理.
	if (svc->flags & SVC_RUNNING) {
		return;
	}
	
	// 在启动service之前需要判断其对应的可执行性文件是否存在
	// zygote对应的可执行文件为：/system/bin/app_process
	if (stat(svc->args[0], &s) != 0) {
		svc->flags != SVC_DISABLE;
		return;
	}
	…
	// service运行在init所创建的子进程中
	pid = fork();
	if (pid == 0) {
		// pid为零，表示运行在子进程中
		…
		// 进程的环境变量设置
		get_property_workspace(..); // 属性的存储空间信息，干啥的？
		add_environment("ANDROID_PROPERTY_WORKSPACE", tmp);
		for (ei = svc->envvars; ei; ei = ei->next) 
			add_environment(ei->name, ei->value);
			
		// 创建socket，并添加到环境变量
		for (…) {
			int s = create_socket(..);
			publish_socket(si->name, s);
		}
		…
		// 设置uid、gid
		setpgid(..);
		
		// 执行/system/bin/app_process，这样就进入到app_process的main函数中，zygote的代码逻辑开始执行了.
		// fork、execve都是标准的linux系统调用
		execve(svc->args[0], (char**)svc->args, (char**)ENV);
	}
	
	// 这里运行在父进程，对svc的一些信息进行更新.
	svc->time_started = gettime(); // 启动时间
	svc->pid = pid;
	svc->flags != SVC_RUNNING;
	
	// 每一个service都有一个属性，zygote的属性就是`init.svc.zygote`，现在它的值为`running`
	notify_service_state(…);
}
```
到此，zygote通过`fork`和`exevc`创建执行了.

### zygote重启

在zygote的service SECTION中，有一些关于`onrestart`的命令，回顾一下：
```rc
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
	socket zygote stream 666
	onrestart write /sys/android_power/request_state wake
	onrestart write /sys/power/state on
	onrestart restart media
```
表示当zygote重启时，会执行的操作，下面看看zygote进程死后，它的父进程init父如何处理：
```c
// init.c
// 当子进程(zygote)退出时，init的这个信号处理函数会被调用.
static void sigchld_handler()
{
	// 它的处理函数，很简单，向文件描述符signal_fd中写数据.
	//这个signal_fd就是init开始时通过socketpair创建的两个socet中的一个
	//既然是一对，其中一个写，自然另一个socket就能接收到.
	write(signal_fd, &s, 1);
}
```
还记得init会通过`poll`监听来自各个方面的事件吗？其中就包括socketpair创建的socket事件，当出现可读事件时，poll开始返回。
```c
nr = poll(…);

if (usfds[2].revents == POLLIN) {
	// 读取sigchld_handler中写入的数据
	read(…);
	// 调用处理函数
	while(!wait_for_one_process(0));
	continue;
}

static int wait_for_one_process()
{
	// 等待指定进程ID（pid）的子进程终止，并获取它们的退出状态。pid == -1表示等待任何子进程。
	while ( pid == waitpid(…))
	
	// 找到死掉的service？这里以zygote为例.
	svc = service_find_by_pid(pid);
	
	if (!(svc->flags & SVC_ONESHOT)) {
		// 杀掉zygote创建的所有子进程(参数1小于-1表示杀死进程组中的每个进程)，这就是zygote死后，JAVA世界崩溃的原因.
		kill(-pid, SIGKILL);
	}
	
	// 清理socket信息
	for (…) {
		unlink(…);
	}
	
	…
	now = gettime();
	…
	
	// 如果设置了SVC_CRITICAL标识，则四分钟内该服务重启的次数不能超过4次，否则机器可能会在重启时进入recovery模式.
	// 从init.rc中，只有servicemanager进程有这种待遇。看样子蛮重要啊，也没见书中介绍啊？
	if (svc->flags & SVC_CRITICAL) {
		if (svc->time_crashed + CRITICAL_CRASH_WINDOW >= now)  // 重启时间判断
			if (++svc->nr_crashed > CRITICAL_CRASH_THRESHOLD) {  // 重启次数判断
				…
				__reboot(…);
			}
		} else {
			svc->time_crashed = now;
			svc->nr_crashed = 1;
		}
	}
	
	svc->flags |= SVC_RESTARTING;
	// 开始执行该service onrestart中的COMMAND，终于到正事了.
	list_for_each(node, &svc->onrestart.commands) {
		cmd = node_to_item(…)
		cmd->func(…)
	}
	
	// 设置init.svc.zygote属性值为restarting
	notify_service_state(…)
}
```
当poll函数返回后，会进入下一个循环，并重新启动zygote
```c
// 做完上述工作后，init将进入一个无限循环 -> 本章节开头解析的代码.
for (;;) {
	for (i = 0; I < fd_count; ++i)
		ufds[i].revents = 0;    // events一般表示监听的事件，revent(return?realy?)表示实际发生/返回的事件。
	// 执行动作
	drain_action_queue();
	// 重启已经死去的进程 ，在这里会重启所有flag标志位SVC_RESTARTING的service -> zygote
	restart_processes();
	
	// 刚才的poll函数返回后
	nr = poll(ufds, fd_count, timeout);
```

## 属性服务

init做的主要两件事，一件为启动zygote，另一件则为启动属性服务。

>书中解释下属性服务的由来，它是一种类似windows注册表的东西，用来存储key/value这样的键值对，这样系统和程序可以将一些属性存储起来，
>即使系统重启或者程序重启，还能够根据存储在注册表中设置的属性，进行相应的初始化工作，android的属性服务（property service）就是类似注册表一样的机制。

在`init.c`中与属性服务有关的代码有下面两行：
```c
property_init()      // 初始化属性服务，提前进行一些准备工作.
property_set_fd = start_property_service();    // 启动属性服务.
```

### 属性服务初始化
```
// property_service.c
void property_init()
{
	// 初始化属性存储区域，分配空间用于保存属性信息？
	init_property_area();
	// 加载default.prop文件
	load_properties_from_file(PROP_PATH_RAMDISK_DEFAULT);
}

static int init_property_area(void)
{
	prop_area *pa;
	
	// 如果已分配，则直接返回
	if (pa_info_array)
		return -1;
	
	/*
	存储的空间的结构体如下：
	typedef struct {
		void *data;         // 存储空间其实地址
		size_t size; // 存储空间的大小int fd;      // 共享内存的文件描述符
	} workspace pa_workspace;
	*/
	
	// 内部调用android系统的ashmem_create_region函数创建一块共享内存，大小为PA_SIZE（32765字节(32K)）用于存储属性信息.
	if (init_workspace(&pa_workspace, PA_SIZE))
	//  问了chatGPT，目的是为了执行exec()时前，关闭对应的文件描述符，不传递给新进程，可以提高程序的安全性和健壮性。
	fcntl(pa_workspace.fd, F_SETFD, FD_CLOEXEC); 
	
	// 前PA_INFO_START（1024）字节，用于存储头部信息
	pa_info_array = (void*)((char*)pa_workspace.data) + PA_INFO_START);
	
	pa = pa_workspace.data;
	memset(pa, 0, PA_SIZE);
	pa->magic = PROP_AREA_MAGIC; // 搞的和协议似的.
	pa->version = PROP_AREA_VERSION;
	
	// bionic libc输出的变量，干嘛的呢？估计和跨进程的共享内存访问相关。
	__system_property_area__ = pa;
}
```
关于上面的代码，作者给出了一些解释，属性区域虽然是init进程创建的，但是android系统也希望其他进程也能访问该区域，
所以就把该区域创建在共享内存上了，因为共享内存是可以跨进程的。
>这里有个疑问，从后面的内容可以看出，其他进程是通过socket实现和属性服务的进程间通信的，好像并没有使用到共享内存，所以整个共享内存是为了共享给谁呢？   
有了这个共享内存，那其他进程要怎么知道这个共享内存呢？
>android利用gcc的construct属性，这个属性指明了一个__libc_prenit函数，当bionic lib库被加载时，将自动调用这个libc_prenit接口，
>该这个函数内部就将完成共享内存到本地进程的映射工作 
>>这个功能挺牛逼，可以用在普通的动态库吗？这样就不需要每个使用该库的模块手动的创建实例，然后调用初始化接口了，在库链接时直接调用了，非常方便.

这里给出gcc constuctor属性的使用示例：
```c
#include <stdio.h>

// 该属性可以用于声明需要在main函数之前执行的代码.
// 该属性也可以用于动态库的初始化，在库加载时，执行相关的代码.
void my_init_function() __attribute__((constructor));

void my_init_function() {
    printf("This function is called before main()!\n");
    // 执行一些初始化操作
}

int main() {
    printf("main() is being executed.\n");
    return 0;
}
```

看看binoic lib库中与属性相关的逻辑：
```c
//libc_init_dynamic.c
// constructor属性，指示加载器加载该库后，首先调用__libc_prenit函数.
void __attribute__((constructor)) __libc_prenit(void);
void __libc_oprenit(void)
{
	…
	__libc_init_common(elfdata);
	…
}

// libc_init_common.c
void __libc_init_common(…)
{
	…
	// 初始化客户端的属性存储区域？？
	__system_properties_init();
}

// system_properties.c
int __system_properties_init(void)
{
	…
	// 提取环境变量的值，该值在zygote创建过程中设置(add_environment("ANDROID_PROPERTY_WORKSPACE", tmp);)
	env = getenv("ANDROID_PROPERTY_WORKSPACE");
	// 该值是一个文件描述符
	fd = atoi(env);
	sz = atoi(env+1);
	// 将init创建的共享内存映射到本进程空间，这样本地进程就可以使用这块共享内存了.
	// 这里设置了PROT_READ(只读)属性，所以客户进程只能读取属性(get)，无法设置属性(set)。我猜是set的话，要考虑进程间的资源竞争？比较麻烦？
	pa = mmap(0, sz, PROT_READ, MAP_SHARED, fd, 0);
	if (pa == MAP_FAILED) 
		return -1;
	
	if ((pa->magic != PROP_AREA_MAGIC) || (pa->version != PROP_AREA_VERSION)) {
		munmap(pa, sz);
		return -1;
	}
	
	// 这里为啥又有这种操作呢？
	__system_property_area__ = pa;
}
```

### 属性服务启动
属性服务是由`start_property_service`函数启动，上层客户端通过`共享内存`机制读取属性值，通过`socket通信`机制设置属性值。
```c
// property_service.c
int start_property_service(void)
{
	int fd;
	/*
	 加载属性文件，其实就是解析这些文件，然后将`key-value`键值对保存在前面创建的属性空间（共享内存）中.
	Android系统提供了四个存储属性的文件，分别是：
	#define PROP_PATH_RAMDISK_DEFAULT         "/default.prop"
	#define PROP_PATH_SYSTEM_BUILD               "/system/build.prop"
	#define PROP_PATH_SYSTEM_DEFAULT          "/system/default.prop"
	#define PROP_PATH_LOCAL_OVERRIDE          "/data/local.prop"
	*/
	load_properties_from_file(PROP_PATH_SYSTEM_BUILD);
	load_properties_from_file(PROP_PATH_SYSTEM_DEFAULT);
	load_properties_from_file(PROP_PATH_LOCAL_OVERRIDE);
	
	//有一些属性需要保存到永久介质上，存储在`/data/property`目录下，并且这些文件的文件名必须以`persist.`开头
	load_persistent_properties();
	
	// 创建一个socket，用于IPC通信.
	fd = create_socket(PROP_SERVICE_NAME, SOCK_STREAM, 066, 0, 0);
	if (fd < 0) return -1;
	fcntl(fd, F_SETFD, FD_CLOEXEC);
	fcntl(fd, F_SETFL, O_NONBLOCK);
	listen(fd, 8);
	
	// 创建属性服务，返回的是一个socket套接字.
	return fd;
}
```
>属性服务好像和zygote不一样，貌似并不是一个进程，更像是一套流程的抽象，它的根基是共享内存、socket、几个存储属性的文件.
属性服务处理socket接收请求的地方在init进程，代码如下：
```c
// init.c
if (ufds[1].revents == POLLIN)
	handle_property_set_fd(property_set_fd);  // 参数就是属性服务创建的套接字
	
// property_service.c
void handle_property_set_fd(int fd)
{
	// 先接收TCP连接
	if ((s = accept(fd, (struct sockaddr*)&addr, &addr)) < 0) {
		return;
	}
	
	// 取出客户端进程的权限属性？？
	// 获取客户端的凭证信息(eg：PID\UID\GID)，验证客户端的身份和权限，从而实现更精细的访问控制。
	if (getsockopt(s, SOL_SOCKET, SO_PEERCRED, &cr, &cr_size) < 0) {
		return;
	}
	
	// 接收数据请求
	r = recv(s, &msg, sizeof(msg), 0);
	close(s);
	
	//和常规的通信协议数据处理很相似.
	switch(msg.cmd) {
		case PROP_MSG_SETPROP:
			msg.name[PROP_NAME_MAX-1] = 0;
			msg.value[PROP_VALUE_MAX-1] = 0;
			// 如果是`ctl`开头的消息，则认为是控制消息，用于执行一些命令
			//例如：adb shell setprop ctl.start bootanim 查看开机动画，abd shell setprop ctl.stop bootanim 关闭开机动画.
			if (memcmp(msg.name, "ctl.", 4) == 0) {
				// 检查控制权限
				if (check_control_perms(msg.value, cr.uid, cr.gid)) {
					handle_control_message((char*)msg.name+4, (char*)msg.value);
				} else {
					//检查客户端进程是否有足够的权限
					if (check_perms(msg.name, cr.uid, cr.gid)) {
						//调用property_set设置
						property_set((cahr*)msg.name, (cahr*)msg.value);
					}
				}
			}
			…
		default:
			break;
	}
}

// property_service.c
int property_set(const char* name, const char* value)
{
	…
	prop_area *pa;
	prop_info* pi;
	…
	//从属性存储空间中寻找是否已经存在该属性
	pi = (prop_info*)__system_property_find(name);
	if (pi != 0) {
		// 找到了对应属性
		// 如果属性名已`ro.`开头，则表示只读，不能设置，所以直接返回.
		if (!strncmp(name, "ro.", 3)) return -1;
		pa = __system_property_area__;
		//更新该属性的值
		update_prop_info(pi, value, valuelen);
		// 下面两句代码是干嘛的？
		pa->serial++; // ??
		__futex_wake(*&pa->serial, INT32_MAX);
	} else {
		// 如果没有找到对应的属性，则认为是增加属性，Android最多支持247项属性，超过则直接返回。PS：不知道最新的android版本是否放开限制.
		pa = __system_property_area__;
		if (pa->count == PA_COUNT_MAX) return -1;
		
		// 偏移？
		pi = pa_info_array + pa->count;
		pi->serial = (valuelen << 24);  // 干嘛的？
		memcpy(pi->name, name, namelen+1)
		memcpy(pi->value, value, valuelen+1)  // 开始向共享内存写属性属性数据了.
		…
		pa->count++;
		pa->serial++;
		__futex_wake(*&pa->serial, INT32_MAX);
	}
	// 还有一些需要特殊处理的属性，比如:net.change开头的属性
	if (strncmp("net", name, …)) {
		…
	} else if (… && strncmp("persist.", …)) {
		// persist.开头的属性，需要写入到对应的文件中.
		write_persistent_property(name, value);
	}
	/*
	     处理因属性变化，init.rc中需要执行的COMMAND，例如：
	    on property:persist.service.adb.enable=1
		start adbd
	    当persist.service.adb.enable属性设置为1后，就会执行start adbd这个命令
	*/
	// 上述逻辑是在该函数完成的.
	property_changed(name, value);
}
```
>这种请求处理方式，如果有多个client同时请求属性设置，或者单个client频繁的请求属性设置，不会产生属性设置的时延问题吗？

再看看客户端如何发送属性请求的，如下：
```c
// properties.c
// 客户端通过property_set发送请求，该接口由libcutils库提供.
int property_set(const char* key, const char* value)
{
	prop_msg msg;
	…
	msg.cmd = PROP_MSG_SETPORP; // 设置消息码
	strcpy((char*)msg.name, key);
	strcpy((char*)msg.value, value);
	…
	return send_prop_msg(&msg);
}

static int send_prop_msg(prop_msg* msg)
{
	// 建立和属性服务器的socket连接
	s = socket_local_client(PROP_SERVICE_NAME, …, SOCK_STREAM);
	if (s < 0) return -1;
	// 通过socket发送出去
	while ((r = send(s, msg, sizeof(prop_msg), 0)) < 0) {
		…
	}
	if (r == sizeof(prop_msg)) {
		r = 0; // 发送成功
	} else {
		r = -1; // 发送失败
	}
	close(s);
	
	// 返回发送成功还是失败.
	return r;
}
```
>这种每次设置属性，都是同步操作，都需要创建socket、连接服务器、然后发送，不需要考虑时延的问题吗？


到此本章节的内容就结束了，相对还是比较简单的。

***
--------------------------------------------------------------结构----------------------------------------------------------------------

+ init启动   
	+ init.rc解析  
		- rc语法与解析  
		- zygote启动  
	+ 属性服务启动  
		- 属性空间的初始化  
		- client如何“读”属性空间  
		- socket的创建和消息处理  
		- client如何“写”属性空间
***
--------------------------------------------------------------反馈----------------------------------------------------------------------
+ 开机动画的流程是什么样的？如何替换开机动画？
+ Uevent干嘛的呢？
+ linux通过监听对端文件描述符，获取对端事件，感觉好神奇，有时间了解下内部原理。
+ init进程启动过程中，会和那些其他进程产生交互？交互啥数据？
+ init进程启动完成后，在运行期间会和那些其他进程产生交互？交互啥数据？ 
