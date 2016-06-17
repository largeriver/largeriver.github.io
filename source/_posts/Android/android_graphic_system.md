---
title: "[摘]Android 图形系统 （Android 4.3）"
date: 2015-07-01 10:13:49
categories:
- android

tags:
- graphic
- framework
---

# 前言
本文摘自 《深入理解Android设计思想》

在Android4.3之前的版本中，最终的显示工作是要交给FrameBuffer的，而从4.3开始，合成工作由SurfaceFlinger交给硬件合成器来完成，显示工作将由硬件合成器HAL来完成。
参考链接：http://blog.csdn.net/xuesen_lin/article/details/8954508
<!-- more -->
# OpenGL ES与EGL
**SurfaceFlinger**虽然是GUI的核心，但相对于OpenGL ES来讲，它其实**只是一个“应用”**。

![](/img/android_graphic_system_01.png)
我们根据上面这个图，由底层往上层来逐步分析整个架构：

1. Linux内核提供了统一的framebuffer显示驱动，设备节点/dev/graphics/fb*或者/dev/fb*，以fb0表示第一个Monitor，当前实现中只用到了一个显示屏
2. Android的HAL层提供了Gralloc，分为fb和gralloc两个设备。

	* **fb 负责内核中framebuffer**的打开、初始化配置，以及提供post、setSwapInterval等操作
	* **Gralloc 则管理帧缓冲区的分配和释放**。上层只能通过Gralloc访问帧缓冲区，这样一来就实现了有序的封装保护

3. 由于**OpenGL ES是一个通用的函数库**，在不同的平台系统上需要被“本地化”——即把它与具体平台上的窗口系统建立起关联，这样才能保证它正常工作。从FramebufferNativeWindow这个名称就能判断出来，它就是将OpenGL ES在Android平台上本地化的中介之一。后面我们还会看到应用程序端所使用的另一个“本地窗口”。为OpengGL ES配置本地窗口的是EGL
4. **OpenGL或者OpenGL ES 更多的只是一个接口协议，实现上既可以采用软件，也能依托于硬件。**这一方面给产品开发带来了灵活性，我们可以根据成本与市场定位来决定具体的硬件设计，从而达到很好的定制需求;另一方面，既然有多种实现的可能，那么OpenGL ES在运行时是如何取舍的呢？这也是EGL的作用之一。它会去读取egl.cfg这个配置文件，然后根据用户的设定来动态加载libagl(软件实现)或者libhgl(硬件实现)。然后上层才可以正常使用各种glXXX接口
5. SurfaceFlinger中持有一个GraphicPlane成员变量mGraphicPlanes来描述“显示屏”;GraphicPlane类中又包含了一个DisplayHardware对象实例(mHw)。具体是在SurfaceFlinger::readyToRun中，完成对它们的创建与初始化。并且DisplayHardware在初始化时还将调用eglInitialize、eglCreateWindowSurface等接口，利用EGL来完成对OpenGLES环境的搭建。其中:
        surface =**eglCreateWindowSurface**(display, config, mNativeWindow.get(), NULL);
mNativeWindow 就是一个FramebufferNativeWindow对象。DisplayHardware为OpenGL ES设置了“本地化”所需的窗口

6. 很多模块都可以调用OpenGLES提供的API(这些接口以“gl”为前缀，比如glViewport、glClear、glMatrixMode、glLoadIdentity等等)，包括SurfaceFlinger、DisplayHardware等
7. 与OpenGL ES相关的模块，可以分为如下几类：

	* 配置类  即帮助OpenGL ES完成配置的，包括**EGL**、DisplayHardware都可以认为是这一类
	* 依赖类  也就是OpenGL ES要正常运行起来所依赖的“本地化”的东西，上图中是指FramebufferNativeWindow
	* 使用类  使用者也可能是配置者，比如DisplayHardware既扮演了“帮助”OpenGL的角色，同时它也是其使用方。另外只要处在与OpenGL ES同一个环境(Context)中的模块，都可以使用它来完成操作，比如SurfaceFlinger


# Gralloc与Framebuffer

## Gralloc库的加载
Gralloc对应的模块是由**FramebufferNativeWindow**(OpenGLES的本地窗口之一，后面小节有详细介绍)在构造时加载的：

	hw_get_module(HWC_HARDWARE_MODULE_ID, &mModule);
这个**hw_get_module**函数我们在前面已经见过很多次了，它是上层加载HAL库的入口，这里传入的模块ID名为：

	#define GRALLOC_HARDWARE_MODULE_ID  "gralloc"

按照hw_get_module的作法，它会在如下路径中查找与ID值匹配的库：

	#define HAL_LIBRARY_PATH1 "/system/lib/hw"
	#define HAL_LIBRARY_PATH2 "/vendor/lib/hw"

Gralloc对应的 lib库名有如下几种形式：

    gralloc.[ro.hardware].so
    gralloc.[ro.product.board].so
    gralloc.[ro.board.platform].so
    gralloc.[ro.arch].so
或者当上述的系统属性组成的文件名都不存在时，就使用默认的：

    gralloc.default.so

最后这个库是Android原生态的实现，位置在hardware/libhardware/modules/gralloc/中，它由gralloc.cpp、framebuffer.cpp和mapper.cpp三个主要源文件编译生成。

## Gralloc提供的接口
Gralloc实际操作了两个设备：

* fb0 就是我们前面说的主屏幕，
* gpu0负责图形缓冲区的分配和释放。
![](/img/android_graphic_system_02.png)


# 本地窗口
[原文](http://blog.csdn.net/xuesen_lin/article/details/8954748)

在OpenGL的学习过程中，我们不断提及“本地窗口”(NativeWindow)这一概念。那么对于Android系统来说，它是如何将OpenGL ES本地化的呢，或者说，它提供了什么样的本地窗口？
根据整个Android系统的GUI设计理念，我们不难猜想到至少需要两种本地窗口：

 1.  面向管理者(SurfaceFlinger) 。既然SurfaceFlinger扮演了系统中所有UI界面的管理者，那么它无可厚非地需要直接或间接地持有“本地窗口”。从前一小节我们已经知道，这个窗口就是**FramebufferNativeWindow**。
 2.  面向应用程序。我们先给出答案，这类窗口是**SurfaceTextureClient**

## 为什么需要两种窗口系统

### 理想的窗口系统

![](/img/android_graphic_system_03.png)
理想的窗口系统

假如整个系统仅有一个需要显示UI的程序，我们有理由相信它是可以胜任的。但是如果有N个UI程序的情况呢？Framebuffer显然只有一个，不可能让各个应用程序自己单独管理。

### 改进的窗口系统

![](/img/android_graphic_system_04.png)
改进的窗口系统

在这个改进的窗口系统中，我们有了两类本地窗口，即Window-1和Window-2。

 1. **第一种窗口**是能直接显示在终端屏幕上的——它**使用了帧缓冲区**，
 2. 而**后一种Window实际上是从内存缓冲区分配的空间**。

当系统中存在多个应用程序时，这能保证它们都可以获得一个“本地窗口”，并且这些窗口最终也能显示到屏幕上——SurfaceFlinger会收集所有程序的显示需求，对它们做统一的图像混合操作(有点类似于AudioFlinger)，然后输出到自己的Window-1上。
当然，这个改进的窗口系统有一个前提，即应用程序与SurfaceFlinger都是基于OpenGL ES来实现的。有没有其它选择呢？答案是肯定的，比如**应用程序端完全可以采用Skia等第三方的图形库，只要保持它们与SurfaceFlinger间的“协议”不变就可以了**，如下所示：

![](/img/android_graphic_system_05.png)

理论上来说，采用哪一种方式都是可行的。不过对于开发人员，特别是没有OpenGLES项目经验的人而言，前一种系统的门槛相对较高。事实上，Android系统同时提供了这两种实现来供上层选择。正常情况下我们按照SDK向导生成的apk应用，就属于后面的情况;而对于希望使用OpenGLES来完成复杂的界面渲染的应用开发者，也可以使用GLSurfaceView来达到目标。

![](/img/android_graphic_system_06.png)
Canvas为在画布的意思。Android上层的作图几乎都通过Canvas实例来完成，其实Canvas更多是一种接口的包装。

## FramebufferNativeWindow（面向SurfaceFlinger）



EGL创建一个Window Surface的函数原型列出如下：

```c++
	typedef  EGLNativeWindowType  NativeWindowType;//注意这两种类型其实是一样的
	struct ANativeWindow;
	…
	typedef struct ANativeWindow*          EGLNativeWindowType;

	EGLSurface eglCreateWindowSurface(  EGLDisplay dpy, EGLConfig config,
	                          NativeWindowType  window, const EGLint *attrib_list);
```


表格 11‑2 不同平台下的EGLNativeWindowType
|操作系统|数据类型|
|------|------|
|Win32, WinCE|HWND，即句柄|
|Symbian|Void*|
|Android|ANativeWindow*          |
|Unix|Window|
|其它|暂时不支持|


在Android平台下，ANativeWindow指针类型就是OpenGL接口中的windows surface类型。

### ANativeWindow的定义
ANativeWindow的定义在Window.h中：

```c++
	/*system/core/include/system/Window.h*/
	struct ANativeWindow
	{…
	    const uint32_t flags; //与Surface或updater有关的属性
	    const int   minSwapInterval;//所支持的最小交换间隔时间
	    const int   maxSwapInterval;//所支持的最大交换间隔时间
	    const float xdpi; //水平方向的密度，以dpi为单位
	    const float ydpi;//垂直方向的密度，以dpi为单位
	    intptr_t    oem[4];//为OEM定制驱动所保留的空间
	    //设置交换间隔时间，后面我们会讲解swap的作用
	    int     (*setSwapInterval)(struct ANativeWindow*window, int interval);
	    /*EGL通过这个接口来申请一个buffer。
	    以前面我们所举的例子来说，两个本地窗口所提供的buffer分别来自于帧缓冲区和内存空间。
	    单词“dequeue”的字面意思是“出队列”，这从侧面告诉我们，一个Window所包含的buffer很可能不只一份
	    */
	    int     (*dequeueBuffer)(struct ANativeWindow*window, struct ANativeWindowBuffer** buffer);
	    //申请到的buffer并没有被锁定，这种情况下是不允许我们去修改其中的内容的。所以我们必须要先调用lockBuffer来获得一个锁
	    int     (*lockBuffer)(struct ANativeWindow*window, struct ANativeWindowBuffer* buffer);
	    //当EGL对一块buffer渲染完成后，它调用这个接口来unlock和post buffer  
	    int    (*queueBuffer)(struct ANativeWindow* window, struct ANativeWindowBuffer*buffer);
	    //用于向本地窗口咨询相关信息
	    int     (*query)(const struct ANativeWindow*window, int what, int* value);
	    /*
	    用于执行本地窗口支持的各种操作，比如：
		NATIVE_WINDOW_SET_USAGE
		NATIVE_WINDOW_SET_CROP
		NATIVE_WINDOW_SET_BUFFER_COUNT
		NATIVE_WINDOW_SET_BUFFERS_TRANSFORM
		NATIVE_WINDOW_SET_BUFFERS_TIMESTAMP
	    */
	    int     (*perform)(struct ANativeWindow* window,int operation, ... );
	    //这个接口可以用来取消一个已经dequeued的buffer，要特别注意同步的问题
	    int     (*cancelBuffer)(struct ANativeWindow*window, struct ANativeWindowBuffer* buffer);
	    void* reserved_proc[2];
	};

```


从上面对ANativeWindow的描述可以看出，它更像是一份“协议”，规定了一个本地窗口的形态和功能。这对于支持多种本地窗口的系统是必须的，因为只有这样子我们才能针对某种特定的平台窗口，来填充具体的实现。
FramebufferNativeWindow是如何履行这份“协议”的。

### FramebufferNativeWindow构造函数
基于FramebufferNativeWindow的功能，可以大概推测出它的构造函数里应该至少完成如下的初始化操作：

 - 加载GRALLOC_HARDWARE_MODULE_ID模块，详细流程我们在Gralloc小节已经解释过了
 - 分别打开fb和gralloc设备。我们在Gralloc小节也已经分析过了，打开后的设备由全局变量fbDev和grDev管理
 - 根据设备的属性来给FramebufferNativeWindow赋初值
 - 根据FramebufferNativeWindow的实现来填充ANativeWindow中的“协议”
 - 其它一些必要的初始化

### dequeueBuffer
。。。

## SurfaceTextureClient（面向应用）

针对**应用程序端的本地窗口**是**SurfaceTextureClient**，和FramebufferNativeWindow一样，它必须继承ANativeWindow：

```C++
class SurfaceTextureClient : public ANativeObjectBase<ANativeWindow, SurfaceTextureClient,RefBase>
```

SurfaceTextureClient的构造函数只是简单地调用了init函数，后者则对ANativeWindow::dequeueBuffer等函数指针及内部变量赋了初值。由于整个函数的功能很简单，我们只摘录其中的一部分：


```c++
/*frameworks/native/libs/gui/SurfaceTextureClient.cpp*/
void SurfaceTextureClient::init() {
    /*给ANativeWindow中的函数指针赋值*/
   ANativeWindow::setSwapInterval  =hook_setSwapInterval;
    ANativeWindow::dequeueBuffer    = hook_dequeueBuffer;
    …
    /*为各内部变量赋值，因为此时用户还没有真正发起申请，所以基本是0*/
    mReqWidth = 0;
    mReqHeight = 0;
    …
    mDefaultWidth = 0;
    mDefaultHeight = 0;
    mUserWidth = 0;
    mUserHeight = 0;…
}
```

SurfaceTextureClient是面向Android系统中所有UI应用程序的，也就是说它承担着单个应用进程中的UI显示需求。基于这点考虑，可以推测出它的内部实现至少会有以下几点：

- 提供给上层(主要是java层)绘制图像的“画板”
前面说过，这个本地窗口分配的内存应该不是来自于帧缓冲区，那么具体是由谁分配的，又是如何管理的呢？
- 它与SurfaceFlinger间是如何分工的?
显然SurfaceFlinger需要收集系统中所有应用程序绘制的图像数据，然后集中显示到物理屏幕上。在这个过程中，SurfaceTextureClient扮演了什么样的角色呢？


SurfaceTextureClient只是一个中介，它间接调用mSurfaceTexture也就是ISurfaceTexture的服务。那么ISurfaceTexture在Server端又是由谁来完成的呢？

![](/img/android_graphic_system_07.png)

## 小结
通过这两个小节，我们学习了显示系统中两个重要的本地窗口，即FramebufferNativewindow和SurfaceTextureClient。

 1. 第一个窗口是专门为SurfaceFlinger服务的，它由Gralloc提供支持，相对逻辑上很好理解。
 2. 而SurfaceTextureClient则是为应用程序服务的，同时它从本质上还是由SurfaceFlinger服务统一管理的，因而涉及到很多跨进程的通信细节。

这个小节我们只是简单地勾勒出其中的框架，接下去就要分几个方面来做完整的分析了。

*  BufferQueue
为应用程序服务的本地窗口SurfaceTextureClient在server端的实现是BufferQueue。我们将详细解析BufferQueue的内部实现，并结合应用程序端的使用流程来理解清楚它们之间的关系。

*  Buffer、Consumer、Producer是“生产者-消费者”模型中的三个参与对象，如何协调好它们的工作是应用程序能否正常显示UI的关键。在接下来内容的安排上，我们先讲解Buffer(BufferQueue)与Producer(应用程序)间的交互，然后再专门切入Consumer(SurfaceFlinger)做详细分析

# BufferQueue
原文：http://blog.csdn.net/xuesen_lin/article/details/8954803

BufferQueue，它是SurfaceTextureClient实现本地窗口的关键。从逻辑上来推断，BufferQueue应该是驻留在SurfaceFlinger这边的进程中。我们需要进一步解决的疑惑是：

*  每个应用程序可以对应几个BufferQueue，它们是一对一、多对一或者是一对多？
*  应用程序所需要的绘图空间是由谁分配的？在音频系统的学习中，我们知道AudioTrack和AudioFlinger是通过共享内存的形式来进行数据传递的，那么显示系统中是否也是类似情况？
*  应用程序与SurfaceFlinger如何互斥共享数据区？和在Audio系统中遇到的问题一样，我们面临的是经典的“生产者-消费者”模型。显示系统又是如何协调好这两者间的互斥访问的呢？

## BufferQueue的内部原理

因为BufferQueue是ISurfaceTexture的本地实现，所以它必须重载接口中的各虚函数，比如queueBuffer、requestBuffer、dequeueBuffer等等。

![](/img/android_graphic_system_08.png)

另外，这个类的内部有一个非常重要的成员数组，即mSlots[NUM_BUFFER_SLOTS]，大家是否还记得前面SurfaceTextureClient类中也有一个一模一样的数组：

	class SurfaceTextureClient…{
	                BufferSlot  mSlots[NUM_BUFFER_SLOTS];

数组的成员是BufferSlot，其中包含的GraphicBuffer变量(mGraphicBuffer)用于记录这个Slot所涉及的缓冲区，另外还有一个BufferState变量mBufferState用于跟踪每个缓冲区的状态，比如：
```c++
        enum BufferState {
            FREE = 0, /*Buffer当前可用，也就是说可以被dequeued。此时Buffer的owner
                       可认为是BufferQueue*/
            DEQUEUED = 1, /*Buffer已经被dequeued，还未被queued或canceld。此时
                          Buffer的owner可认为是producer(应用程序)，这意味着server
                          端(BufferQueue)不可以对这块缓冲区进行操作*/
            QUEUED = 2, /*Buffer已经被客户端queued，除特别情况外此时还不能对它进
                          行dequeue，而可以acquired。此时的owner是BufferQueue*/
            ACQUIRED = 3/*Buffer的owner改为consumer，可以released，
                           然后状态又返回FREE*/
        };
```

Buffer状态迁移图如下：
![](/img/android_graphic_system_09.png)


在这样的模型下，我们怎么保证Consumer可以及时的处理buffer呢？换句话说，当一块buffer数据ready后，应该怎么告知Consumer来操作呢？
仔细观察的话，可以看到BufferQueue里还同时提供了一个特别的类，名称为ConsumerListener，其中的函数接口包括：

	    struct ConsumerListener :public virtual RefBase {       
	        virtual void onFrameAvailable() = 0;/*当一块buffer可以被消费时，这个函数会被调用，特别注意此
	                                        时没有共享锁的保护*/
	        virtual void onBuffersReleased() = 0;/*BufferQueue通知consumer它已经释放其slot中的一个或多个
	                                       GraphicBuffer引用*/
	    };
 这样子就很清楚了，当有一帧数据准备就绪后，BufferQueue就会调用**onFrameAvailable**()来通知Consumer进行消费。

## BufferQueue中的缓冲区分配

原文：http://blog.csdn.net/xuesen_lin/article/details/8954834


当客户端 调用SurfaceTextureClient::dequeueBuffer()时，会导致SurfaceFlinger在BufferQueue中分配缓冲。

### 服务端

BufferQueue中有一个mSlots数组用于管理其内的各缓冲区，最大容量为32。从它的声明方式来看，这个mSlots在程序一开始就静态分配了32个BufferSlot大小的空间。不过这并不代表缓冲区也是一次性静态分配的，恰恰相反，从BufferSlot的内部变量指针mGraphicBuffer可以看出，**缓冲区的空间分配应当是动态的**(从下面的注释也能看出一些端倪)：

	// mGraphicBuffer points to the buffer allocated for this slot or isNULL if no buffer has been allocated.
	       sp<GraphicBuffer> mGraphicBuffer;

缓冲区的分配在BufferQueue::dequeueBuffer()中进行。

	`status_t BufferQueue::dequeueBuffer(int *outBuf, uint32_t w,uint32_t h, uint32_t format, uint32_t usage)`

在该函数中，首先查找空闲的slot序号，但并不代表可用。当第一次使用该slot时，或者长宽高格式等信息不匹配时，其返回值中将加上标识BUFFER_NEEDS_REALLOCATION，表示需要分配内存。

### 客户端
在客户端SurfaceTextureClient::dequeueBuffer()中的实现中可以发现：

 1. 当mSurfaceTexture->dequeueBuffer()成功返回后，buf包含了mSlots中可用数组成员的序号。
 2. 如果结果中还发现BUFFER_NEEDS_REALLOCATION标志后，应该调用requestBuffer（）来分配内存。
 3. 然后，通过  `sp<GraphicBuffer>& gbuf；*buffer = gbuf.get();` 来获取mSlots[buf]在本进程的地址。

但一个很显然的问题是，既然客户端和BufferQueue运行于两个不同的进程中，那么它们两者中的mSlots[buf]会指向同一块物理内存吗？
SurfaceFlinger收到requestBuffer请求后，会实际分配内存，将其句柄handle通过binder的序列化接口传送到客户端。
通过handle句柄，Client端（应用）可以将指定的内存区域映射到自己的进程空间中，而这块区域与BufferQueue中所指向的物理空间是一致的，从而成功地实现了缓冲区的共享。handle它实际上是GraphicBuffer中打开的一个ashmem句柄，因而也就是两边进程共享缓冲区的关键。
## 应用程序的典型绘图流程

原文：http://blog.csdn.net/xuesen_lin/article/details/8954840

应用程序并不会直接使用BufferQueue。和Android系统中很多其它地方一样，“层层包裹”在这里同样是存在的。

我们选取系统的开机动画这一应用程序，来分析整个图形绘制的流程。值得一提的是，这个开机动画的实现符合前面提到的两个改进的图形系统中的第一个，即应用程序与SurfaceFlinger都是使用OpenGL ES来完成UI显示，不过因为它是一个C++程序，所以不需要上层GLSurfaceView的支持。

这个开机动画的实现类是BootAnimation，它的内部就是借助SurfaceFlinger来完成的。

BootAnimation是一个C++程序，其工程源码路径是/frameworks/base/cmds/bootanimation。和很多native应用一样，它也是在init脚本中被启动的，大概来看下这一过程。

	service bootanim /system/bin/bootanimation
	    class main
	    user graphics
	    group graphics
	    disabled
	    oneshot

当bootanimation被启动后，它首先会进入main函数，即main@Bootanimation_main.cpp，生成一个BootAnimation对象，并开启线程池(因为它需要与SurfaceFlinger等系统服务进行跨进程的通信)。在BootAnimation的构造函数中，同时生成一个SurfaceComposerClient：

```c++
	BootAnimation::BootAnimation() : Thread(false)
	{
	    mSession = newSurfaceComposerClient();
	}	   
```

**ISurfaceTexture是应用程序与BufferQueue的传输通道，而ISurfaceComposerClient则是它与SurfaceFlinger间的桥梁。**

个人理解：ISurfaceTexture 封装对BufferQueue的访问，将绘制的内容放在缓冲区上。而ISurfaceComposerClient 负责告诉SurfaceFlinger这些缓冲区的元数据，以便SurfaceFlinger采用合适的参数对绘制缓冲（surface）做进一步的合成。

这样子的设计是合理的，体现了模块化的思想——SurfaceFlinger的职责是“Flinger”，即把系统中所有应用程序的最终的“绘图结果”进行“混合”，然后统一显示到物理屏幕上。它不应该，也没有办法分出太多的精力去一一关注各个应用程序的“绘画过程”。这个光荣的任务自然而然地落在了BufferQueue的肩膀上，它是每个应用程序“一对一”的辅导老师，指导着UI程序的“画板申请”、“作画流程”等一系列细节。下面的图描述了这三者的关系：

![](/img/android_graphic_system_10.png)

的确是太乱了，我们有必要先来整理下目前已经出现的容易混淆的相关类的关系：

- **ISurfaceComposerClient**: 应用程序与SurfaceFlinger间的通道，在应用进程中则被封装在SurfaceComposerClient这个类中。这是一个匿名binder server，由应用程序(具体位置在SurfaceComposerClient::onFirstRef中)调用SurfaceFlinger这个实名binder的createConnection方法来获取到，服务端的实现是SurfaceFlinger::Client。
- **ISurface**:由应用程序调用ISurfaceComposerClient::createSurface()得到，同时在SurfaceFlinger这一进程中将会有一个Layer被创建，代表了一个“画面”。ISurface就是控制这一画面的handle。
- **Surface**:从逻辑关系上看，它是上述**ISurface**的使用者。从继承关系上看，它是一个**SurfaceTextureClient**，也就是本地窗口。SurfaceTextureClient内部持有**ISurfaceTexture**，即**BufferQueue**的实现接口。换个角度来思考，当EGL想通过Surface这个native window完成某些功能时，后者实际上又利用ISurface和ISurfaceTexture来取得远程服务端的对应服务，以完成EGL的请求。
- **Layer**：既然Layer代表了一个画面图层，那么它肯定需要有存储图层数据的地方。当应用端通过ISurfaceComposerClient::createSurface()来发起创建Surface的请求时，SurfaceFlinger服务进程这边会创建一个Layer。

一个典型的应用程序使用SurfaceFlinger进行绘图的流程如下图所示：
![](/img/android_graphic_system_11.png)

各个class之间的关系式这样的：
![](/img/android_graphic_system_12.png)

## 应用程序与BufferQueue的关系
原文：http://blog.csdn.net/xuesen_lin/article/details/8954853

应用程序与BufferQueue的关系就比较明朗了。虽然中间经历了多次跨进程通信，但对于应用程序来说最终只使用到了BufferQueue(通过ISurfaceTexture)。从本小节的内容中，我们也可以从侧面证明如下几个关键点：

1. 应用程序可以调用createSurface来建立多个Layer，它们是一对多的关系。理由就是createSurface中没有任何机制来限制应用程序的多次调用，相反，它会把一个应用程序多次申请而产生的Layer统一管理。为应用程序申请的layer，一方面需要告知SurfaceFlinger，另一方面也要记录到各Client内部中，这两个步骤是由addClientLayer()分别调用Client::attachLayer()和SurfaceFlinger::addLayer_l()来完成的。对于SurfaceFlinger，它需要对系统中当前所有的Layer进行Z-order排序，以决定用户所能看到的“画面”是什么样的。对于Client，它则利用内部的mLayers成员变量来一一记录新增(attachLayer)和移除(detachLayer)的图层。
2. 每个Layer对应一个BufferQueue，换句话说，一个应用程序可能对应多个BufferQueue。Layer没有直接持有BufferQueue，而是由其内部的mSurfaceTexture来管理。

![](/img/android_graphic_system_13.png)

![](/img/android_graphic_system_14.png)

图 11‑19 应用程序与BufferQueue的对应关系


----------


# SurfaceFlinger
## Project Butter黄油计划
原文：http://blog.csdn.net/xuesen_lin/article/details/8954869

CPU、GPU负责渲染帧数据，Display负责屏幕的显示。
FPS（Frame peer Second） 代表CPU、GPU每秒能够渲染的桢数量。
刷新率 代表 显示设备的刷新率。
VSync(垂直同步)是VerticalSynchronization的简写，它利用VBI时期出现的vertical sync pulse来保证双缓冲在最佳时间点才进行交换。

![](/img/android_graphic_system_15.png)
图 11‑22绘图过程没有采用VSync同步的情况

![](/img/android_graphic_system_16.png)
图 11‑25 FPS低于屏幕刷新率的情况

当CPU/GPU的处理时间超过16ms时，第一个VSync到来时，缓冲区B中的数据还没有准备好，于是只能继续显示之前A缓冲区中的内容。而B完成后，又因为缺乏VSync pulse信号，它只能等待下一个signal的来临。于是在这一过程中，有一大段时间是被浪费的。当下一个VSync出现时，CPU/GPU马上执行操作，此时它可操作的buffer是A，相应的显示屏对应的就是B。这时看起来就是正常的。只不过由于执行时间仍然超过16ms，导致下一次应该执行的缓冲区交换又被推迟了——如此循环反复，便出现了越来越多的“Jank”。

为了避免tear，当FPS > 刷新率时，采用双缓冲技术。当收到VSync，CPU/GPU立即开始准备新的frame；而Display则显示之间已经准备好的frame。
当FPS < 刷新率时，采用三缓冲技术，保证frame准备好能够尽快显示。这时候如果采用双缓冲，准备好的帧可能需要等待近一个VSync周期才能显示。


## SurfaceFlinger的启动与工作原理
原文：http://blog.csdn.net/xuesen_lin/article/details/8954951

## SurfaceComposerClient
原文：http://blog.csdn.net/xuesen_lin/article/details/8954957

![](/img/android_graphic_system_17.png)
图 11‑28 每个应用程序在SurfaceFlinger中都对应一个Client


SurfaceFlinger运行于SystemServer这一系统进程中，需要UI界面显示的应用程序则通过binder服务与它进行跨进程通信。在音频系统的学习中，每一个AudioTrack在AudioFlinger中都可以找到一个对应的Track实现。这种设计方式同样适用于显示系统，即任何有UI界面的程序都在SurfaceFlinger中有且仅有一个Client实例。

应用程序与SurfaceFlinger间的接口是**ISurfaceComposerClient**，**Client**的父类是BnSurfaceComposerClient,它是这一接口的本地端实现。

ISurfaceComposerClient接口中最重要的两个方法createSurface()和destroySurface()分别用于向SurfaceFlinger申请和销毁一个ISurface。**那么既然有了Client，为什么还要再引出另一个binder对象呢？**

这是因为每个SurfaceFlinger的**客户程序**都只会有唯一一个Client连接，但它们**内部拥有的Surface数量却很可能有多个**。通常情况下，同一个Activity中的UI布局共用系统分配的Surface进行绘图，但像SurfaceView这种UI组件就是特例——它独占一个Surface进行绘制。举个例子来说，如果我们制作一个带SurfaceView的视频播放器，其所在的应用程序最终就会有不止一个的Surface存在。这样设计是必须的，因为播放视频对刷新频率要求很高，采用单独的Surface既可以保证视频的流畅度，也同时能让用户的交互动作(比如触摸屏操作)及时得到响应。

![](/img/android_graphic_system_18.png)


# VSync
原文：http://blog.csdn.net/xuesen_lin/article/details/8954986

Android 4.1显示系统中的新特性，其中一个就是加入了VSync同步。
VSync信号是如何产生的呢？
在Android源码surfaceflinger目录下有一个displayhardware文件夹，其中HWComposer的主要职责之一，就是用于产生VSync信号。**VSync信号源 可以是硬件的，也可以是软件模拟的**。
分析HWComposer::HWComposer（...）,假如当前系统可以成功加载HWC_HARDWARE_MODULE_ID=“hwcomposer”，并且通过这个库模块能顺利打开设备(hwc_composer_device_t)，其版本号又大于HWC_DEVICE_API_VERSION_0_3的话，我们就采用“**硬件源**”(此时needVSyncThread为false)，否则需要创建一个新的VSync线程来模拟产生信号。


## VSync信号的产生和分发


## VSync信号的处理.
http://blog.csdn.net/xuesen_lin/article/details/8955012

现在SurfaceFlinger只处理REFRESH一个消息，其主要工作是：

*  handleTransaction
即处理事务，什么样的事务呢？在SurfaceFlinger::setTransactionState()中我们可以看到，假如当前的orientation和新的不符合时，会将eTransactionNeeded置位;当应用程序请求createSurface、removeSurface，或者addLayer、removeLayer时也会把它置位。另一个flag被置位的情况则包括：layer的size、alpha、matrix、transparentregion、visibility变化等等。
总结起来，就是当与系统显示相关的状态(比如新增/减少了Surface，显示屏的变化等等)改变，或者某个Layer自身状态(比如它的大小尺寸、可见性、透明度等等)改变时，就需要执行Transaction。
*  handlePageFlip
由前面的分析我们知道，每个Layer对应着最多32个BufferSlot，这样系统在进行一次刷新时，必须先决定使用哪个buffer，并利用这一缓冲区更新纹理。另外，我们还需要计算所有图层的可见区域和“脏区域”，以便最终的合成显示。
*  handleRefresh
版本更新遗留下的函数，当前实现中没有起到作用，相信在后续升级中会进一步完善。
* handleWorkList
创建HWComposer中的mList，这个列表将用于后续的layer合成。这个函数比较简单，我们不单独介绍。
*  handleRepaint
计算出最终的脏区域，并执行实际的合成工作(composeSurfaces)，我们将做详细源码分析。
*  postFramebuffer
将上一步中生成的缓冲区数据post到framebuffer中，这样才能真正在物理屏幕上显示出来。分为两条路径，即HWComposer::commit和直接调用eglSwapBuffers()

## handleTransaction
http://blog.csdn.net/xuesen_lin/article/details/8955121

## handlePageFlip
http://blog.csdn.net/xuesen_lin/article/details/8955138

## handleRefresh
http://blog.csdn.net/xuesen_lin/article/details/8955173

## handleRepaint
http://blog.csdn.net/xuesen_lin/article/details/8955183

## postFramebuffer
http://blog.csdn.net/xuesen_lin/article/details/8955198
