## 概述

从不知道什么地方找到这张android系统启动的架构图，如下：

![image](https://github.com/user-attachments/assets/8530c6cf-6d81-4acd-b006-5e24ee61f860)

从上图看到，`init进程`是C++ Framework（Native）的起点和创造者，而`zygote`就是Java Framework的起点和创造者。PS：zygote(受精卵)，"名副其实"

本章节将较详细的介绍`zygote`的整体工作流程。

## zygote分析

zygote是init进程经由`int.rc`启动的`native`进程，zygote其实是一个`"别名"`，它在`Android.mk`中的名字其实叫`app_process`，为什么这么做，不得而知。

```c
// zygote"换名"的代码
// App_main.cpp
int main()
{
	…
	// 设置本进程的名字为zygote，这样ps看到的进程名就是zygote
	// 原理是调用linux的pctrl系统调用`prctl(PR_SET_NAME, "zygote", 0, 0, 0);`
	set_process_name("zygote");
	…
}
```
### AppRuntime分析
`App_main.cpp`中有一个结构体:`AppRuntime`，它是`AndroidRuntime`的派生类，是`zygote`的一个**核心类**。
>这里感觉作者讲的不够好，没有先介绍AppRuntime和AndroidRuntime的功能作用，就接着上代码了，比较突兀.
>>从命名方式可以看出，它和整个android，尤其是java/app的运行环境有关，可以猜到必然涉及虚拟机、JNI等流程的初始化

`AppRuntime`的核心接口有`start`、`onStarted`、`onZygoteInit`、`onExit`等等。
```cpp
// AndroidRuntime.cpp
// start接口，在app_main.cpp中的main接口中被调用.
void AndroidRuntime::start(const char* className, const bool startSystemServer)
{
	…
	// ① 创建虚拟机
	if (startVm(&mJavaVM, &env) != 0)
		goto bail;
	
	// ② 注册JNI函数
	if (startReg(env) < 0)
		goto bail;
		
	…
	
	// 从函数名（start）和函数参数（className）看，接口的作用可能和函数调用有关系。
	// 此处的classname为"com.android.internal.os.ZgoteInit"
	// 获取java ZgoteInit类
	startClass = env->FindClass(slashClassName); 
	// 找到ZgoteInit的static main函数
	startMeth = env->GetSaticMethodID(startClass, "main", "([Ljava/lang/String)V");
	…
	
	// ③ 通过JNI调用java ZgoteInit的main接口，此时zygote从C++世界，进入如java世界
	env->CallStaticVoidMethod(startClass, startMeth, strArray);
	…
}
```
从`AndroidRuntime::start`的代码分析，可以找到三个关键点：虚拟机创建、JNI注册、zygoteInit.main执行，这三者是开创Java世界的三部曲。

### 创建虚拟机
```cpp
// AndroidRuntime.cpp
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv)
{
	// 该函数绝大部分代码都是设置虚拟机的参数.
	// 这里作者提到两个比较重要的参数.
	
	// 第一个JNI Check，它的作用是对JNI中字符串进行格式检查（是否符合UTF-8要求）以及检查资源是否被释放。
	// 该参数只在eng调试版本有效，因为一方面它的检查比较耗时，会影响系统运行速度，另一方面有效检查很严格，比如字符串检查，一旦出错，则调用进程就会abort
	// 可以通过属性"dalvik.vm.checkjni"/"ro.kernel.android.checkjni"，对JNI Check进行开关.
	checkJni = true;
	
	// 第二个是设置虚拟机的heapsize，默认是16MB，绝大多数厂商会修改该值，设置为32M.
	// 这里作者提到，能否根据不同的app程序，动态的设置虚拟机的heapsize呢？
	// 比如需要加载高清大尺寸图片的app，分配大一点的heapsize，只显示文本的app，分配小一点的heapsize？
	// 该值也可以通过属性"dalvik.vm.heapsize"配置
	opt.optionString = heapsizeOptsBuf;
	
	// 创建虚拟机
	// 这里有个疑问，JNI不是虚拟机支持的操作吗，此时虚拟机还没有创建，就可以直接调用JNI的代码了？
	// 看了内部实现，好像和JNI没啥关系（它是jni.h的一个接口），内部好像就是创建了一个VM相关的对象？
	JNI_CreateJavaVM(…);
}
```
### 注册JNI函数
我的理解，JNI机制有对应的库实现，而虚拟机是该库的中间调度者，从而实现java<->C++之间的互相调用。
前面的章节提到过JNI有动态注册和静态注册，JNI接口注册过后，才能正常的调用。
在zygote的java事件创建过程中，需要用到的一些函数是采用native方式实现的，所有需要提前的注册，以便后续的调用。
```cpp
// AndroidRuntime.cpp
int AndroidRuntime::startReg(JNIEnv* env)
{
	…
	// 注册JIN函数，gRegJNI是一个全局数组.
	register_jni_procs(gRegJNI, …);
	…
}

// gRegJNI声明static const RegJNIRec gRegJNI[] = {
	REG_JNI(register_android_debug_JNITest),
	REG_JNI(register_com_android_internal_os_RuntimeInit),
	REG_JNI(register_android_os_SystemClock),
	REG_JNI(register_android_util_EventLog),
	REG_JNI(register_android_util_Log),
}
```

### Welcome to Java World

zygote通过jni最终调用`com.android.internal.os.ZygoteInit`的main函数，下面是这个入口函数：
```java
// ZygoteInit.java
public static void main(String argv[]) {
	// 咱们看看这个java世界的入口函数，做了什么。
	…
	// ① 注册zygote用的socket，谁是socket的client和server端？干嘛用的呢？
	registerZygoteSocket();
	
	// ② 预加载类和资源，加载资源可以理解，加载类什么鬼？
	preloadClasses();
	preloadResources();
	…
	// 强制执行一次垃圾回收，java世界刚开始创建，就gc的目的是？吃饭前先洗洗碗？
	gc();
	…
	// ③ 启动system_server进程
	startSystemServer();
	…
	// ④ 干嘛的呢？
	runSelectLoopMode();
	…
	// 怎么又关闭socket了？
	closeServerSocket();
	…
	// ⑤ 出现异常时调用，干嘛的？
	caller.run();
	…
}
```
书中对`ZygoteInit`中的`main`函数中的关键5个地方一一进行了分析，如下：

**1.建立IPC通信服务端**
zygote及系统中其他程序的通信没有使用`Binder`（到现在为止Binder服务起来了吗？），而是采用基于AU_UNIX类型的Socket。
```java
// ZygoteInit.java
// 这个函数的作用就是建立socket，用于??和??的通信？
private static void registerZygoteSocket()
{
	int fileDesc;
	// 从环境变量中获取socket的fd，这个变量在execv启动zygote时传入？
	String evn = System.getenv(ANDROID_SOCKET_ENV);
	fileDesc = Integer.parseInt(env);
	…
	// 创建服务器端socket，监听来自client的请求？app？java service？
	sServerSocket = new LocalServerSocket(createFileDescriptor(fileDesc));
}
```
**2.预加载类和资源**
```java
// ZygoteInit.java
private static void preloadClasses()
{
	…
	private static final String PRELOADED_CLASSES = "/system/etc/preloaded-classes"; 
	…
	// 预加载类的信息存储在PRELOADED_CLASSES
	InputStream is = ZygoteInit.class.getClassLoader().getResourceAsStream(PRELOADED_CLASSES);
	…
	BufferedReader br = new BufferedReader(new InputSteamReader(is), 256);
	// 读取文件的每一行，忽略#开头的注释行。文件？什么文件？里面什么内容？
	while ((line = br.readLine()) != null) {
		line = line.trim();
		if (line.startWith("#") || line.equals("")) {
			continue;
		}
		…
		// 通过java反射来加载类，line中存储的是预加载的类名
		//反射是种啥机制？为啥能凭一个类名字符串，就把类加载了？
		Class.forName(line);
		…
	}
}
```

这里看下预加载的类的文件内容，需要加载的类有`上千个`，都加载上来，估计要花不少的时间。
```txt
kona:/system/etc $ cat preloaded-classes | head -n 30
#
# Copyright (C) 2017 The Android Open Source Project
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# Preloaded-classes filter file for phones.
#
# Classes in this file will be allocated into the boot image, and forcibly initialized in
# the zygote during initialization. This is a trade-off, using virtual address space to share
# common heap between apps.
#
# This file has been derived for mainline phone (and tablet) usage.
#
android.R$styleable
android.accessibilityservice.AccessibilityServiceInfo$1
android.accessibilityservice.AccessibilityServiceInfo
android.accounts.Account$1
android.accounts.Account
android.accounts.AccountManager$10
android.accounts.AccountManager$11
…
```
`preload_class`文件是由`framework/base/tools/preload`工具生成，它需要判断每个类加载的时间是否大于1250毫秒，这是怎么判断的？
超过这个时间的类，就会被写到`preload-classes`文件中，最后由`zygote`预加载。
作者指出，`preloadClass`函数的执行时间过长，也是导致Android系统启动慢的原因之一。

`preloadResource`资源的加载，主要是加载`framework-res.apk`中的资源，啥内容啊？整体流程和`preloadClass`类型，这里没有做过多的介绍。
问了`chatGPT`，一般资源包括：`布局文件、字符串资源、图片资源等等`

**3.启动system_server**
`system_server`是framework的核心，如果它死了，会导致`zygote`自杀。
```java
// ZygoteInit.java
private static boolean startSystemServer() 
{
	// 设置参数，有点像启动参数.
	String args[] = {
		"--setuid=1000",
		"--setgid=1000",
		…
		"--nice-name=system_server", // 进程名：system_server
		"com.android.server.SystemServer", // 启动的类名
	};
	// 将字符串数组参数转换为Arguments对象
	ZygoteConnection.Arguments parsedArgs = new ZygoteConnection.Arguments(args);
	…
	// fork一个子进程，这是Zygote在java世界第一个fork的进程吗？
	pid = Zygote.forkSystemServer(parsedArgs.uid, parsedArgs.gid, …);
	…
	if (pid == 0) {
		// system_server进程的工作
		handleSystemServerProcess(parsedArgs);
	}
}
```
**4.有求必应之等待请求**
从标题就知道，这个步骤和之前创建的socket有关系，希望可以解答我们的一些疑问，代码如下：
```java
// ZygoteInit.java
private static void runSelectLoopMode()
{
	…
	ArrayList<FileDescriptor> fds = new ArrayList();
	// sServerSocket zygote java main开头创建的socket
	fds.add(sServerSocket.getFileDescriptor());
	…
	while (true) {
		fdArray = fds.toArray(fdArray);
		// selectReadable内部调用select，使用多路复用I/O模型，当有客户端连接或者有数据时，该函数会返回。
		//在新版本中，貌似没有再使用select
		int index = selectReadable(fdArray);
		…
		else if (index == 0) {
			// 表示有客户端连接过来，客户端在zygote的"socket"抽象为ZygoteConnection。
			ZygoteConnection newPeer = acceptCommandPeer();
			peers.add(newPeer);
			fds.add(newPeer.getFileDesciptor());
		} else {
			// 表示客户端发送了请求，一般是什么请求呢？
			// peers.get返回的是ZygoteConnection，后续处理交给runOnce函数完成.
			done = peers.get(index).runOnce();
		}
	}
}
```
### 关于Zygote的总结
书中采用的第一~七七创世风的总结，我这里换一种总结方式：
>zygote启动了虚拟机，这是java世界的根基，所有java程序运行的环境。
>zygote完成了jni注册，这样可以实现java和native和互相访问，从而触发`ZygoteInit.java:main`的调用
>zygote完成预加载工作(类/资源)，方便后续服务的启动和执行。
>zygote完成了`system_server`的创建，它是整个Zygote过程的核心
>zygote完成了`socket`的监听，用于客户端的请求监听处理。

>还是有点抽象，打个比方，盘古开天辟地（java世界），首先要创建一个“生命”能存活的“环境”(虚拟机/runtime)，
>java世界的创建/运行也依赖native世界的能力，因此需要建立两者之间的“通道”（jni注册），
>“生命”除了“环境”外，它的启动和执行还依赖于一些资源，所以要提前预加载（资源），
>盘古开天过程中，创造了一个“帮手”(system_server)，让它负责其他重要服务的创建，
>同时盘古还留下了一个“号角”（socket），当“生命”（java客户端程序）有请求时，通过它告诉“盘古”，从而处理请求。

故事讲得有点不伦不类，但是意思就是这么个意思。

## SyetemServer分析

SsytemServer简称`ss`，它是zygote的`嫡长子`，重要性不言而喻。

### SystemServer的诞生

通过前面的代码，我们知道`SS`是由`Zygote.forkSystemServer`函数创建的，这里看看它的实现：
```c
// 不出意外，它是一个native函数。PS：想想也对，需要调用C/C++的fork/waitpid等接口，必须会和native打交道。
// dalvik_system_Zygote.c
static void Dalvik_dalvik_system_Zygote_forkSystemServer(..)
{
	pid = forkAndSpecializeCommon(args)l
	if (pid > 0) {
		gDvm.systemServerPid = pid;  // 保存system_server的进程id
		// 函数退出前先检查刚创建的子进程是否退出了.
		if (waitpid(…) == pid) {
			// 如果system_server退出了, Zygote直接干掉自己，矮油，生死与共啊。
			kill(getpid(), SIGKILL);
		}
	}
}

static pid_t forkAndSpecializeCommon()
{
	…
	// 设置信号处理
	setSignalHandler();
	
	// 创建子进程
	pid = fork();
	if (pid == 0) {
		// …
	}
}

static void setSignalHandler()
{
	…
	// 设置信号处理函数，该信号是子进程死亡的信号
	err = sigaction(SIGCHLD, &sa, NULL);
}

static void sigchldHandler()
{
	…
	while ((pid == waitpid(-1, …))) {
		// 如果死去的子进程是SS，则Zygote把自己也干掉。PS：这俩真是好基友。
		if (pid == gDvm.systemServerPid) {
			kill(getpid(), SIGKILL);
		}
	}
}
```
我们来看看`zygote`用`肋骨`创建的`夏娃`到底有多么重要的使命：
```java
// ZygoteInit.java
private static boolean startSystemServer() 
{
	…
	pid = Zygote.forkSystemServer();
	if (pid == 0) {
		// 这里就是SS的进程，工作都在下面这个函数啊
		handleSystemServerProcess(parsedArgs);
	}
}

private static void handleSystemServerProcess()
{
	// 关闭从Zygote哪里继承下来的Socket
	closeServerSocket();
	// 设置SS进程的一些参数
	setCapabilities(..);
	// 调用ZygoteInit函数
	RuntimeInit.zygoteInit(…);
}
```
system_server是一个进程，进程的入口自然是`main`函数，到现在还在入口外面打转，干嘛这么绕啊。
```java
// RuntimeInit.java
public static final void zygoteInit(…)
{
	// 常规的初始化
	commInit();
	// navtive层初始化。PS：话说system_server怎么扯到了zygote的初始化？zygote的初始化不应该在它自己的main函数调用吗？
	zygoteInitNative();
	…
	// 通过下面的接口，调用"com.android.server.SystemServer"的main函数，终于到main函数了，这种调用方式蛮奇怪的.
	invokeStaticMain(…);
}
```
好家伙，`zygoteInitNative`还是一个native函数，如下：
```c++
// AndroidRuntime.cpp
static void com_android_internal_os_RuntimeInit_zygoteInit(…)
{
	// gCurRuntime是AppRuntime的实例对象，它是Zygote中定义的变量，既然system_server是Zygote的子进程，自然也继承了该变量？
	gCurRuntime->onZygoteInit();
}

class AppRuntime : public AndroidRuntime
static AndroidRuntime* gCurRuntime = NULL; // 全局变量
// 看看它的构造函数
AndroidRuntime::AndroidRuntime()
{
	SkGraphics::Init();  // Skia库初始化。PS：好意外，图形库的初始化竟然放在这里，不应该由专门的和图形强相关的java服务启动吗？
	…
	gCurRuntime = this;
}

// App_main.cpp
virtual void onZygoteInit()
{
	// 终于第一次见到和BInder相关的逻辑了.
	sp<ProcessState> proc = ProcessState::self();
	if (proc->supportsProcessess()) {
		proc->startThreadPool(); // 启动一个线程，用于BInder通信。
	}
}
```
到此，梳理一下思路，`system_server`进程启动后，调用了`zygoteInit`这个名字奇怪的函数，
该函数调用了`zygoteInitNative`创建了一个Binder通信线程，接着调用了`invokeStaticMain`调用system_server的main函数。
话说，`zygoteInit`为什么不叫`startSystemServer`？`zygoteInitNative`为啥不叫`startSystemServerBinder`等等更贴切自身功能的名字？
吐槽归吐槽，接着看重头戏，system_server的main函数。

```java
private static void invokeStaticMain(…)
{
	…
	// 这里用的反射机制吗？
	Class <?> cl;
	cl = Class.forName(className);
	…
	//创建了函数，那么什么时候被调用呢？
	Method m = cl.getMethod("main", new Class [] { ..} );
	…
	// 抛出了一个异常。PS：神经病啊，为什么不直接调用呢？
	throw new ZygoteInit.MethodAndArgsCaller(m, argv);
}
```
>真是小刀拉屁股，开眼了，通过抛出异常，然后在异常中包裹"函数指针"，再在某个地方，捕获这个异常，然后再调用"函数指针"。
>怎么理解呢，异常某种意义上，可以看做是一种跨函数的通信方式，可以从调用堆栈的某层，将数据传到栈底的某层。

>再打个比方，函数A调用函数B，函数B调用函数C，堆栈是一层叠着一层，叠了N层，此时再调用函数D，而函数D是一个重量级的函数，
>再在堆栈上叠上函数D的话，有点不堪重负了(爆栈溢出)，此时将调用直接通过异常跳转到函数A，清空函数A之后的堆栈，让函数D“轻装上阵”。

好累，绕了一大圈，终于真正的到system_server的main函数了，话说谷歌的程序员到底遇到了啥场景，不得不设计这样糟心的流程啊。
要是换了一家小公司是这样实现的，大家可能就骂脏话了，什么屎山代码，但是是谷歌写的，大家又开始思考其中有什么深意了。
PS：入关后，自有大儒为我辩经。

```java
// SystemServer.java
public static void main() {
	// 加载libandroid_servers.so，为JNI调用服务的.
	System.loadLibrary("android_servers");
	//调用native的init1函数。PS：这名字起的..
	init1(args);
}

//com_android_server_SystemServer.cpp
extern "C" int system_init();
static void android_server_SystemServer_init1(…)
{
	system_init();
}

extern "C" status_t system_init()
{
	…
	property_get("system_init.startsurfaceflinger", propBuf, "1");
	if (strcmp(propBuf, "1") == 0) {
		// SurfaceFlinger服务在system_server中启动
		SurfaceFlinger::instantiate();
	}
	// 就启动一个系统服务？其他服务呢？没写？
	…
	// 调用systemServer的init2函数
	runtime->callStatic("com/android/server/SystemServer", "init2");
	
	// binder通信相关
	ProcessState::self()->startThreadPool();
	IPCThreadState::self()->joinThreadPool();
}

// SystemServer.java
public static final void init2() {
	Thread thr = new ServerThread();
	thr.setName("android.server.ServerThread");
	thr.start(); // 启动一个ServerThread
}

// SystemServer.java::ServerThread
public void run() {
…
	// 启动Entropy Service
	ServiceManager.addService("entropy", new EntropyService());
	// 启动电源管理服务
	power = new PowerManagerService();
	ServiceManager.addService(Context.POWER_SERVICE, power);
	…
	// 初始化看门狗
	Watchdog.getInstance().init(…);
	…
	// 启动WindowsManager服务
	…
	// 启动ActivityManager服务
	…
	// 进入消息循环，处理各种消息.
	Looper.loop();
}
```
>目前看，system_server创建各种系统重要服务的方式，无非是创建对应服务类对象而已？这样服务就创建了？没怎么明白。

### 关于SystemServer的总结

上午追了那么多代码，我现在有印象的就只有一句话：
>system_server是zygote通过fork创建的进程，在这个进程的入口main函数中，启动了很多系统重要的服务.
至于其他的代码细节，记不住，也没人记得住。

关于上面的疑问，咨询了chatGPT，它的回答如下：
>与其说systemServer创建了很多服务，不如说是注册了很多服务，这些服务都驻留在system_server进程，
>注册过程，涉及到binder相关逻辑，每个服务都会分配一个binder接口？上层服务通过binder的形式与这些服务进行通信。

所以书中的`启动/创建`服务的字眼比较容易让人疑惑，将其理解为`注册`就比较清晰了。

## Zygotye的分裂

Zygote(受精卵)，目前就"孵化(分裂)"了一个进程，有点名不副实啊，记得Zygote启动过程中会创建一个socket，并且会使用select监听该socket的事件。
>那么谁会向Zygote发送请求呢？Zygote又会在什么情况下，再次“孵化（分裂）”呢？

书中以一个Activity的启动为例，分析Zygote如何分裂和繁殖的.

### ActivityManagerService发送请求

这块的代码比较复杂，在看另一本书时，也详细的分析过，这里不再重复，详细的代码细节可以参考《Android系统源代码情景分析》的第七章

>简单来说，就是APP -> ActivityManagerService -> SystemServer -> Zygote -> fork创建进程

我的理解是，打开一个app，就是luancher进程，发送启动Activity请求给SystemServer的ActivityManagerService，
AMS经过一系列的处理，会发送请求给Zygote进程，Zygote会为该APP创建进程，并在该进程中打开Activity.

### 有求必应之响应请求

这里介绍了Zygote接收到请求后的代码流程，这部分逻辑之前看另一本书时也详细分析过，可参考：《Android系统源代码情景分析》的第七章

## 扩展思考

### 虚拟机heapsize的限制

作者认为无法动态为每个APP分配堆内存的原因是，每个APP进程都是zygote孵化的子进程，该子进程会继承父进程的虚拟机配置，其中包括了堆内存的大小，
也就是说zygote和其他app进程是共享一份虚拟机heapseize配置的，无论谁修改了该值，其他的app都共享该值。

解决的办法无非是确保每个app进程，都有一份独立的heapsize配置。

### 开机速度优化

从init/zygote的流程看，可能会导致开机慢的原因有下面几个：
+ init进程解析**.rc时，启动各种服务，可能会产生耗时操作.
+ init进程解析**.rc时，文件I/O操作，可能会产生耗时操作.
+ zygote进程预加载的阶段，加载类和资源了可能会产生耗时操作.
+ SystemServer注册各种系统服务时，会先创建对应的示例，这个过程也可能会产生耗时操作.

除了上面几个，作者提到在开机启动过程中，会对系统内的所有apk文件进行扫描并收集信息，这个动作耗时的时间比较长，为啥init和zygote章节都没有提到这个事？？

作者还提到可以移除掉预加载类步骤来加快开机流程，但是这样会牺牲APP进程的启动时间和内存占用。
如果zygote已经预加载了类，当fork app子进程时，因为两者的数据共享，子进程直接拷贝父类的响应数据即可，不需要自己再重新加载类，这样就提高了APP启动速度。
同时根据copy-on-write机制，在类使用时才拷贝，还能进一步降低APP进程的内存占用。
同时APP启动时高频高体验操作，系统启动是超低频操作，应该牺牲哪个也就一目了然了。

### watchdog分析

watchDog的中文含义是`“看门狗”`，它参考了硬件设计上的概念，早期硬件设备设备运行不稳定，需要经常跑去看看硬件有没有出问题，
因此专门设计了一个硬件看门狗，每隔一段时间，看门狗都会去检查一下`某个参数是否被设置`？？如果发现该参数没有被设置，
则判断为系统出错，然后就会强制重启。
>有点类似心跳包，只要没有收到心跳包，说明心跳停止了，系统自然是出大问题了。

Android中也给一些重要的系统服务设置了看门狗，一旦这些系统服务出现异常，就会杀掉system_server，同时出发zygote自杀，最终由init重启整个java世界。

watchDog的使用流程总结为三个步骤：
+ Watchdog.getInstance().init()
+ Watchdog.getInstance().start()
+ Watchdog.getInstance().addMonitor()

**1.创建和初始化Watchdog**
```java
// Watchdog.java
public static Watch getInstance() {
	if (sWatchdog == null) {
		sWatchdog = new Watchdog();
	}
	return sWatchdog;
}

public class Watchdog extends Thread
// Watchdog从线程类派生，所以会在一个单独的线程中执行。
private Watchdog() {
	…
	// 会构造一个Handler，进行消息处理.
}

public void init(…) {
	…
	mBootTime = System.currentTimeMillis(); // 得到当前时间.
	…
}

```
**2.看门狗跑起来**
 watchdog的start函数会触发run函数的调用，如下：
```java
// Watchdog.java
public void run() {
	…
	while (true) {
		mCompleted = false; // 表示各个服务是否检查完成.
		…
		// 给线程的消息队列发消息，用户请求Watchdog检查Service是否工作正常.
		mHandler.sendEmptyMessage(MONITOR);
		…
		// 检查两次后，如果还是有问题，则会杀死SS
		if (!Ddebug.isDebuggerConnected()) {
			Process.killProcess(Process.myPid());
			System.exit(10); // 杀掉.
		}
		…
	}
}
```
run函数的逻辑看起来比较简单，就是每隔一段时间给另外一个线程发送一条MONITOR消息，
哪个线程会检查各个Service的健康情况，而看门沟会等待检查结果，如果第二次还没有返回结果，那么它就会杀掉system_server.

>这个角度看Watchdog更类似一个服务，其他服务通过watchdog的接口"注册"，watchdog就会定期的检查这些服务了。

**3.队列检查**
系统服务中，共有三个Service需要交给Watchdog检查：AMS、PMS、WMS。
检查的地方在`HeartbeatHandler`的handleMessage中，代码如下：
```java
// Watchdog.java
final class HeartbeatHandler extends Handler {
	@override
	public void handleMessage(Message msg) {
		switch(msg.what) {
			…
			case MONITOR: {
				…
				long now = SystemClock.uptimeMillis();
				final int size = mMonitors.size();
				// 挨个检查每个服务
				for (int i = 0; i < size; i++) {
					mCurrentMonitor = mMonitors.get(i);
					// 调用monitor函数，每个服务（service）需要自行实现该接口
					mCurrentMonitor.monitor();
				}
				
				// 如果没问题，则设置mCompleted为真.
				synchronized (Watchdog.this) {
					mCompleted = true;
					mCurrentMonitor = null;
				}
			}
		}
	}
}
```
这里以PMS为例，看看如何"注册"watchdog检查，代码如下：
```java
// PowerManagerService.java
PowerManagerService()
{
	…
	// 在构造函数中把自己添加到watchDog的检查队列
	Watchdog.getInstance().addMonitor(this);
}

public void monitor() {
	// 在这里自行实现对service的检查.
	synchronized (mLocks) { … }
}
```
整个watchDog的实现和使用都是非常简单，不做过多介绍。

## 本章小结

略

***
------------------------------------------------------------------------------结构------------------------------------------------------------------------------
***
------------------------------------------------------------------------------反馈------------------------------------------------------------------------------
+ init启动的serverManager是干嘛的？它和SystemServer有关系吗？
+ 虚拟机到底是啥？是一个进程？
+ 预加载类的目的是什么？为了后续调用时的性能表现？
+ 需加强对反射的理解。
***
------------------------------------------------------------------------------练习------------------------------------------------------------------------------
