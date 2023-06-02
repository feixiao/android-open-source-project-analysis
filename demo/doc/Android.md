### 手画一下 Android 系统架构图，描述一下各个层次的作用？

Android 系统架构图

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/android_system_structure.png" width="600"/>

从上到下依次分为六层：

- 应用框架层
- 进程通信层
- 系统服务层
- Android 运行时层
- 硬件抽象层
- Linux 内核层

### Activity 如与 Service 通信？

可以通过 bindService 的方式，先在 Activity 里实现一个 ServiceConnection 接口，并将该接口传递给 bindService()方法，在 ServiceConnection 接口的 onServiceConnected()方法
里执行相关操作。

### 描述一下 Android 的事件分发机制？

Android 事件分发机制的本质：事件从哪个对象发出，经过哪些对象，最终由哪个对象处理了该事件。此处对象指的是 Activity、Window 与 View。

Android 事件的分发顺序：Activity（Window） -> ViewGroup -> View

Android 事件的分发主要由三个方法来完成，如下所示：

```java
// 父View调用dispatchTouchEvent()开始分发事件
public boolean dispatchTouchEvent(MotionEvent event){
    boolean consume = false;
    // 父View决定是否拦截事件
    if(onInterceptTouchEvent(event)){
        // 父View调用onTouchEvent(event)消费事件，如果该方法返回true，表示
        // 该View消费了该事件，后续该事件序列的事件（Down、Move、Up）将不会在传递
        // 该其他View。
        consume = onTouchEvent(event);
    }else{
        // 调用子View的dispatchTouchEvent(event)方法继续分发事件
        consume = child.dispatchTouchEvent(event);
    }
    return consume;
}
```

### 描述一下 View 的绘制原理？

View 的绘制流程主要分为三步：

1. onMeasure：测量视图的大小，从顶层父 View 到子 View 递归调用 measure()方法，measure()调用 onMeasure()方法，onMeasure()方法完成绘制工作。
2. onLayout：确定视图的位置，从顶层父 View 到子 View 递归调用 layout()方法，父 View 将上一步 measure()方法得到的子 View 的布局大小和布局参数，将子 View 放在合适的位置上。
3. onDraw：绘制最终的视图，首先 ViewRoot 创建一个 Canvas 对象，然后调用 onDraw()方法进行绘制。onDraw()方法的绘制流程为：① 绘制视图背景。② 绘制画布的图层。 ③ 绘制 View 内容。
   ④ 绘制子视图，如果有的话。⑤ 还原图层。⑥ 绘制滚动条。

### 了解 APK 的打包流程吗，描述一下？

Android 的包文件 APK 分为两个部分：代码和资源，所以打包方面也分为资源打包和代码打包两个方面，这篇文章就来分析资源和代码的编译打包原理。

APK 整体的的打包流程如下图所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/native/vm/apk_package_flow.png"/>

具体说来：

1. 通过 AAPT 工具进行资源文件（包括 AndroidManifest.xml、布局文件、各种 xml 资源等）的打包，生成 R.java 文件。
2. 通过 AIDL 工具处理 AIDL 文件，生成相应的 Java 文件。
3. 通过 Javac 工具编译项目源码，生成 Class 文件。
4. 通过 DX 工具将所有的 Class 文件转换成 DEX 文件，该过程主要完成 Java 字节码转换成 Dalvik 字节码，压缩常量池以及清除冗余信息等工作。
5. 通过 ApkBuilder 工具将资源文件、DEX 文件打包生成 APK 文件。
6. 利用 KeyStore 对生成的 APK 文件进行签名。
7. 如果是正式版的 APK，还会利用 ZipAlign 工具进行对齐处理，对齐的过程就是将 APK 文件中所有的资源文件举例文件的起始距离都偏移 4 字节的整数倍，这样通过内存映射访问 APK 文件
   的速度会更快。

### 了解 APK 的安装流程吗，描述一下？

APK 的安装流程如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/package/apk_install_structure.png" width="600"/>

1. 复制 APK 到/data/app 目录下，解压并扫描安装包。
2. 资源管理器解析 APK 里的资源文件。
3. 解析 AndroidManifest 文件，并在/data/data/目录下创建对应的应用数据目录。
4. 然后对 dex 文件进行优化，并保存在 dalvik-cache 目录下。
5. 将 AndroidManifest 文件解析出的四大组件信息注册到 PackageManagerService 中。
6. 安装完成后，发送广播。

### 当点击一个应用图标以后，都发生了什么，描述一下这个过程？

点击应用图标后会去启动应用的 LauncherActivity，如果 LancerActivity 所在的进程没有创建，还会创建新进程，整体的流程就是一个 Activity 的启动流程。

Activity 的启动流程图（放大可查看）如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/activity_start_flow.png" />

整个流程涉及的主要角色有：

- Instrumentation: 监控应用与系统相关的交互行为。
- AMS：组件管理调度中心，什么都不干，但是什么都管。
- ActivityStarter：Activity 启动的控制器，处理 Intent 与 Flag 对 Activity 启动的影响，具体说来有：1 寻找符合启动条件的 Activity，如果有多个，让用户选择；2 校验启动参数的合法性；3 返回 int 参数，代表 Activity 是否启动成功。
- ActivityStackSupervisior：这个类的作用你从它的名字就可以看出来，它用来管理任务栈。
- ActivityStack：用来管理任务栈里的 Activity。
- ActivityThread：最终干活的人，是 ActivityThread 的内部类，Activity、Service、BroadcastReceiver 的启动、切换、调度等各种操作都在这个类里完成。

注：这里单独提一下 ActivityStackSupervisior，这是高版本才有的类，它用来管理多个 ActivityStack，早期的版本只有一个 ActivityStack 对应着手机屏幕，后来高版本支持多屏以后，就
有了多个 ActivityStack，于是就引入了 ActivityStackSupervisior 用来管理多个 ActivityStack。

整个流程主要涉及四个进程：

- 调用者进程，如果是在桌面启动应用就是 Launcher 应用进程。
- ActivityManagerService 等所在的 System Server 进程，该进程主要运行着系统服务组件。
- Zygote 进程，该进程主要用来 fork 新进程。
- 新启动的应用进程，该进程就是用来承载应用运行的进程了，它也是应用的主线程（新创建的进程就是主线程），处理组件生命周期、界面绘制等相关事情。

有了以上的理解，整个流程可以概括如下：

1. 点击桌面应用图标，Launcher 进程将启动 Activity（MainActivity）的请求以 Binder 的方式发送给了 AMS。
2. AMS 接收到启动请求后，交付 ActivityStarter 处理 Intent 和 Flag 等信息，然后再交给 ActivityStackSupervisior/ActivityStack
   处理 Activity 进栈相关流程。同时以 Socket 方式请求 Zygote 进程 fork 新进程。
3. Zygote 接收到新进程创建请求后 fork 出新进程。
4. 在新进程里创建 ActivityThread 对象，新创建的进程就是应用的主线程，在主线程里开启 Looper 消息循环，开始处理创建 Activity。
5. ActivityThread 利用 ClassLoader 去加载 Activity、创建 Activity 实例，并回调 Activity 的 onCreate()方法。这样便完成了 Activity 的启动。

### BroadcastReceiver 与 LocalBroadcastReceiver 有什么区别？

- BroadcastReceiver 是跨应用广播，利用 Binder 机制实现。
- LocalBroadcastReceiver 是应用内广播，利用 Handler 实现，利用了 IntentFilter 的 match 功能，提供消息的发布与接收功能，实现应用内通信，效率比较高。

### Android Handler 机制是做什么的，原理了解吗？

Android 消息循环流程图如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/native/process/android_handler_structure.png"/>

主要涉及的角色如下所示：

- Message：消息，分为硬件产生的消息（例如：按钮、触摸）和软件产生的消息。
- MessageQueue：消息队列，主要用来向消息池添加消息和取走消息。
- Looper：消息循环器，主要用来把消息分发给相应的处理者。
- Handler：消息处理器，主要向消息队列发送各种消息以及处理各种消息。

整个消息的循环流程还是比较清晰的，具体说来：

1. Handler 通过 sendMessage()发送消息 Message 到消息队列 MessageQueue。
2. Looper 通过 loop()不断提取触发条件的 Message，并将 Message 交给对应的 target handler 来处理。
3. target handler 调用自身的 handleMessage()方法来处理 Message。

事实上，在整个消息循环的流程中，并不只有 Java 层参与，很多重要的工作都是在 C++层来完成的。我们来看下这些类的调用关系。

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/native/process/android_handler_class.png"/>

注：虚线表示关联关系，实线表示调用关系。

在这些类中 MessageQueue 是 Java 层与 C++层维系的桥梁，MessageQueue 与 Looper 相关功能都通过 MessageQueue 的 Native 方法来完成，而其他虚线连接的类只有关联关系，并没有
直接调用的关系，它们发生关联的桥梁是 MessageQueue。

### Android Binder 机制是做什么的，为什么选用 Binder，原理了解吗？

Android Binder 是用来做进程通信的，Android 的各个应用以及系统服务都运行在独立的进程中，它们的通信都依赖于 Binder。

为什么选用 Binder，在讨论这个问题之前，我们知道 Android 也是基于 Linux 内核，Linux 现有的进程通信手段有以下几种：

1. 管道：在创建时分配一个 page 大小的内存，缓存区大小比较有限；
2. 消息队列：信息复制两次，额外的 CPU 消耗；不合适频繁或信息量大的通信；
3. 共享内存：无须复制，共享缓冲区直接付附加到进程虚拟地址空间，速度快；但进程间的同步问题操作系统无法实现，必须各进程利用同步工具解决；
4. 套接字：作为更通用的接口，传输效率低，主要用于不通机器或跨网络的通信；
5. 信号量：常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。6. 信号: 不适用于信息交换，更适用于进程中断控制，比如非法内存访问，杀死某个进程等；

既然有现有的 IPC 方式，为什么重新设计一套 Binder 机制呢。主要是出于以上三个方面的考量：

- 高性能：从数据拷贝次数来看 Binder 只需要进行一次内存拷贝，而管道、消息队列、Socket 都需要两次，共享内存不需要拷贝，Binder 的性能仅次于共享内存。
- 稳定性：上面说到共享内存的性能优于 Binder，那为什么不适用共享内存呢，因为共享内存需要处理并发同步问题，控制负责，容易出现死锁和资源竞争，稳定性较差。而 Binder 基于 C/S 架构，客户端与服务端彼此独立，稳定性较好。
- 安全性：我们知道 Android 为每个应用分配了 UID，用来作为鉴别进程的重要标志，Android 内部也依赖这个 UID 进行权限管理，包括 6.0 以前的固定权限和 6.0 以后的动态权限，传荣 IPC 只能由用户在数据包里填入 UID/PID，这个标记完全
  是在用户空间控制的，没有放在内核空间，因此有被恶意篡改的可能，因此 Binder 的安全性更高。

### 描述一下 Activity 的生命周期，这些生命周期是如何管理的？

Activity 与 Fragment 生命周期如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/complete_android_fragment_lifecycle.png"/>

读者可以从上图看出，Activity 有很多种状态，状态之间的变化也比较复杂，在众多状态中，只有三种是常驻状态：

- Resumed（运行状态）：Activity 处于前台，用户可以与其交互。
- Paused（暂停状态）：Activity 被其他 Activity 部分遮挡，无法接受用户的输入。
- Stopped（停止状态）：Activity 被完全隐藏，对用户不可见，进入后台。

其他的状态都是中间状态。

我们再来看看生命周期变化时的整个调度流程，生命周期调度流程图如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/activity_lifecycle_structure.png" />

所以你可以看到，整个流程是这样的：

1. 比方说我们点击跳转一个新 Activity，这个时候 Activity 会入栈，同时它的生命周期也会从 onCreate()到 onResume()开始变换，这个过程是在 ActivityStack 里完成的，ActivityStack
   是运行在 Server 进程里的，这个时候 Server 进程就通过 ApplicationThread 的代理对象 ApplicationThreadProxy 向运行在 app 进程 ApplicationThread 发起操作请求。
2. ApplicationThread 接收到操作请求后，因为它是运行在 app 进程里的其他线程里，所以 ApplicationThread 需要通过 Handler 向主线程 ActivityThread 发送操作消息。
3. 主线程接收到 ApplicationThread 发出的消息后，调用主线程 ActivityThread 执行响应的操作，并回调 Activity 相应的周期方法。

注：这里提到了主线程 ActivityThread，更准确来说 ActivityThread 不是线程，因为它没有继承 Thread 类或者实现 Runnable 接口，它是运行在应用主线程里的对象，那么应用的主线程
到底是什么呢？从本质上来讲启动启动时创建的进程就是主线程，线程和进程处理是否共享资源外，没有其他的区别，对于 Linux 来说，它们都只是一个 struct 结构体。

### Activity 的通信方式有哪些？

- startActivityForResult
- EventBus
- LocalBroadcastReceiver

### Android 应用里有几种 Context 对象，

Context 类图如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/context_uml.png" width="600" />

可以发现 Context 是个抽象类，它的具体实现类是 ContextImpl，ContextWrapper 是个包装类，内部的成员变量 mBase 指向的也是个 ContextImpl 对象，ContextImpl 完成了
实际的功能，Activity、Service 与 Application 都直接或者间接的继承 ContextWrapper。

### 描述一下进程和 Application 的生命周期？

一个安装的应用对应一个 LoadedApk 对象，对应一个 Application 对象，对于四大组件，Application 的创建和获取方式也是不尽相同的，具体说来：

- Activity：通过 LoadedApk 的 makeApplication()方法创建。
- Service：通过 LoadedApk 的 makeApplication()方法创建。
- 静态广播：通过其回调方法 onReceive()方法的第一个参数指向 Application。
- ContentProvider：无法获取 Application，因此此时 Application 不一定已经初始化。

### Android 哪些情况会导致内存泄漏，如何分析内存泄漏？

常见的产生内存泄漏的情况如下所示：

- 持有静态的 Context（Activity）引用。
- 持有静态的 View 引用，
- 内部类&匿名内部类实例无法释放（有延迟时间等等），而内部类又持有外部类的强引用，导致外部类无法释放，这种匿名内部类常见于监听器、Handler、Thread、TimerTask
- 资源使用完成后没有关闭，例如：BraodcastReceiver，ContentObserver，File，Cursor，Stream，Bitmap。
- 不正确的单例模式，比如单例持有 Activity。
- 集合类内存泄漏，如果一个集合类是静态的（缓存 HashMap），只有添加方法，没有对应的删除方法，会导致引用无法被释放，引发内存泄漏。
- 错误的覆写了 finalize()方法，finalize()方法执行执行不确定，可能会导致引用无法被释放。

查找内存泄漏可以使用 Android Profiler 工具或者利用 LeakCanary 工具。

### Android 有哪几种进程，是如何管理的？

Android 的进程主要分为以下几种：

**前台进程**

用户当前操作所必需的进程。如果一个进程满足以下任一条件，即视为前台进程：

- 托管用户正在交互的 Activity（已调用 Activity 的 onResume() 方法）
- 托管某个 Service，后者绑定到用户正在交互的 Activity
- 托管正在“前台”运行的 Service（服务已调用 startForeground()）
- 托管正执行一个生命周期回调的 Service（onCreate()、onStart() 或 onDestroy()）
- 托管正执行其 onReceive() 方法的 BroadcastReceiver

通常，在任意给定时间前台进程都为数不多。只有在内存不足以支持它们同时继续运行这一万不得已的情况下，系统才会终止它们。 此时，设备往往已达到内存分页状态，因此需要终止一些前台进程来确保用户界面正常响应。

**可见进程**

没有任何前台组件、但仍会影响用户在屏幕上所见内容的进程。 如果一个进程满足以下任一条件，即视为可见进程：

- 托管不在前台、但仍对用户可见的 Activity（已调用其 onPause() 方法）。例如，如果前台 Activity 启动了一个对话框，允许在其后显示上一 Activity，则有可能会发生这种情况。
- 托管绑定到可见（或前台）Activity 的 Service。

可见进程被视为是极其重要的进程，除非为了维持所有前台进程同时运行而必须终止，否则系统不会终止这些进程。

**服务进程**

正在运行已使用 startService() 方法启动的服务且不属于上述两个更高类别进程的进程。尽管服务进程与用户所见内容没有直接关联，但是它们通常在执行一些用户关
心的操作（例如，在后台播放音乐或从网络下载数据）。因此，除非内存不足以维持所有前台进程和可见进程同时运行，否则系统会让服务进程保持运行状态。

**后台进程**

包含目前对用户不可见的 Activity 的进程（已调用 Activity 的 onStop() 方法）。这些进程对用户体验没有直接影响，系统可能随时终止它们，以回收内存供前台进程、可见进程或服务进程使用。 通常会有很多后台进程在运行，因此它们会保存在 LRU （最近最少使用）列表中，以确保包含用户最近查看的 Activity 的进程最后一个被终止。如果某个 Activity 正确实现了生命周期方法，并保存了其当前状态，则终止其进程不会对用户体验产生明显影响，因为当用户导航回该 Activity 时，Activity 会恢复其所有可见状态。

**空进程**

不含任何活动应用组件的进程。保留这种进程的的唯一目的是用作缓存，以缩短下次在其中运行组件所需的启动时间。 为使总体系统资源在进程缓存和底层内核缓存之间保持平衡，系统往往会终止这些进程。

ActivityManagerService 负责根据各种策略算法计算进程的 adj 值，然后交由系统内核进行进程的管理。

### SharePreference 性能优化，可以做进程同步吗？

在 Android 中, SharePreferences 是一个轻量级的存储类，特别适合用于保存软件配置参数。使用 SharedPreferences 保存数据，其背后是用 xml 文件存放数据，文件
存放在/data/data/ < package name > /shared_prefs 目录下.

之所以说 SharedPreference 是一种轻量级的存储方式，是因为它在创建的时候会把整个文件全部加载进内存，如果 SharedPreference 文件比较大，会带来以下问题：

1. 第一次从 sp 中获取值的时候，有可能阻塞主线程，使界面卡顿、掉帧。
2. 解析 sp 的时候会产生大量的临时对象，导致频繁 GC，引起界面卡顿。
3. 这些 key 和 value 会永远存在于内存之中，占用大量内存。

优化建议

1. 不要存放大的 key 和 value，会引起界面卡、频繁 GC、占用内存等等。
2. 毫不相关的配置项就不要放在在一起，文件越大读取越慢。
3. 读取频繁的 key 和不易变动的 key 尽量不要放在一起，影响速度，如果整个文件很小，那么忽略吧，为了这点性能添加维护成本得不偿失。
4. 不要乱 edit 和 apply，尽量批量修改一次提交，多次 apply 会阻塞主线程。
5. 尽量不要存放 JSON 和 HTML，这种场景请直接使用 JSON。
6. SharedPreference 无法进行跨进程通信，MODE_MULTI_PROCESS 只是保证了在 API 11 以前的系统上，如果 sp 已经被读取进内存，再次获取这个 SharedPreference 的时候，如果有这个 flag，会重新读一遍文件，仅此而已。

### 如何做 SQLite 升级？

数据库升级增加表和删除表都不涉及数据迁移，但是修改表涉及到对原有数据进行迁移。升级的方法如下所示：

1. 将现有表命名为临时表。
2. 创建新表。
3. 将临时表的数据导入新表。
4. 删除临时表。

重写

如果是跨版本数据库升级，可以由两种方式，如下所示：

1. 逐级升级，确定相邻版本与现在版本的差别，V1 升级到 V2,V2 升级到 V3，依次类推。
2. 跨级升级，确定每个版本与现在数据库的差别，为每个 case 编写专门升级大代码。

### 进程保护如何做，如何唤醒其他进程？

进程保活主要有两个思路：

1. 提升进程的优先级，降低进程被杀死的概率。
2. 拉活已经被杀死的进程。

如何提升优先级，如下所示：

监控手机锁屏事件，在屏幕锁屏时启动一个像素的 Activity，在用户解锁时将 Activity 销毁掉，前台 Activity 可以将进程变成前台进程，优先级升级到最高。

如果拉活

利用广播拉活 Activity。

### 理解序列化吗，Android 为什么引入 Parcelable？

所谓序列化就是将对象变成二进制流，便于存储和传输。

- Serializable 是 java 实现的一套序列化方式，可能会触发频繁的 IO 操作，效率比较低，适合将对象存储到磁盘上的情况。
- Parcelable 是 Android 提供一套序列化机制，它将序列化后的字节流写入到一个共性内存中，其他对象可以从这块共享内存中读出字节流，并反序列化成对象。因此效率比较高，适合在对象间或者进程间传递信息。

### 如何计算一个 Bitmap 占用内存的大小，怎么保证加载 Bitmap 不产生内存溢出？

Bitamp 占用内存大小 = 宽度像素 x （inTargetDensity / inDensity） x 高度像素 x （inTargetDensity / inDensity）x 一个像素所占的内存

注：这里 inDensity 表示目标图片的 dpi（放在哪个资源文件夹下），inTargetDensity 表示目标屏幕的 dpi，所以你可以发现 inDensity 和 inTargetDensity 会对 Bitmap 的宽高
进行拉伸，进而改变 Bitmap 占用内存的大小。

在 Bitmap 里有两个获取内存占用大小的方法。

- getByteCount()：API12 加入，代表存储 Bitmap 的像素需要的最少内存。
- getAllocationByteCount()：API19 加入，代表在内存中为 Bitmap 分配的内存大小，代替了 getByteCount() 方法。

在不复用 Bitmap 时，getByteCount() 和 getAllocationByteCount 返回的结果是一样的。在通过复用 Bitmap 来解码图片时，那么 getByteCount() 表示新解码图片占用内存的大
小，getAllocationByteCount() 表示被复用 Bitmap 真实占用的内存大小（即 mBuffer 的长度）。

为了保证在加载 Bitmap 的时候不产生内存溢出，可以受用 BitmapFactory 进行图片压缩，主要有以下几个参数：

- BitmapFactory.Options.inPreferredConfig：将 ARGB_8888 改为 RGB_565，改变编码方式，节约内存。
- BitmapFactory.Options.inSampleSize：缩放比例，可以参考 Luban 那个库，根据图片宽高计算出合适的缩放比例。
- BitmapFactory.Options.inPurgeable：让系统可以内存不足时回收内存。

### Android 里的内存缓存和磁盘缓存是怎么实现的。

内存缓存基于 LruCache 实现，磁盘缓存基于 DiskLruCache 实现。这两个类都基于 Lru 算法和 LinkedHashMap 来实现。

LRU 算法可以用一句话来描述，如下所示：

> LRU 是 Least Recently Used 的缩写，最近最久未使用算法，从它的名字就可以看出，它的核心原则是如果一个数据在最近一段时间没有使用到，那么它在将来被
> 访问到的可能性也很小，则这类数据项会被优先淘汰掉。

LruCache 的原理是利用 LinkedHashMap 持有对象的强引用，按照 Lru 算法进行对象淘汰。具体说来假设我们从表尾访问数据，在表头删除数据，当访问的数据项在链表中存在时，则将该数据项移动到表尾，否则在表尾新建一个数据项。当链表容量超过一定阈值，则移除表头的数据。

为什么会选择 LinkedHashMap 呢？

这跟 LinkedHashMap 的特性有关，LinkedHashMap 的构造函数里有个布尔参数 accessOrder，当它为 true 时，LinkedHashMap 会以访问顺序为序排列元素，否则以插入顺序为序排序元素。

DiskLruCache 与 LruCache 原理相似，只是多了一个 journal 文件来做磁盘文件的管理和迎神，如下所示：

``
libcore.io.DiskLruCache
1
1
1

DIRTY 1517126350519
CLEAN 1517126350519 5325928
REMOVE 1517126350519

````

注：这里的缓存目录是应用的缓存目录/data/data/pckagename/cache，未root的手机可以通过以下命令进入到该目录中或者将该目录整体拷贝出来：

```java

//进入/data/data/pckagename/cache目录
adb shell
run-as com.your.packagename
cp /data/data/com.your.packagename/

//将/data/data/pckagename目录拷贝出来
adb backup -noapk com.your.packagename
````

我们来分析下这个文件的内容：

- 第一行：libcore.io.DiskLruCache，固定字符串。
- 第二行：1，DiskLruCache 源码版本号。
- 第三行：1，App 的版本号，通过 open()方法传入进去的。
- 第四行：1，每个 key 对应几个文件，一般为 1.
- 第五行：空行
- 第六行及后续行：缓存操作记录。

第六行及后续行表示缓存操作记录，关于操作记录，我们需要了解以下三点：

1. DIRTY 表示一个 entry 正在被写入。写入分两种情况，如果成功会紧接着写入一行 CLEAN 的记录；如果失败，会增加一行 REMOVE 记录。注意单独只有 DIRTY 状态的记录是非法的。
2. 当手动调用 remove(key)方法的时候也会写入一条 REMOVE 记录。
3. READ 就是说明有一次读取的记录。
4. CLEAN 的后面还记录了文件的长度，注意可能会一个 key 对应多个文件，那么就会有多个数字。

### 了解插件化和热修复吗，它们有什么区别，理解它们的原理吗？

- 插件化：插件化是体现在功能拆分方面的，它将某个功能独立提取出来，独立开发，独立测试，再插入到主应用中。依次来较少主应用的规模。
- 热修复：热修复是体现在 bug 修复方面的，它实现的是不需要重新发版和重新安装，就可以去修复已知的 bug。

利用 PathClassLoader 和 DexClassLoader 去加载与 bug 类同名的类，替换掉 bug 类，进而达到修复 bug 的目的，原理是在 app 打包的时候阻止类打上 CLASS_ISPREVERIFIED 标志，然后在
热修复的时候动态改变 BaseDexClassLoader 对象间接引用的 dexElements，替换掉旧的类。

### 关于性能优化你有什么实践经验？

# 应用优化

Android 的应用优化可以从以下几个角度去考虑。

性能优化

1. 节制的使用 Service，当启动一个 Service 时，系统总是倾向于保留这个 Service 依赖的进程，这样会造成系统资源的浪费，可以使用 IntentService，执行完成任务后会自动停止。
2. 当界面不可见时释放内存，可以重写 Activity 的 onTrimMemory()方法，然后监听 TRIM_MEMORY_UI_HIDDEN 这个级别，这个级别说明用户离开了页面，可以考虑释放内存和资源。
3. 避免在 Bitmap 浪费过多的内存，使用压缩过的图片，也可以使用 Fresco 等库来优化对 Bitmap 显示的管理。
4. 使用优化过的数据集合 SparseArray 代替 HashMap，HashMap 为每个键值都提供一个对象入口，使用 SparseArray 可以免去基本对象类型转换为引用数据类想的时间。

布局优化

1. 使用 include 复用布局文件。
2. 使用 merge 标签避免嵌套布局。
3. 使用 stub 标签仅在需要的时候在展示出来。

高性能编码

1. 避免创建不必要的对象，尽可能避免频繁的创建临时对象，例如在 for 循环内，减少 GC 的次数。
2. 尽量使用基本数据类型代替引用数据类型。
3. 静态方法调用效率高于动态方法，也可以避免创建额外对象。
4. 对于基本数据类型和 String 类型的常量要使用 static final 修饰，这样常量会在 dex 文件的初始化器中进行初始化，使用的时候可以直接使用。
5. 多使用系统 API，例如数组拷贝 System.arrayCopy()方法，要比我们用 for 循环效率快 9 倍以上，因为系统 API 很多都是通过底层的汇编模式执行的，效率比较高。
