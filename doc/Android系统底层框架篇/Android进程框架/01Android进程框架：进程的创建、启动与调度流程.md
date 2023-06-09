## Android 进程框架：进程的创建、启动与调度流程

**文章目录**

- 一 进程的创建与启动流程
- 二 进程的优先级
- 三 进程的调度流程

Android 系统的启动流程如下图所示：
![](../../../art/native/process/android_process.png)

##### Loader 层

1. 当手机处于关机状态时，长按电源键开机，引导芯片开始从固化在 Boot ROM 里的预设代码开始执行，然后加载引导程序 Boot Loader 到 RAM。
2. Boot Loader 被加载到 RAM 之后开始执行，该程序主要完成检查 RAM，初始化硬件参数等功能。

##### Kernel 层

3. 引导程序之后进入 Android 内核层，先启动 swapper 进程（idle 进程），该进程用来初始化进程管理、内存管理、加载 Display、Camera Driver、Binder Driver 等相关工作。
4. swapper 进程进程之后再启动 kthreadd 进程，该进程会创建内核工作线程 kworkder、软中断线程 ksoftirqd、thernal 等内核守护进程，kthreadd 进程是所有内核进程的鼻祖。

##### Native 层

5. 接着会启动 init 进程，init 进程是所有用户进程的鼻祖，它会接着孵化出 ueventd、logd、healthd、installd、adbd、lmkd 等用户守护进程，启动 ServiceManager 来管理系统
   服务，启动 Bootnaim 开机动画。
6. init 进程通过解析 init.rc 文件 fork 生成 Zygote 进程，该进程是 Android 系统第一个 Java 进程，它是所有 Java 进程父进程，该进程主要完成了加载 ZygoteInit 类，注册 Zygote Socket
   服务套接字；加载虚拟机；预加载 Class；预加载 Resources。

##### Framework 层

7. init 进程接着 fork 生成 Media Server 进程，该进程负责启动和管理整个 C++ Framwork（包含 AudioFlinger、Camera Service 等服务）。

8. Zygote 进程接着会 fork 生成 System Server 进程，该进程负责启动和管理整个 Java Framwork（包含 ActivityManagerService、WindowManagerService 等服务）。

##### App 层

Zygote 进程孵化出的第一个应用进程是 Launcher 进程（桌面），它还会孵化出 Browser 进程（浏览器）、Phone 进程（电话）等。我们每个创建的应用都是一个单独的进程。

通过上述流程的分析，想必读者已经对 Android 的整个进程模型有了大致的理解。作为一个应用开发者我们往往更为关注 Framework 层和 App 层里进程的创建与管理相关原理，我们来
一一分析。

## 一 进程的创建与启动流程

在正式介绍进程之前，我们来思考一个问题，何为进程，进程的本质是什么？🤔

> 我们知道，代码是静态的，有代码和资源组成的系统要想运行起来就需要一种动态的存在，进程就是程序的动态执行过程。何为进程？
> 进程就是处理执行状态的代码以及相关资源的集合，包括代码端段、文件、信号、CPU 状态、内存地址空间等。

进程使用 task_struct 结构体来描述，如下所示：

- 代码段：编译后形成的一些指令
- 数据段：程序运行时需要的数据
  - 只读数据段：常量
  - 已初始化数据段：全局变量，静态变量
  - 未初始化数据段（bss)：未初始化的全局变量和静态变量
- 堆栈段：程序运行时动态分配的一些内存
- PCB：进程信息，状态标识等

关于进程的更多详细信息，读者可以去翻阅 Linux 相关书籍，这里只是给读者带来一种整体上的理解，我们的重心还是放在进程再 Android 平台上的应用。

在文章开篇的时候，我们提到了系统中运行的各种进程，那么这些进程如何被创建呢？🤔

我们先来看看我们最熟悉的应用进程是如何被创建的，前面我们已经说来每一个应用都运行在一个单独的进程里，当 ActivityManagerService 去启动四大组件时，
如果发现这个组件所在的进程没有启动，就会去创建一个新的进程，启动进程的时机我们在分析四大组件的启动流程的时候也有讲过，这里再总结一下：

- Activity ActivityStackSupervisor.startSpecificActivityLocked()
- Service ActiveServices.bringUpServiceLocked()
- ContentProvider ActivityManagerService.getContentProviderImpl()
  = Broadcast BroadcastQueue.processNextBroadcast()

这个新进程就是 zygote 进程通过复制自身来创建的，新进程在启动的过程中还会创建一个 Binder 线程池（用来做进程通信）和一个消息循环（用来做线程通信）
整个流程如下图所示：
![](../../../art/native/process/process_start_flow.png)

- 1. 当我们点击应用图标启动应用时或者在应用内启动一个带有 process 标签的 Activity 时，都会触发创建新进程的请求，这种请求会先通过 Binder 发送给 system_server 进程，也即是发送给 ActivityManagerService 进行处理。

* 2. system_server 进程会调用 Process.start()方法，会先收集 uid、gid 等参数，然后通过 Socket 方式发送给 Zygote 进程，请求创建新进程。

- 3. Zygote 进程接收到创建新进程的请求后，调用 ZygoteInit.main()方法进行 runSelectLoop()循环体内，当有客户端连接时执行 ZygoteConnection.runOnce()方法，最后 fork 生成新的应用进程。

- 4. 新创建的进程会调用 handleChildProc()方法，最后调用我们非常熟悉的 ActivityThread.main()方法。

注：整个流程会涉及 Binder 和 Socket 两种进程通信方式，这个我们后续会有专门的文章单独分析，这个就不再展开。

整个流程大致就是这样，我们接着来看看具体的代码实现，先来看一张进程启动序列图：
![](../../../art/native/process/process_start_sequence.png)

从第一步到第三步主要是收集整理 uid、gid、groups、target-sdk、nice-name 等一系列的参数，为后续启动新进程做准备。然后调用 openZygoteSocketIfNeeded()方法
打开 Socket 通信，向 zygote 进程发出创建新进程的请求。

注：第二步中的 Process.start()方法是个阻塞操作，它会一直等待进程创建完毕，并返回 pid 才会完成该方法。

我们来重点关注几个关键的函数。

### 1.1 Process.openZygoteSocketIfNeeded(String abi)

关于 Process 类与 Zygote 进程的通信是如何进行的呢？🤔

Process 的静态内部类 ZygoteState 有个成员变量 LocalSocket 对象，它会与 ZygoteInit 类的成员变量 LocalServerSocket 对象建立连接，如下所示：

客户端

```java
public static class ZygoteState {
    final LocalSocket socket;
}
```

服务端

```java
public class ZygoteInit {
    //该Socket与/dev/socket/zygote文件绑定在一起
    private static LocalServerSocket sServerSocket;
}
```

我们来具体看看代码里的实现。

```java
 public static class ZygoteState {

    public static ZygoteState connect(String socketAddress) throws IOException {
        DataInputStream zygoteInputStream = null;
        BufferedWriter zygoteWriter = null;
        //创建LocalSocket对象
        final LocalSocket zygoteSocket = new LocalSocket();

        try {
            //将LocalSocket与LocalServerSocket建立连接，建立连接的过程就是
            //LocalSocket对象在/dev/socket目录下查找一个名称为"zygote"的文件
            //然后将自己与其绑定起来，这样就建立了连接。
            zygoteSocket.connect(new LocalSocketAddress(socketAddress,
                    LocalSocketAddress.Namespace.RESERVED));

            //创建LocalSocket的输入流，以便可以接收Zygote进程发送过来的数据
            zygoteInputStream = new DataInputStream(zygoteSocket.getInputStream());

            //创建LocalSocket的输出流，以便可以向Zygote进程发送数据。
            zygoteWriter = new BufferedWriter(new OutputStreamWriter(
                    zygoteSocket.getOutputStream()), 256);
        } catch (IOException ex) {
            try {
                zygoteSocket.close();
            } catch (IOException ignore) {
            }

            throw ex;
        }

        String abiListString = getAbiList(zygoteWriter, zygoteInputStream);
        Log.i("Zygote", "Process: zygote socket opened, supported ABIS: " + abiListString);

        return new ZygoteState(zygoteSocket, zygoteInputStream, zygoteWriter,
                Arrays.asList(abiListString.split(",")));
    }
}
```

建立 Socket 连接的流程很明朗了，如下所示：

1. 创建 LocalSocket 对象。
2. 将 LocalSocket 与 LocalServerSocket 建立连接，建立连接的过程就是 LocalSocket 对象在/dev/socket 目录下查找一个名称为"zygote"的文件，然后将自己与其绑定起来，这样就建立了连接。
3. 创建 LocalSocket 的输入流，以便可以接收 Zygote 进程发送过来的数据。
4. 创建 LocalSocket 的输出流，以便可以向 Zygote 进程发送数据。

### 1.2 ZygoteInit.main(String argv[])

> ZygoteInit 是 Zygote 进程的启动类，该类会预加载一些类，然后便开启一个循环，等待通过 Socket 发过来的创建新进程的命令，fork 出新的
> 子进程。

ZygoteInit 的入口函数就是 main()方法，如下所示：

```java
public class ZygoteInit {

    public static void main(String argv[]) {
            // Mark zygote start. This ensures that thread creation will throw
            // an error.
            ZygoteHooks.startZygoteNoThreadCreation();

            try {
                //...
                registerZygoteSocket(socketName);
                //...
                //开启循环
                runSelectLoop(abiList);

                closeServerSocket();
            } catch (MethodAndArgsCaller caller) {
                caller.run();
            } catch (Throwable ex) {
                Log.e(TAG, "Zygote died with exception", ex);
                closeServerSocket();
                throw ex;
            }
        }

    // 开启一个选择循环，接收通过Socket发过来的命令，创建新线程
    private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {

        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        //sServerSocket指的是Socket通信的服务端，在fds中的索引为0
        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        //开启循环
        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                //处理轮询状态，当pollFds有时间到来时则往下执行，否则阻塞在这里。
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {

                //采用IO多路复用机制，当接收到客户端发出的连接请求时或者数据处理请求到来时则
                //往下执行，否则进入continue跳出本次循环。
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                //索引为0，即为sServerSocket，表示接收到客户端发来的连接请求。
                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                }
                //索引不为0，表示通过Socket接收来自对端的数据，并执行相应的操作。
                else {
                    boolean done = peers.get(i).runOnce();
                    //处理完成后移除相应的文件描述符。
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
}
```

可以发现 ZygoteInit 在其入口函数 main()方法里调用 runSelectLoop()开启了循环，接收 Socket 发来的请求。请求分为两种：

1. 连接请求
2. 数据请求

没有连接请求时 Zygote 进程会进入休眠状态，当有连接请求到来时，Zygote 进程会被唤醒，调用 acceptCommadPeer()方法创建 Socket 通道 ZygoteConnection

```java
private static ZygoteConnection acceptCommandPeer(String abiList) {
    try {
        return new ZygoteConnection(sServerSocket.accept(), abiList);
    } catch (IOException ex) {
        throw new RuntimeException(
                "IOException during accept()", ex);
    }
}
```

然后调用 runOnce()方法读取连接请求里的数据，然后创建新进程。

此外，连接的过程中服务端接受的到客户端的 connect()操作会执行 accpet()操作，建立连接手，客户端通过 write()写数据，服务端通过 read()读数据。

### 1.3 ZygoteConnection.runOnce()

```java
class ZygoteConnection {

    boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {

            String args[];
            Arguments parsedArgs = null;
            FileDescriptor[] descriptors;

            try {
                //读取客户端发过来的参数列表
                args = readArgumentList();
                descriptors = mSocket.getAncillaryFileDescriptors();
            } catch (IOException ex) {
                Log.w(TAG, "IOException on command socket " + ex.getMessage());
                closeSocket();
                return true;
            }

            //... 参数处理

            try {

                //... 参数处理


                //调用Zygote.forkAndSpecialize（来fork出新进程
                pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                        parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                        parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                        parsedArgs.appDataDir);
            } catch (ErrnoException ex) {
                logAndPrintError(newStderr, "Exception creating pipe", ex);
            } catch (IllegalArgumentException ex) {
                logAndPrintError(newStderr, "Invalid zygote arguments", ex);
            } catch (ZygoteSecurityException ex) {
                logAndPrintError(newStderr,
                        "Zygote security policy prevents request: ", ex);
            }

            try {
                //pid == 0时表示当前是在新创建的子进程重磅执行
                if (pid == 0) {
                    // in child
                    IoUtils.closeQuietly(serverPipeFd);
                    serverPipeFd = null;
                    handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);

                    // should never get here, the child is expected to either
                    // throw ZygoteInit.MethodAndArgsCaller or exec().
                    return true;
                }
                // pid < 0表示创建新进程失败，pid > 0 表示当前是在父进程中执行
                else {
                    // in parent...pid of < 0 means failure
                    IoUtils.closeQuietly(childPipeFd);
                    childPipeFd = null;
                    return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
                }
            } finally {
                IoUtils.closeQuietly(childPipeFd);
                IoUtils.closeQuietly(serverPipeFd);
            }
     }
}
```

该方法主要用来读取进程启动参数，然后调用 Zygote.forkAndSpecialize()方法 fork 出新进程，该方法是创建新进程的核心方法，它主要会陆续调用三个
方法来完成工作：

1. preFork()：先停止 Zygote 进程的四个 Daemon 子线程的运行以及初始化 GC 堆。这四个 Daemon 子线程分别为：Java 堆内存管理现场、堆线下引用队列线程、析构线程与监控线程。
2. nativeForkAndSpecialize()：调用 Linux 系统函数 fork()创建新进程，创建 Java 堆处理的线程池，重置 GC 性能数据，设置进程的信号处理函数，启动 JDWP 线程。
3. postForkCommon()：启动之前停止的 Zygote 进程的四个 Daemon 子线程。

上面的方法都完成会后，新进程会创建完成，并返回 pid，接着就调用 handleChildProc()来启动新进程。handleChildProc()方法会接着调用 RuntimeInit.zygoteInit()来
完成新进程的启动。

### 1.4 RuntimeInit.zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)

这个就是个关键的方法了，它主要用来创建一些运行时环境，我们来看一看。

```java
public class RuntimeInit {

    public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
        redirectLogStreams();
        //创建应用进程的时区和键盘等通用信息
        commonInit();
        //在应用进程中创建一个Binder线程池
        nativeZygoteInit();
        //创建应用信息
        applicationInit(targetSdkVersion, argv, classLoader);
    }
}
```

该方法主要完成三件事：

1. 调用 commonInit()方法创建应用进程的时区和键盘等通用信息。
2. 调用 nativeZygoteInit()方法在应用进程中创建一个 Binder 线程池。
3. 调用 applicationInit(targetSdkVersion, argv, classLoader)方法创建应用信息。

Binder 线程池我们后续的文章会分析，我们重点来看看 applicationInit(targetSdkVersion, argv, classLoader)方法的实现，它主要用来完成应用的创建。

该方法里的 argv 参数指的就是 ActivityThread，该方法会调用 invokeStaticMain()通过反射的方式调用 ActivityThread 类的 main()方法。如下所示：

```java
public class RuntimeInit {

      private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
              throws ZygoteInit.MethodAndArgsCaller {
          //...

          // Remaining arguments are passed to the start class's static main
          invokeStaticMain(args.startClass, args.startArgs, classLoader);
      }

      private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
              throws ZygoteInit.MethodAndArgsCaller {
          Class<?> cl;

          //通过反射调用ActivityThread类的main()方法
          try {
              cl = Class.forName(className, true, classLoader);
          } catch (ClassNotFoundException ex) {
              throw new RuntimeException(
                      "Missing class when invoking static main " + className,
                      ex);
          }

          Method m;
          try {
              m = cl.getMethod("main", new Class[] { String[].class });
          } catch (NoSuchMethodException ex) {
              throw new RuntimeException(
                      "Missing static main on " + className, ex);
          } catch (SecurityException ex) {
              throw new RuntimeException(
                      "Problem getting static main on " + className, ex);
          }
          //...
      }
}
```

走到 ActivityThread 类的 main()方法，我们就很熟悉了，我们知道在 main()方法里，会创建主线程 Looper，并开启消息循环，如下所示：

```java
public final class ActivityThread {

   public static void main(String[] args) {
       //...
       Environment.initForCurrentUser();
       //...
       Process.setArgV0("<pre-initialized>");
       //创建主线程looper
       Looper.prepareMainLooper();

       ActivityThread thread = new ActivityThread();
       //attach到系统进程
       thread.attach(false);

       if (sMainThreadHandler == null) {
           sMainThreadHandler = thread.getHandler();
       }

       //主线程进入循环状态
       Looper.loop();

       throw new RuntimeException("Main thread loop unexpectedly exited");
   }
}
```

前面我们从 Process.start()开始讲起，分析了应用进程的创建及启动流程，既然有启动就会有结束，接下来我们从
Process.killProcess()开始讲起，继续分析进程的结束流程。

## 二 进程的优先级

进程按照优先级大小不同又可以分为实时进程与普通进程。
![](../../../art/native/process/process_priority.png)

prio 值越小表示进程优先级越高，

- 静态优先级：优先级不会随时间改变，内核也不会修改，只能通过系统调用改变 nice 值，优先级映射公式为：static_prio = MAX_RT_PRIO + nice + 20，其中 MAX_RT_PRIO = 100，那么取值区间为[100, 139]；对应普通进程；
- 实时优先级：取值区间为[0, MAX_RT_PRIO -1]，其中 MAX_RT_PRIO = 100，那么取值区间为[0, 99]；对应实时进程；
- 懂爱优先级：调度程序通过增加或者减少进程优先级，来达到奖励 IO 消耗型或按照惩罚 CPU 消耗型的进程的效果。区间范围[0, MX_PRIO-1]，其中 MX_PRIO = 140，那么取值区间为[0,139]；

## 三 进程调度流程

进程的调度在 Process 类里完成。

### 3.1 优先级调度

优先级调度方法

```java
setThreadPriority(int tid, int priority)
```

进程的优先级以及对应的 nice 值如下所示：

- THREAD_PRIORITY_LOWEST 19 最低优先级
- THREAD_PRIORITY_BACKGROUND 10 后台
- THREAD_PRIORITY_LESS_FAVORABLE 1 比默认略低
- THREAD_PRIORITY_DEFAULT 0 默认
- THREAD_PRIORITY_MORE_FAVORABLE -1 比默认略高
- THREAD_PRIORITY_FOREGROUND -2 前台
- THREAD_PRIORITY_DISPLAY -4 显示相关
- THREAD_PRIORITY_URGENT_DISPLAY -8 显示(更为重要)，input 事件
- THREAD_PRIORITY_AUDIO -16 音频相关
- THREAD_PRIORITY_URGENT_AUDIO -19 音频(更为重要)

### 3.2 组优先级调度

进程组优先级调度方法

```java
setProcessGroup(int pid, int group)
setThreadGroup(int tid, int group)
```

组优先级及对应取值

- THREAD_GROUP_DEFAULT -1 仅用于 setProcessGroup，将优先级<=10 的进程提升到-2
- THREAD_GROUP_BG_NONINTERACTIVE 0 CPU 分时的时长缩短
- THREAD_GROUP_FOREGROUND 1 CPU 分时的时长正常
- THREAD_GROUP_SYSTEM 2 系统线程组
- THREAD_GROUP_AUDIO_APP 3 应用程序音频
- THREAD_GROUP_AUDIO_SYS 4 系统程序音频

### 3.3 调度策略

调度策略设置方法

```java
setThreadScheduler(int tid, int policy, int priority)
```

- SCHED_OTHER 默认 标准 round-robin 分时共享策略
- SCHED_BATCH 批处理调度 针对具有 batch 风格（批处理）进程的调度策略
- SCHED_IDLE 空闲调度 针对优先级非常低的适合在后台运行的进程
- SCHED_FIFO 先进先出 实时调度策略，android 暂未实现
- SCHED_RR 循环调度 实时调度策略，android 暂未实现

### 3.4 进程 adj 调度

另外除了这些基本的调度策略，Android 系统还定义了两个和进程相关的状态值，一个就是定义在 ProcessList.java 里的 adj 值，另一个
是定义在 ActivityManager.java 里的 procState 值。

> 定义在 ProcessList.java 文件，oom_adj 划分为 16 级，从-17 到 16 之间取值。

- UNKNOWN_ADJ 16 一般指将要会缓存进程，无法获取确定值
- **CACHED_APP_MAX_ADJ 15 不可见进程的 adj 最大值 1**
- **CACHED_APP_MIN_ADJ 9 不可见进程的 adj 最小值 2**
- SERVICE_B_AD 8 B List 中的 Service（较老的、使用可能性更小）
- PREVIOUS_APP_ADJ 7 上一个 App 的进程(往往通过按返回键)
- HOME_APP_ADJ 6 Home 进程
- SERVICE_ADJ 5 服务进程(Service process)
- HEAVY_WEIGHT_APP_ADJ 4 后台的重量级进程，system/rootdir/init.rc 文件中设置
- **BACKUP_APP_ADJ 3 备份进程 3**
- **PERCEPTIBLE_APP_ADJ 2 可感知进程，比如后台音乐播放 4**
- **VISIBLE_APP_ADJ 1 可见进程(Visible process) 5**
- **FOREGROUND_APP_ADJ 0 前台进程（Foreground process） 6**
- PERSISTENT_SERVICE_ADJ -11 关联着系统或 persistent 进程
- PERSISTENT_PROC_ADJ -12 系统 persistent 进程，比如 telephony
- SYSTEM_ADJ -16 系统进程
- NATIVE_ADJ -17 native 进程（不被系统管理）

更新进程 adj 值的方法定义在 ActivityManagerService 中，分别为：

- updateOomAdjLocked：更新 adj，当目标进程为空，或者被杀则返回 false；否则返回 true;
- computeOomAdjLocked：计算 adj，返回计算后 RawAdj 值;
- applyOomAdjLocked：应用 adj，当需要杀掉目标进程则返回 false；否则返回 true。

那么进程的 adj 值什么时候会被更新呢？🤔

**Activity**

- ActivityManagerService.realStartActivityLocked: 启动 Activity
- ActivityStack.resumeTopActivityInnerLocked: 恢复栈顶 Activity
- ActivityStack.finishCurrentActivityLocked: 结束当前 Activity
- ActivityStack.destroyActivityLocked: 摧毁当前 Activity

**Service**

- ActiveServices.realStartServiceLocked: 启动服务
- ActiveServices.bindServiceLocked: 绑定服务(只更新当前 app)
- ActiveServices.unbindServiceLocked: 解绑服务 (只更新当前 app)
- ActiveServices.bringDownServiceLocked: 结束服务 (只更新当前 app)
- ActiveServices.sendServiceArgsLocked: 在 bringup 或则 cleanup 服务过程调用 (只更新当前 app)

**BroadcastReceiver**

- BroadcastQueue.processNextBroadcast: 处理下一个广播
- BroadcastQueue.processCurBroadcastLocked: 处理当前广播
- BroadcastQueue.deliverToRegisteredReceiverLocked: 分发已注册的广播 (只更新当前 app)

**ContentProvider**

- ActivityManagerService.removeContentProvider: 移除 provider
- ActivityManagerService.publishContentProviders: 发布 provider (只更新当前 app)
- ActivityManagerService.getContentProviderImpl: 获取 provider (只更新当前 app)

另外，Lowmemorykiller 也会根据当前的内存情况逐级进行进程释放，一共有六个级别（上面加粗的部分）：

- CACHED_APP_MAX_ADJ
- CACHED_APP_MIN_ADJ
- BACKUP_APP_ADJ
- PERCEPTIBLE_APP_ADJ
- VISIBLE_APP_ADJ
- FOREGROUND_APP_ADJ

> 定义在 ActivityManager.java 文件，process_state 划分 18 类，从-1 到 16 之间取值

- PROCESS_STATE_CACHED_EMPTY 16 进程处于 cached 状态，且为空进程
- PROCESS_STATE_CACHED_ACTIVITY_CLIENT 15 进程处于 cached 状态，且为另一个 cached 进程(内含 Activity)的 client 进程
- PROCESS_STATE_CACHED_ACTIVITY 14 进程处于 cached 状态，且内含 Activity
- PROCESS_STATE_LAST_ACTIVITY 13 后台进程，且拥有上一次显示的 Activity
- PROCESS_STATE_HOME 12 后台进程，且拥有 home Activity
- PROCESS_STATE_RECEIVER 11 后台进程，且正在运行 receiver
- PROCESS_STATE_SERVICE 10 后台进程，且正在运行 service
- PROCESS_STATE_HEAVY_WEIGHT 9 后台进程，但无法执行 restore，因此尽量避免 kill 该进程
- PROCESS_STATE_BACKUP 8 后台进程，正在运行 backup/restore 操作
- PROCESS_STATE_IMPORTANT_BACKGROUND 7 对用户很重要的进程，用户不可感知其存在
- PROCESS_STATE_IMPORTANT_FOREGROUND 6 对用户很重要的进程，用户可感知其存在
- PROCESS_STATE_TOP_SLEEPING 5 与 PROCESS_STATE_TOP 一样，但此时设备正处于休眠状态
- PROCESS_STATE_FOREGROUND_SERVICE 4 拥有给一个前台 Service
- PROCESS_STATE_BOUND_FOREGROUND_SERVICE 3 拥有给一个前台 Service，且由系统绑定
- PROCESS_STATE_TOP 2 拥有当前用户可见的 top Activity
- PROCESS_STATE_PERSISTENT_UI 1 persistent 系统进程，并正在执行 UI 操作
- PROCESS_STATE_PERSISTENT 0 persistent 系统进程
- PROCESS_STATE_NONEXISTENT -1 不存在的进程

根据上面说描述的 adj 值和 state 值，我们又可以按照重要性程度的不同，将进程划分为五级：

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
