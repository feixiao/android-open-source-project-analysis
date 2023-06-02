# <img src="./art/logo.png" alt="Android open source project analysis" width="80" height="80" align="bottom"/> Android open source project analysis

## 功能介绍

[![License](https://img.shields.io/github/license/guoxiaoxing/android-open-source-project-analysis.svg)](https://jitpack.io/#guoxiaoxing/android-open-source-project-analysis)
[![Stars](https://img.shields.io/github/stars/guoxiaoxing/android-open-source-project-analysis.svg)](https://jitpack.io/#guoxiaoxing/android-open-source-project-analysis)
[![Stars](https://img.shields.io/github/forks/guoxiaoxing/android-open-source-project-analysis.svg)](https://jitpack.io/#guoxiaoxing/android-open-source-project-analysis)
[![Forks](https://img.shields.io/github/issues/guoxiaoxing/android-open-source-project-analysis.svg)](https://jitpack.io/#guoxiaoxing/android-open-source-project-analysis)

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

- [Git repositories on android](https://android.googlesource.com/)
- [Android Open Source Project](https://source.android.com/)

**代码版本**

- 细分版本：N6F26U
- 分支：Android-7.1.1_r28
- 版本：Nougat
- 支持设备：Nexus 6

**分析思路**

Android 是一个庞大的系统，Android Framework 只是对系统的一个封装，里面还牵扯到 JNI、C++、Java 虚拟机、Linux 系统内核、指令集等。面对如此庞大的系统，我们得有一定的
章法去阅读源码，否则就会只见树木不见森林，陷入卷帙浩繁的细节与琐碎之中。

- **不要去记录那些 API 调用链，绘制一个序列图理清思路即可**，Android Framework 中有很多复杂的 API 调用链，你去关注这些东西，用处不大。你需要学会的是跟踪调用链和梳理流程的
  技巧，**思考一下作者是怎么找到关键入口的，核心的实现在什么地方。**
- 要善于思考，要多问为什么，面对一个模块，你要去思考这个模块解决了什么问题，这个问题的本质是什么，为什么这么解决，如果让我来写，我会怎么设计。事实上不管是是计算机还是
  手机，从 CPU、到内存、到操作系统、到应用层，看似纷繁复杂，但问题的本质无非就是这么几种：**时间片怎么分配？线程/进程怎么调度？通信的机制是什么？** 只是在不同的场景下加了具体
  的优化，但问题的本质没有改变，我们要善于抓住本质。
- 要善于去粗存精，Android Framework 也是人写的，有精华也有糟粕，并不是每行代码你都需要问个为什么，很多时候没有那么多为什么，只是当时那种情况下就那样设计了。但是
  对于关键函数我们要去深究它的实现细节。

**写作风格**

和大家一样，笔者也是在前人的书籍和博客的基础上开始学习 Android 的底层实现的，站在前人的肩膀上会看的更远。但是这些书籍和博客有个问题在于，文章中罗列了大量的代码，这样
很容易把初学者带入到琐碎的细节之中，所以本系列文章在行文中更多的会以图文并茂和提纲总结的方式来分析问题，关键的地方才会去解析源码，力求让大家从宏观上理解 Android 的底
层实现。另外，基本上一个主题对应一篇文章，所以文章会比较长，但是文章会有详细的标题划分和提纲总结，可以有的放矢，阅读自己需要的内容。

好了，让我们开始我们的寻宝之旅吧~😆

**Android 系统架构图**

Android 系统架构图

<img src="./art/android_system_structure.png"/>

从上到下依次分为六层：

- 应用框架层
- 进程通信层
- 系统服务层
- Android 运行时层
- 硬件抽象层
- Linux 内核层

在正式阅读本系列文章之前，请先阅读导读相关内容，这会帮助你更加快捷的理解文章内容。

- [导读](./doc/导读.md)

## Android 系统应用框架篇

**Android 窗口管理框架**

- [01Android 窗口管理框架：Android 窗口管理框架概述](./doc/Android系统应用框架篇/Android窗口管理框架/01Android窗口管理框架：Android窗口管理框架概述.md)
- [02Android 窗口管理框架：Android 应用视图的载体 View](./doc/Android系统应用框架篇/Android窗口管理框架/02Android窗口管理框架：Android应用视图载体View.md)
- [03Android 窗口管理框架：Android 应用视图的管理者 Window](./doc/Android系统应用框架篇/Android窗口管理框架/03Android窗口管理框架：Android应用视图管理者Window.md)
- [04Android 窗口管理框架：Android 应用窗口管理服务 WindowServiceManager](./doc/Android系统应用框架篇/Android窗口管理框架/04Android窗口管理框架：Android应用窗口管理服务WindowServiceManager.md)
- [05Android 窗口管理框架：Android 布局解析者 LayoutInflater](./doc/Android系统应用框架篇/Android窗口管理框架/05Android窗口管理框架：Android布局解析者LayoutInflater.md)
- [06Android 窗口管理框架：Android 列表控件 RecyclerView](./doc/Android系统应用框架篇/Android窗口管理框架/06Android窗口管理框架：Android列表控件RecyclerView.md)

**Android 组件管理框架**

- [01Android 组件管理框架：Android 组件管理框架概述](./doc/Android系统应用框架篇/Android组件管理框架/01Android组件管理框架：组件管理框架概述.md)
- [02Android 组件管理框架：Android 组件管理服务 ActivityServiceManager](./doc/Android系统应用框架篇/Android组件管理框架/02Android组件管理框架：Android组件管理服务ActivityServiceManager.md)
- [03Android 组件管理框架：Android 视图容器 Activity](./doc/Android系统应用框架篇/Android组件管理框架/03Android组件管理框架：Android视图容器Activity.md)
- [04Android 组件管理框架：Android 视图片段 Fragment](./doc/Android系统应用框架篇/Android组件管理框架/04Android组件管理框架：Android视图片段Fragment.md)
- [05Android 组件管理框架：Android 后台服务 Service](./doc/Android系统应用框架篇/Android组件管理框架/05Android组件管理框架：Android后台服务Service.md)
- [06Android 组件管理框架：Android 内容提供者 ContentProvider](./doc/Android系统应用框架篇/Android组件管理框架/06Android组件管理框架：Android内容提供者ContentProvider.md)
- [07Android 组件管理框架：Android 广播接收者 BroadcastReceiver](./doc/Android系统应用框架篇/Android组件管理框架/07Android组件管理框架：Android广播接收者BroadcastReceiver.md)
- [08Android 组件管理框架：Android 应用上下文 Context](./doc/Android系统应用框架篇/Android组件管理框架/08Android组件管理框架：Android应用上下文Context.md)

**Android 包管理框架**

- [01Android 包管理框架：APK 的打包流程](./doc/Android系统应用框架篇/Android包管理框架/01Android包管理框架：APK的打包流程.md)
- [02Android 包管理框架：APK 的安装流程](./doc/Android系统应用框架篇/Android包管理框架/02Android包管理框架：APK的安装流程.md)
- [03Android 包管理框架：APK 的加载流程](./doc/Android系统应用框架篇/Android包管理框架/03Android包管理框架：APK的加载流程.md)

**Android 资源管理框架**

- [01Android 资源管理框架：资源管理器 AssetManager](./doc/Android系统应用框架篇/Android资源管理管理框架/01Android资源管理框架：资源管理器AssetManager.md)

## Android 系统底层框架篇

**Android 进程框架**

- [01Android 进程框架：进程的创建、启动与调度流程](./doc/Android系统底层框架篇/Android进程框架/01Android进程框架：进程的创建、启动与调度流程.md)
- [02Android 进程框架：线程与线程池](./doc/Android系统底层框架篇/Android进程框架/02Android进程框架：线程与线程池.md)
- [03Android 进程框架：线程通信的桥梁 Handler](./doc/Android系统底层框架篇/Android进程框架/03Android进程框架：线程通信的桥梁Handler.md)
- [04Android 进程框架：进程通信的桥梁 Binder](./doc/Android系统底层框架篇/Android进程框架/04Android进程框架：进程通信的桥梁Binder.md)
- [05Android 进程框架：进程通信的桥梁 Socket](./doc/Android系统底层框架篇/Android进程框架/05Android进程框架：进程通信的桥梁Socket.md)

**Android 内存框架**

- [01Android 内存框架：内存管理系统](./doc/Android系统底层框架篇/Android内存框架/01Android内存框架：内存管理系统.md)
- [02Android 内存框架：Ashmem 匿名共享内存系统](./doc/Android系统底层框架篇/Android内存框架/02Android内存框架：Ashmem匿名共享内存系统.md)

**Android 虚拟机框架**

- [01Android 虚拟机框架：Java 类加载机制](./doc/Android系统底层框架篇/Android虚拟机框架/01Android虚拟机框架：Java类加载机制.md)

## Android 应用开发实践篇

**Android 界面开发**

- [01Android 界面开发：View 自定义实践概览](./doc/Android应用开发实践篇/Android界面开发/01Android界面开发：View自定义实践概览.md)
- [02Android 界面开发：View 自定义实践布局篇](./doc/Android应用开发实践篇/Android界面开发/02Android界面开发：View自定义实践布局篇.md)
- [03Android 界面开发：View 自定义实践绘制篇](./doc/Android应用开发实践篇/Android界面开发/03Android界面开发：View自定义实践绘制篇.md)
- [04Android 界面开发：View 自定义实践交互篇](./doc/Android应用开发实践篇/Android界面开发/04Android界面开发：View自定义实践交互篇.md)

**Android 应用优化**

- [01Android 应用优化：优化概述](./doc/Android应用开发实践篇/Android应用优化/01Android应用优化：优化概述.md)
- [02Android 应用优化：启动优化](./doc/Android应用开发实践篇/Android应用优化/02Android应用优化：启动优化.md)
- [03Android 应用优化：界面优化](./doc/Android应用开发实践篇/Android应用优化/03Android应用优化：界面优化.md)
- [04Android 应用优化：内存优化](./doc/Android应用开发实践篇/Android应用优化/04Android应用优化：内存优化.md)
- [05Android 应用优化：图像优化](./doc/Android应用开发实践篇/Android应用优化/05Android应用优化：图像优化.md)
- [06Android 应用优化：网络优化](./doc/Android应用开发实践篇/Android应用优化/06Android应用优化：网络优化.md)
- [07Android 应用优化：并发优化](./doc/Android应用开发实践篇/Android应用优化/07Android应用优化：并发优化.md)
- [08Android 应用优化：优化工具](./doc/Android应用开发实践篇/Android应用优化/08Android应用优化：优化工具.md)

**Android 媒体开发**

- [01Android 媒体开发：Bitmap 实践指南](./doc/Android应用开发实践篇/Android媒体开发/01Android媒体开发：Bitmap实践指南.md)
- [02Android 媒体开发：Camera 实践指南](./doc/Android应用开发实践篇/Android媒体开发/02Android媒体开发：Camera实践指南.md)

**其他**

- [01Android 混合编程：WebView 实践](./doc/Android应用开发实践篇/其他/01Android混合编程：WebView实践.md)
- [02Android 网络编程：网络编程实践](./doc/Android应用开发实践篇/其他/02Android网络编程：网络编程实践.md)

## Android 系统软件设计篇

- [01Android 系统软件设计篇：软件设计原则](./doc/Android系统软件设计篇/01Android系统软件设计篇：软件设计原则.md)
- [02Android 系统软件设计篇：设计模式](./doc/Android系统软件设计篇/02Android系统软件设计篇：设计模式.md)
