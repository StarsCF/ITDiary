[TOC]

## OSX/iOS 的系统架构

![](/Users/fanchongchong/Documents/Github/ITDiary/iOS/6、 OSX:iOS 系统架构/images/RunLoop_3.png)

苹果官方将整个系统大致划分为上述4个层次：
应用层包括用户能接触到的图形应用，例如 Spotlight、Aqua、SpringBoard 等。
应用框架层即开发人员接触到的 Cocoa 等框架。
核心框架层包括各种核心框架、OpenGL 等内容。
Darwin 即操作系统的核心，包括系统内核、驱动、Shell 等内容，这一层是开源的，其所有源码都可以在 [opensource.apple.com](http://opensource.apple.com/) 里找到。

我们在深入看一下 Darwin 这个核心的架构：

![](/Users/fanchongchong/Documents/Github/ITDiary/iOS/6、 OSX:iOS 系统架构/images/RunLoop_4.png)

其中，在硬件层上面的三个组成部分：Mach、BSD、IOKit (还包括一些上面没标注的内容)，共同组成了 XNU 内核。
XNU 内核的内环被称作 Mach，其作为一个微内核，仅提供了诸如处理器调度、IPC (进程间通信)等非常少量的基础服务。
BSD 层可以看作围绕 Mach 层的一个外环，其提供了诸如进程管理、文件系统和网络等功能。
IOKit 层是为设备驱动提供了一个面向对象(C++)的一个框架。

Mach 本身提供的 API 非常有限，而且苹果也不鼓励使用 Mach 的 API，但是这些API非常基础，如果没有这些API的话，其他任何工作都无法实施。在 Mach 中，所有的东西都是通过自己的对象实现的，进程、线程和虚拟内存都被称为”对象”。和其他架构不同， Mach 的对象间不能直接调用，只能通过消息传递的方式实现对象间的通信。**”消息”是 Mach 中最基础的概念，消息在两个端口 (port) 之间传递，这就是 Mach 的 IPC (进程间通信) 的核心。**

Mach 的消息定义是在 <mach/message.h> 头文件的，很简单：

```c
typedef struct {
  mach_msg_header_t header;
  mach_msg_body_t body;
} mach_msg_base_t;
 
typedef struct {
  mach_msg_bits_t msgh_bits;
  mach_msg_size_t msgh_size;
  mach_port_t msgh_remote_port;
  mach_port_t msgh_local_port;
  mach_port_name_t msgh_voucher_port;
  mach_msg_id_t msgh_id;
} mach_msg_header_t;
```

一条 Mach 消息实际上就是一个二进制数据包 (BLOB)，其头部定义了当前端口 local_port 和目标端口 remote_port，发送和接受消息是通过同一个 API 进行的，其 option 标记了消息传递的方向：

```c
mach_msg_return_t mach_msg(
			mach_msg_header_t *msg,
			mach_msg_option_t option,
			mach_msg_size_t send_size,
			mach_msg_size_t rcv_size,
			mach_port_name_t rcv_name,
			mach_msg_timeout_t timeout,
			mach_port_name_t notify);
```

为了实现消息的发送和接收，mach_msg() 函数实际上是调用了一个 Mach 陷阱 (trap)，即函数mach_msg_trap()，陷阱这个概念在 Mach 中等同于系统调用。当你在用户态调用 mach_msg_trap() 时会触发陷阱机制，切换到内核态；内核态中内核实现的 mach_msg() 函数会完成实际的工作，如下图：

![](/Users/fanchongchong/Documents/Github/ITDiary/iOS/6、 OSX:iOS 系统架构/images/RunLoop_5.png)

这些概念可以参考维基百科: [System_call](http://en.wikipedia.org/wiki/System_call)、[Trap_(computing)](http://en.wikipedia.org/wiki/Trap_(computing))。

关于具体的如何利用 mach port 发送信息，可以看看[ NSHipster 这一篇文章](http://nshipster.com/inter-process-communication/)，或者[这里](http://segmentfault.com/a/1190000002400329)的中文翻译 。

关于Mach的历史可以看看这篇很有趣的文章：[Mac OS X 背后的故事（三）Mach 之父 Avie Tevanian](http://www.programmer.com.cn/8121/)。

>参考文章
>
>1、[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)