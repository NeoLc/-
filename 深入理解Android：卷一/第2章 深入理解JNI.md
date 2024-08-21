## JNI基本概念

JNI是（Java Native Inteface）的缩写，JNI技术是为了实现Java对native代码(非java实现)的调用。  
在其他的领域，也有类似的技术，比如：js调用c++、python调用c++、rust调用c++。

## Java层的JNI操作示例

JNI的一般调用流程为：java code -> jni lib -> native lib.

其中jni的命名可以是`任意`的，不过android一般`约定`为：`lib模块名_jni.so`的格式.

其中模块名指的是native的模块名？，比如调用的native lib为`libmedia.so`，则调用该native库的jni库命名为：`libmedia_jni.so`？

jni库必须为动态库，这样虚拟机才能加载它？

**java常规的使用JNI的示例**

```java
// 貌似没看到"头文件"？
public class MediaScanner
{
           // 之所以放在静态语句中，我猜是因为static语句最先执行，这样可以确保任意native在调用前，jni库都已经加载好了。
	static {
		// ① 加载jni库，"media_jni"在内部会扩展为"libmedia_jni.so"
		System.LoadLibrary("media_jni");
		// 调用native接口.
		native_init();
	}
	
	// ② native是java关键字，表示这是一个native接口.
	private static native final void native_init();
	
	//  正常java接口是没有native关键字修饰的.
	public void scanDirectories(..)
}
```

从代码示例看，java使用jni的操作主要有两个重要部分：一是JNI库的加载、二是native接口的声明。
完成这两步后，就可以java就可以调用native代码了？

### 加载JNI

作者没细说如何加载JNI库，也没说JNI的放置在哪里。
只说了不管jni在什么时候加载，在哪里加载，只要能确保native接口调用之前，jni库加载成功即可。

### java的native函数

也没有细说，只感叹下JNI技术对Java程序员很友好。

## JNI层实现示例

以mediaScanner为例
```java
// android_media_MediaScanner.cpp
static void android_media_MediaScanner_native_init(JNIEev *env)
{
	Jclass claszz;
	Clazz = env->FindClass("android/media/MediaScanner");
	
	fields.context = env.GetFieldID(clazz, "mNativeContext", "I");
	
	return;
}
```
`natvie_init`函数位于`android.media`这个包中，它的全路径名应该是`android.media.MediaScanner.native_init`，
而`native_init`在JNI层的实现函数名字为`android_media_MediaScanner_native_init`，好像就是把"."变成了"_"，
这是因为在Native语言中"."有特殊的含义(啥含义，我怎么不知道？)，所以替换成了"_"。
所以通过这种`命名规律`，java中的native函数，可以容易找到在JNI层对应的实现函数.

JNI文件格式，一般是`packagename_classname.*`的形式，
`android_media_MediaScanner.cpp`，`android.media`为包名，`MediaScanner`为类名.

说实话，这段代码对Java程序员就不太友好了，最直接的疑问是：`native_init`是如何调用到`android_media_MediaScanner_native_jni`的呢？
从软件上讲，简单的实现就是`嵌套`或`映射`，但是一个是java一个是C++，咋关联起来的呢？

### 注册JNI函数

Java中通过`注册`来解决这个问题，`注册`的作用就是：**将java的native函数和JNI层的对应实现函数关联起来**。

**完成注册过后，java的native调用就可以顺利的转到JNI层对应的函数中执行了。**

JNI函数的注册方法有两种：

+  静态方法

静态方法与其说是静态，不如说是`手动`方法，步骤如下：
	1. 先编译java文件，生成class文件
	2. 使用`javah`，命令如:`javah -o packagename_classname packagename.classname`
	这样会生成一个`packagenam_classname.h`文件(参考android_media_MediaScanner.h)，里面包含了对应的JNI函数声明
	3. 实现`packagenam_classname.h`中的JNI函数.

>然后呢？写完`cpp`之后呢？不需要编译？怎么编译？编译后的JNI库放在那里？注册呢？静态，静在哪里呢？作者好像没有说明？

在这种方法下，当Java调用native_init函数中，会自动到JNI库中，查找`android_media_MediaSacnner_native_init`这个函数(符号)，
找到之后，`虚拟机`会将JNI库中的这个符号(其实就是函数指针)和java中的`native_init`关联起来，这样java调用`native_init`时，
等价于调用`android_media_MediaSacnner_native_init`，整个的流程，都是通过`虚拟机`实现的。

所以，所谓的静态注册就是：**名字按格式转换、到jni库找对应符号、将java native函数和找到的jni函数指针绑定在一起，这样就实现注册了？**

+  动态方法

如果所谓的`注册`就是将java native函数和JNI函数一一对应起来，那么干脆写一个`数据结构`来实现映射好了。
java中这个结构体叫做`JNINativeMethod`，它的定义如下：
```java
typedef struct {
	// Java中Native函数的名字，eg:"native_init"
	const char* name;
	
	// java函数的签名信息(参数类型/返回值类型的组合)，字符串表示。
	const char* signature;
	
	// JNI层对应函数的函数指针.
	void* fnPtr;
} JNINativeMethod;
```
从该结构的定义(c/c++格式)，就知道它是在JNI层使用的，示例用法如下：
```java
static JNINativeMethod gMethods[] = {
	…
	{
		"native_init",
		"()v",
		(void*)android_media_MediaScanner_native_init
	}
	…
}

// 注册JNINativeMethod数组
Int register_android_media_MediaScanner(JNIEnv *env)
{
	// 调用AndroidRuntime的registerNativeMethods函数，内部又通过JNI调用了"RegisterNatives"完成动态注册工作.
	return  AndroidRuntime::registerNativeMethods(env, "android/media/MediaScanner", gMethods, NELEN(gMethods))
}
```

>这里有个疑问，JNI的.h/.cpp文件谁来生成呢？使用了`JNINativeMethod`后怎么就完成注册了呢？
>`register_android_media_MediaScanner`这个接口谁来调用呢？没感觉比静态注册简单啊？

作者给出的答案，我没怎么看明白，当java层调用`System.LoadLibrary`加载完JNI动态库后，紧接着会调用该库中一个叫`JNI_Onload`的函数，
如果有，则调用它，动态注册的工作就在这个函数内完成的？还需要再实现`JNI_Onload`接口？啥玩意呢？绕这么一圈？
```java
// android_media_MediaPlayer.cpp
Jint JNI_Onload(JavaVM* vm, void* )
{
	…
	// 动态注册MediaScanner的JNI函数
	if (register_android_media_MediaScanner(env) < 0) {
	  …
	}
	…
	return JNI_VERSION_1_4; // 必须返回这个值，否则会报错？
}
```

>不能这样设计吗：程序员定义需要java native调用的接口，然后通过gen工程生成.h/cpp文件, 手动的实现cpp文件，编译生成so
>只要java加载了这个so，在调用native接口时，虚拟机察觉到该调用时native接口，
> 则自动到jni so中查找对应的符号，然后执行对应的函数，不是挺简单的吗？ 
>离大谱，貌似静态注册就是这样的…

再回顾下作者为什么觉得动态注册比静态注册好：
+ 每个java生成的class文件，都得使用`javah`生成头文件       -- 动态注册不需要吗？它的头文件怎么生成的？
+ java`初次`调用native都需要搜索对应的JNI名字，影响效率  -- 这个理由很充分,搜索可能是N复杂度的.

>这里有个疑问，动态注册是双向的吗？java调C++和C++调用java，都会使用到这个注册表(JNINativeMethod)？
>动态注册是不是jni的函数名字就可以随意写了，不需要再遵守`packagename_classname_functionname`的格式？？

### JNI标准使用流程

官方的示例：https://github.com/vitaut-archive/android-ndk-example/tree/master/example

使用的流程就两步：
+ 编写jni cpp文件，然后编译成jni库。PS：静态注册的话，JNI命名要严格按照要求，可以不写`.h`
+ java中load该库，然后使用native接口即可

>PS：为什么在官方的示例中没有看到动态注册的痕迹

我自己在`unbuntu`上也尝试jni的使用，过程如下：

**1.实现HelloJNI.java**
```java
// 因为单个java文件调用jni，而不是package，所有没有进行类似`package com.example.HelloJNI`的声明
public class HelloJNI {                     
    public native void sayHello();          
                                            
    static {                                
        System.loadLibrary("hello_jni");    
    }                                       
                                            
    public static void main(String[] args) {
        HelloJNI helloJNI = new HelloJNI(); 
        helloJNI.sayHello();                
    }                                       
}                                           
```
**2.生成HelloJNI.h**
```sh
// java11需采用这种的方式生成，直接使用`javah -cp . HelloJNI`会报错，找不到对应的class
javac -cp . -h . HelloJNI.java
```
**3.实现HelloJNI.cpp**
```cpp
#include "HelloJNI.h"                                           
#include <iostream>                                             
                                                                
extern "C" {                                                    
                                                                
JNIEXPORT void JNICALL Java_HelloJNI_sayHello(JNIEnv *, jobject)
{                                                               
    std::cout << "hello jni" << std::endl;                      
}                                                                                                                               
}                                                               
```
**4.生成jni库**
```sh
// 这里使用了ndk的工具链，使用系统默认的会报很多编译错误
/home/lichuang/code/soft_tools/android-ndk-r26b//toolchains/llvm/prebuilt/linux-x86_64/bin/x86_64-linux-android32-clang++ -fPIC -shared -o libhello_jni.so HelloJNI.cpp -I/home/lichuang/code/soft_tools/android-ndk-r26b/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/include/
```
**5.运行**
```sh
// 运行前，需要将依赖库拷贝过来
cp ./toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/x86_64-linux-android/libc++_shared.so ~/neo/tmp/jni_test/
cp ./toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/x86_64-linux-android/32/libm.so ~/neo/tmp/jni_test/
cp ./toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/x86_64-linux-android/32/libc.so ~/neo/tmp/jni_test/
cp ./toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/x86_64-linux-android/32/libdl.so ~/neo/tmp/jni_test

// 开始运行
LD_LIBRARY_PATH=./ java HelloJNI
```
**6.虚拟机崩溃**
```sh
// 崩溃信息
# # A fatal error has been detected by the Java Runtime Environment:
#
# SIGSEGV (0xb) at pc=0x00007f8eec60d040, pid=87881, tid=87882

// 运行失败，没找到原因，时间原因，没有继续定位
//怀疑和ABI兼容性，或者和版本兼容性相关

java -version：查看 Java 版本信息。
java -XX:+PrintFlagsFinal -version：查看 Java 虚拟机的所有可用参数及其默认值。
java -XshowSettings:all -version：查看 Java 虚拟机的所有配置信息。
```

还有比较常见的就是AOSP中对JNI的使用
参考：http://androidxref.com/2.2.3/xref/frameworks/base/media/jni/android_media_MediaPlayer.cpp#register_android_media_MediaPlayer

### 数据类型转换

java到C++的jni转换必然涉及到数据类型的转换，java数据类型分为：基本数据类型和引用数据类型。

>`j`开头的Native类型，我把它理解为java类型在JNI中的抽象

![image](https://github.com/user-attachments/assets/86d13131-273c-4845-8f3b-08b6442b78cb)

![image](https://github.com/user-attachments/assets/73f239b4-8265-4a5c-9ede-be0489d3adaf)

从对照表看，有个比较特殊的转换"java objects -> jobject"，java的对象类型，都转换为JNI中的`jobject`类型，有点类似`void*`的效果.

类型系统本身就是一门语言中最关键的部分之一，两个语言的类型转换，估计不简单哦。。。

### JNIEnv介绍

从前是的jni示例中，可以经常的看到`JNIEnv`，书中介绍它是一个`线程相关`的，`代表JNI环境`的结构体，下图是它的内部结构：

![image](https://github.com/user-attachments/assets/693a87fc-c935-4ebc-9cad-0e01b3c4a70f)

`JNIEnv`提供了一些JNI`系统函数`，指的是`JNI机制`本身支持的内部接口？`Env环境`有点`上下文`的意思

这个参数是哪里来的呢？书中的说法是虚拟机传过来的，当虚拟机把`java native`函数转换为`jni 函数`调用时，会传入该参数。
在前面提到`JNI_Onload`中有个参数`JavaVm`，使用该对象的接口`AttachCurrentThread`即可获取`本线程`对应的`JNIEnv`结构体。

>不太明白`JNIEnv`为什么会和`线程`扯上关系，我的理解是：JAVA如果有多个线程操作native函数，每个线程的数据都不一样，
>比如传给native的参数值不一样，这样在JNI层需要一个数据结构，保存各个线程的专用数据，我猜的，不知道对不对？？

### JNIEnv操作Jobject

所有的java对象对应到JNI层都是`Jobject`，对象的本质就是`数据+操作`，也就是`成员变量`和`成员函数`,
所以`JNIEnv`操作Jobject就是读写成员变量和调用成员函数，`JNIEnv`通过以下两个接口进行：

```
// jfiekdID表示java类的成员变量
jfiekdID GetFieldID(jclass clazz, const char* name, const char* sig);

// jmethodID表示java类的成员函数
jmethodID GetMethod(jclass clazz, const char* name, const char* sig);

// 其中`jclass`表示类名，`name`表示成员函数/变量的名字，`sig`表示签名信息，为啥需要签名？函数可能是因为重载，变量是为啥？
```

>其实从C++的角度看，成员变量/函数，和普通的变量和函数本质上没有多大的区别，无非是签名、参数、作用域等不同而已。
>将它们从类中提取出来，并不是多么难的事情，所以不用把JNI想的多么神奇，从而产生畏难情绪。

这两个接口的用法示例如下：

![image](https://github.com/user-attachments/assets/35e27e1b-ce80-4c76-b20d-21941b358d4b)

取出成员变量和成员函数之后，怎么样去使用它呢？ 

![image](https://github.com/user-attachments/assets/a895bb9a-f315-4fc0-b4cd-f075aa4dbe4f)

JNI提供了一系列的类似`CallVoidMethod`的函数，形式如下：
```
// type对应java函数的返回值类型, 如：CallIntMethod、CallVoidMethod
NativeType Call<type>Method(JNIEnv* env, jobject obj, jmethodID methodID, …) ;
// JNI还提供了用于调用java static的系列函数
NativeType CallStatic<Type>Method(…)
```

>怎么好像都是调JAVA的，调C++的呢？也简单说明下啊。。

前面说的都是操作jobject成员函数，操作jobject成员变量的JNI接口如下：
```
// 获取fieldID后，使用以下两个接口实现jobject成员变量的读写.
NativeType Get<type>Field(JNIEnv* env, jobject obj, jfieldID fieldID);
void Set<type>Field(JNIEnv* env, jobject obj, jfieldID fieldID, NativeType value);
```

书中列出了一些常见的`Get/Set`函数：
![image](https://github.com/user-attachments/assets/3723ed6f-a38e-4771-b15e-825db14d6eb6)

### jstring介绍

java中的String也是引用类型，但是因为它的使用频繁，所以JNI中单独创建一个`jstring`类型，来表示java中的`String`类型。
>感觉这个理由挺勉强的，字符串无论在哪个语言中，都是单独一个类型表示，JNI单独给它一个类型，也无可厚非

jstring的常见接口如下：
```java
// 创建一个jstring对象
jstring NewString(JNIEnv* env, const jchar* unicodeChars, jsize len);
// 其他类似的还有：NewStringUTF、GetStringChars、GetString
```

下面是jstring的一个使用示例：
![image](https://github.com/user-attachments/assets/7c4a9eee-bd38-4407-aff9-33a2119bd6a4)

### JNI类型签名

作者石锤了，签名是为了应对重载的场景，由参数和返回值类型共同组合而成，它的格式如下：
```
(参数1类型标识参数2类型标识..参数n类型标识)返回值类型标识
```
举个例子：
```java
void processFile(String path, String mimeType);

//对应的JNI函数签名
(Ljava/lang/String;Ljava/lang/String;Landroid/media/MediaScannerClient;)V

// V为返回值类型void
// () 对应函数的()，内部为参数类型标识
// 参数String为引用类型，格式为：`L包名;`，包名中的"."需替换为"/"，因此`Ljava/lang/String;`标识的是一个`java String`类型
// 非引用类型，不知道什么格式
```
>函数签名好麻烦，看着麻烦，写着麻烦，可以使用宏简化，或者使用`javap -s -p classfile`自动生成函数和成员的签名.

![image](https://github.com/user-attachments/assets/5401a1fc-bbc1-400b-9aa2-392cfee179f5)
![image](https://github.com/user-attachments/assets/4dfc4010-e0d8-4bd7-9e87-981927ac3275)

### 垃圾回收
java中的引用对象，当内部引入计数为0时，就会触发垃圾回收，释放掉。PS：当然内部不是这么简单粗暴的，但原理大差不差。

那么java中的引用对象，传递到JNI之后，C++又没有垃圾回收机制管理引用对象的内存生命周期，会产生什么影响吗？

书中的说法是可能会产生`野指针`，也就是说传递给JNI的引用对象，JNI内部还在使用呢，但是在java层该对象已经被垃圾回收释放了。
java和c++不太一样，好像没有`值拷贝`的概念，都是`引用`传递，JNI拿到java某个对象的引用，但是java层该对象的内存可能被释放掉了。

书中举个一个例子：
```cpp
static jobject save_thiz = NULL;  // 定义一个全局的object

static void android_media_MediaScanner_proccessFile(JNIEnv* env, jobject thiz, jstring path, jstring mimeType, jobject client)
{
	..
	save_thiz = thiz; // 这里是C++，并不会增加thiz这个java对象的引用计数。
	..
	return;          // jni 返回后，thiz这个java对象，在java层可能被垃圾回收
}

void callMediaScanner()
{
	// 这个时候再访问全局的save_thiz是有风险的，save_thiz可能是野指针.
}
```

还有一个例子，比较隐晦：
```cpp
// JNI中的构造函数，以java传入的引用对象为参数.
MyJNIClass::MyJNIClass(JNIEnv *evn, jobject client) 
{
	mClient = client;  // client赋值给成员变量.
}

MyJNIClass::process()
{
	// 此时操作mClient，存在风险，因为mClient引用的java对象可能已经别释放了.
}

```

为了解决这个问题，JNI提供了三种类型的引用，我的理解是在JNI层手动增加java对象的引用计数？
+ Local Reference：本地引用，JNI中，非全局的引用对象都是本地引用，我的理解是这是JNI默认的引用方式.
                                         传入JNI函数和函数中创建的对象都是本地引用，特点是一旦jni`函数返回`，该对象可能被垃圾回收。
+ Global Reference：全局引用，被引用的对象，如果不主动释放，永远也不会被释放。
+ Weak Global Reference：特殊的全局引用，可能被垃圾回收，在使用前，需调用JNIEnv的`IsSameObject`判断是否已回收，该引用很少使用。

书中给出了`全局引用(Global Reference)`的使用示例：
```cpp
// 构造函数
MyMediaScannerClient(JNIEnv* env, jobject client) : mEnv(env),
	// 调用NewGlibalRef创建一个Global Reference，这样mClient就不用再担心被回收了.
	mClinet(env->NewGlobalRef(clinet)),
{ 
	…
}

// 析构函数
~MyMediaScannerClient
{
	mEnv->DeleteGlobalRef(mClient);  // 手动释放这个全局引用
}
```

本地引用，书中给出的示例比较让人迷惑，我的理解是：**一个本地引用，除非你需要一直用到函数返回，否则当你不使用时，就应该立即手动释放。**
```cpp
bool sacnFile()
{
	for (int I = 0; I < 100; ++i) {
		jstring pathStr = mEnv->NewStringUTF(path);
		…
		// 立即释放Local Reference
		// 书中的说法，好像暗示这创建的100个jstring如果现在不释放，函数返回也会释放一样？好像不对吧？不会内存泄漏吗？
		// 而且函数返回自动释放，使用的不是C++的RAII机制吗？？
		// 这里pathStr每次循环离开作用于，不会自动调用jstring的析构函数，释放资源吗？？
		// 还是说有其他我不知道的JNI内部资源管理机制？？
		mEnv->DeleteLocalRef(pathStr);  
	}
}
```
### JNI中的异常处理

>JNI中的代码，给我感觉是C++，但又不是C++的感觉，奇奇怪怪。

无论是java还是C++的代码执行，都可以会抛出异常，那么JNI中的异常是如何处理的呢？

这里有个场景，当JNI中通过`JNIEnv`调用java对象的接口，抛出异常后如何处理？
JNI的处理是：**不产生中断并返回，而是等到JNI函数执行完成后，从JNI返回到java层后，虚拟机再抛出异常**，再由Java进行处理？？
这样也对，否则java对象的异常，JNI的C++代码怎么处理呢？
不过不立刻产生中断的话，后续的代码的执行就`非常危险`了，就好比打开文件失败异常，后面还操作文件句柄，非常容易让程序挂掉。

JNIEnv提供了一些接口，要缓解这个问题：
+ ExceptionOccured：判断是否发生异常？
+ ExceptionClear：用来清理当前JNI层中发生的异常？清理？清理资源？
+ ThrowNew：用来想java层抛出异常。

本小节的异常处理，细究下来估计比较复杂，作者讲的比较简单。
***
--------------------------------------------结构--------------------------------------------

JNI
	- JNI的定义和作用
	- JNI在java层的常规用法、基本步骤
	- JNI如何实现Java native接口、JNI层的函数命名规范.
	- JNI的注册本质及意义、两种注册方法
	- JNI的类型转换（Java type <-> JNI type <-> C++ Type）
	- JNI的常见类型(JNIVM、JNIEnv、JString..)
	- JNI的常见数据操作接口
	- JNI中的签名
	- JNI中的对象生命周期管理
	- JNI中的异常处理

>JNI的主线是java如何调用C++，以及C++如何调用到java，同时java的数据如何传输到C++，C++的数据如何传输到Java。
>>JNI的所有功能、概念、接口、类型，应该都是为这条主线服务的。

比如：jni注册，是为了完成java到C++的接口的映射转换，类型转换/数据操作接口都是为了数据的传输服务的，签名是为了正确的调用接口服务的.

>如果想深入的理解JNI，可以参考JDK文档《Java Native Interface Specification》，它完整和细致地阐述了JNI技术的各个方面，是深入学习JNI的权威指南。
***
--------------------------------------------反馈池--------------------------------------------
+ JAVA代码竟然能调用C++代码，这么神奇的事情是怎么做到的？原理是什么？
+ 标准的JNI使用流程是什么样的？如果看完一章节，JNI不会用就太搞笑了..
+ DeleteLocalRef什么时候需要手动调用呢？








