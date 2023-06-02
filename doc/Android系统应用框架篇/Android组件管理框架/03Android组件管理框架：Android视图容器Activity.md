# Android 组件管理框架：Android 视图容器 Activity

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

**文章目录**

- 一 Activity 的启动流程
- 二 Activity 的回退栈
- 三 Activity 的生命周期
- 四 Activity 的启动模式
- 五 Activity 的通信方式

本篇文章我们来分析 Android 的视图容器 Activity，Android 源码分析系列的文章终于写到了 Activity，这个我们最常用，源码也最复杂的一个组件，之前在网上看到过很多关于 Activity 源码
分析的文章，这些文章写得都挺好，它们往往是从 Activity 启动流程这个角度出发，一个一个函数的去分析整个流程。但是这种做法会让文章通篇看去全是源码，而且会让读者产生一个疑问：这么
长的流程，我该如何掌握，掌握了之后有有什么意义吗？🤔

事实上，单纯去分析流程，确实看不出有什么实践意义，因此我们最好能带着日常开发遇到的问题去看源码，例如特殊场景下的生命周期是怎么变化的，为什么会出现 ANR，不同启动模式下对 Activity
入栈出栈有何影响等。我们带着问题去看看源码里是怎么写的，这样更有目的性，不至于迷失在茫茫多的源码中。

好了，闲话不多说，我们开始吧。😁

Activity 作为 Android 最为常用的组件，它的复杂程度是不言而喻的。当我们点击一个应用的图标，应用的 LancherActivity（MainActivity）开始启动，启动请求以一种 IPC 的方式传入 AMS，AMS 开始
处理启动请求，伴随着 Intent 与 Flag 的解析和 Activity 栈的进出，Activity 的生命周期从 onCreate()方法开始变化，最终将界面呈现在用户的面前。

Activity 的复杂性主要体现在两个方面：

- 复杂的启动流程，超长的函数调用链。
- Activity 栈的管理。
- 启动模式、Flag 以及各种场景对 Activity 生命周期的影响。

针对这些问题，我们来一一分析。

我们先来分析 Activity 启动流程，对 Activity 组件有个整体性的认识。

## 一 Activity 的启动流程

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

👉 注：读者可以发现这上面有很多函数有 Locked 后缀，这代表这些函数需要进行多线程同步（synchronized）操作，它们会读写一些多线程共享的数据。

## 二 Activity 的回退栈

要理解 Activity 回退栈，我们就要先理解 Activity 回退栈的功能结构，它的结构图如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/activity_stack_structure.png" width="600" />

主要角色有：

- ActivityRecord：描述栈里的 Activity 相关信息，对应着一个用户界面，是 Activity 管理的最小单位。
- TaskRecord：是一个栈式管理结构，每个 TaskRecord 可能包含一个或多个 ActivityRecord，栈顶的 ActivityRecord 表示当前用户可见的页面。
- ActivityStack：是一个栈式管理结构，每个 ActivityStack 可能包含一个或多个 TaskRecord，栈顶的 TaskRecord 表示当前用户可见的任务。
- ActivityStackSupervisior：管理者多个 ActivityStack，当前只会有一个获取焦点（focused）的 ActivityStack。
- ProcessReocord：保存着属于用一个进程的所有 ActivityRecord，我们知道同一个应用的 Activity 可以运行在不同的进程里，同一个 TaskRecord 里的 ActivityRecord
  可能属于不同 ProcessRecord，反之，运行在不同 TaskRecord 的 ActivityRecord 可能属于同一个 ProcessRecord。

通过上面的图解分析，我想大家应该理解了 Activity 栈里的数据结构，接下来，我们再简单分析下这个数据类里的字段，字段比较多，大家有个印象就行，不必记住。

### 2.1 ActivityRecord

> ActivityRecord 基本上是一个纯数据类，里面包含了 Activity 的各种信息。

- ActivityInfo 从<activity>标签中解析出来的信息，包含 launchMode，permission，taskAffinity 等
- mActivityType Activity 的类型有三种：APPLICATION_ACTIVITY_TYPE(应用)、HOME_ACTIVITY_TYPE(桌面)、RECENTS_ACTIVITY_TYPE(最近使用)
- appToken 当前 ActivityRecord 的标识
- packageName 当前所属的包名，这是由<activity>静态定义的
- processName 当前所属的进程名，大部分情况都是由<activity>静态定义的，但也有例外
- taskAffinity 相同 taskAffinity 的 Activity 会被分配到同一个任务栈中
- intent 启动当前 Activity 的 Intent
- launchedFromUid 启动当前 Activity 的 UID，即发起者的 UID
- launchedFromPackage 启动当前 Activity 的包名，即发起者的包名
- resultTo 在当前 ActivityRecord 看来，resultTo 表示上一个启动它的 ActivityRecord，当需要启动另一个 ActivityRecord，会把自己作为 resultTo，传递给下一个 ActivityRecord
- state ActivityRecord 所处的状态，初始值是 ActivityState.INITIALIZING
- app ActivityRecord 的宿主进程
- task ActivityRecord 的宿主任务
- inHistory 标识当前的 ActivityRecord 是否已经置入任务栈中
- frontOfTask 标识当前的 ActivityRecord 是否处于任务栈的根部，即是否为进入任务栈的第一个 ActivityRecord
- newIntents Intent 数组，用于暂存还没有调度到应用进程 Activity 的 Intent

这个对象在 ActivityStarter 的 startActivityLocked()方法里被构建，下面分析 ActivityStarter 的时候我们会说。

### 2.2 TaskRecord

> TaskRecord 的职责就是管理 ActivityRecord，事实上，我们平时说的任务栈指的就是 TaskRecord，所有 ActivityRecord 都必须要有宿主任务，如果不存在则新建一个。

- taskid TaskRecord 的唯一标识
- taskType 任务栈的类型，等同于 ActivityRecord 的类型，是由任务栈的第一个 ActivityRecord 决定的
- intent 在当前任务栈中启动的第一个 Activity 的 Intent 将会被记录下来，后续如果有相同的 Intent 时，会与已有任务栈的 Intent 进行匹配，如果匹配上了，就不需要再新建一个 TaskRecord 了
- realActivity, origActivity 启动任务栈的 Activity，这两个属性是用包名(CompentName)表示的，real 和 orig 是为了区分 Activity 有无别名(alias)的情况，如果 AndroidManifest.xml 中定义的 Activity 是一个 alias，则此处 real 表示 Activity 的别名，orig 表示真实的 Activity
- affinity TaskRecord 把 Activity 的 affinity 记录下来，后续启动 Activity 时，会从已有的任务栈中匹配 affinity，如果匹配上了，则不需要新建 TaskRecord
- rootAffinity 记录任务栈中最底部 Activity 的 affinity，一经设定后就不再改变
- mActivities 这是 TaskRecord 最重要的一个属性，TaskRecord 是一个栈结构，栈的元素是 ActivityRecord，其内部实现是一个数组 mActivities
- stack 当前 TaskRecord 所在的 ActivityStack

我们前面说过 TaskRecord 是一个栈结构，它里面的函数当然也侧重栈的管理：增删改查。事实上，在内部 TaskRecord 是用 ArrayList 来实现的栈的操作。

- getRootActivity()/getTopActivity()：任务栈有根部和顶部，可以通过这两个函数分别获取到根部和顶部的 ActivityRecord。获取的过程就是对 TaskRecord.mActivities 进行
  遍历，如果 ActivityRecord 的状态不是 finishing，就认为是有效的 ActivityRecord。
- topRunningActivityLocked()：虽然也是从顶至底对任务栈进行遍历获取顶部的 ActivityRecord，但这个函数同 getTopActivity()有区别：输入参数 notTop，表示在遍历的过程中需要排除 notTop 这个 ActivityRecord;
  addActivityToTop()/addActivityAtBottom()：将 ActivityRecord 添加到任务栈的顶部或底部。
- moveActivityToFrontLocked()：该函数将一个 ActivityRecord 移至 TaskRecord 的顶部，实现方法就是先删除已有的，再在栈顶添加一个新的，这个和 Intent.FLAG_ACTIVITY_REORDER_TO_FRONT 相对应。
- setFrontOfTask()：ActivityRecord 有一个属性是 frontOfTask，表示 ActivityRecord 是否为 TaskRecord 的根 Activity。该函数设置 TaskRecord 中所有 ActivityRecord 的 frontOfTask 属性，从栈底往上
  开始遍历，第一个不处于 finishing 状态的 ActivityRecord 的 frontOfTask 属性置成 true，其他都为 false。
- performClearTaskLocked()：清除 TaskRecord 中的 ActivityRecord。这个和 Intent.FLAG_ACTIVITY_CLEAR_TOP 相对应，当启动 Activity 时，设置了 Intent.FLAG_ACTIVITY_CLEAR_TOP 参数，那么在宿主
  TaskRecord 中，待启动 ActivityRecord 之上的其他 ActivityRecord 都会被清除。

基本上就是围绕 ArrayList 进行增删改查操作，再附加上一些状态变化，整个流程还是比较清晰的。

### 2.3 ActivityStack

> ActivityStack 的职责是管理多个任务栈 TaskRecord。

我们都知道 Activity 有着很多生命周期状态，这些状态就是由 ActivityStack 来推动完成的，在 ActivityStack 里，Activity 有九种状态：

- INITIALIZING：初始化
- RESUMED：已显示
- PAUSING：暂停中
- PAUSED：已暂停
- STOPPING：停止中
- STOPPED：已停止
- FINISHING：结束中
- DESTROYING：销毁中
- DESTROYE：已销毁

这些状态的变化示意了 Activity 生命周期的走向。

我们也来简单看一下 ActivityStack 里的一些字段的含义：

- stackId 每一个 ActivityStack 都有一个编号，从 0 开始递增。编号为 0，表示桌面(Launcher)所在的 ActivityStack，叫做 Home Stack
- mTaskHistory TaskRecord 数组，ActivityStack 栈就是通过这个数组实现的
- mPausingActivity 在发生 Activity 切换时，正处于 Pausing 状态的 Activity
- mResumedActivity 当前处于 Resumed 状态的 ActivityRecord
- mStacks ActivityStack 会绑定到一个显示设备上，譬如手机屏幕、投影仪等，在 AMS 中，通过 ActivityDisplay 这个类来抽象表示一个显示设备，ActivityDisplay.mStacks 表示当前已经绑定到显示设备
  的所有 ActivityStack。当执行一次绑定操作时，就会将 ActivityStack.mStacks 这个属性赋值成 ActivityDisplay.mStacks，否则，ActivityStack.mStacks 就为 null。简而言之，当 mStacks 不为 null 时，
  表示当前 ActivityStack 已经绑定到了一个显示设备。

Activity 状态发生变化时，出来要调整 ActivityRecord.state 的状态，还要调整 ActivityRecord 在栈里的位置，事实上，ActivityStack 也是一个栈式的结构，只不过它管理的是 TaskRecord，和 ActivityRecord
相关的操作也是先找到对应的 TaskRecord，再由 TaskRecord 去完成具体的操作。

我们简单的看一下 ActivityStack 里面的方法。

- findTaskLocked()：该函数的功能是找到目标 ActivityRecord(target)所在的任务栈(TaskRecord)，如果找到，则返回栈顶的 ActivityRecord，否则，返回 null
- findActivityLocked()：根据 Intent 和 ActivityInfo 这两个参数可以获取一个 Activity 的包名，该函数会从栈顶至栈底遍历 ActivityStack 中的所有 Activity，如果包名匹配成功，就返回
- moveToFront()：该函数用于将当前的 ActivityStack 挪到前台，执行时，调用 ActivityStack 中的其他一些判定函数
- isAttached()：用于判定当前 ActivityStack 是否已经绑定到显示设备
- isOnHomeDisplay()：用于判定当前是否为默认的显示设备(Display.DEFAULT_DISPLAY)，通常，默认的显示设备就是手机屏幕
- isHomeStack()：用于判定当前 ActivityStack 是否为 Home Stack，即判定当前显示的是否为桌面(Launcher)
- moveTaskToFrontLocked()：该函数用于将指定的任务栈挪到当前 ActivityStack 的最前面。在 Activity 状态变化时，需要对已有的 ActivityStack 中的任务栈进行调整，待显示 Activity 的宿主任务需要挪到前台
- insertTaskAtTop()：将任务插入 ActivityStack 栈顶

注：这里提到了 ActivityDisplay 的概念，这里简单说一下，我们知道 Android 是支持多屏显示的，每个显示屏对应者一个 ActivityDisplay，默认是手机屏幕，ActivityDisplay 是 ActivityStackSupervisior 的一个内部类，
ActivityStackSupervisor 间接通过 ActivityDisplay 来维护多个 ActivityStack 的状态。 ActivityStack 有一个属性是 mStacks，当 mStacks 不为空时，表示 ActivityStack 已经绑定到了显示设备， 其实 ActivityStack.mStacks
只是一个副本，真正的对象在 ActivityDisplay 中，ActivityDisplay 的一些属性如下所示：

- mDisplayId 显示设备的唯一标识
- mDisplay 获取显示设备信息的工具类，
- mDisplayInfo 显示设备信息的数据结构，包括类型、大小、分辨率等
- mStacks 绑定到显示设备上的 ActivityStack

### 2.4 ActivityStackSupervisior

> ActivityStackSupervisior 用来管理 ActivityStack。

ActivityStackSupervisior 的一些常见属性如下所示：

- mHomeStack 主屏(桌面)所在 ActivityStack
- mFocusedStack 表示焦点 ActivityStack，它能够获取用户输入
- mLastFocusedStack 上一个焦点 ActivityStack
- mActivityDisplays 表示当前的显示设备，ActivityDisplay 中绑定了若干 ActivityStack。通过该属性就能间接获取所有 ActivityStack 的信息

ActivityStackSupervisior 里有很多方法与 ActivityStack 里的方法类似，但是 ActivityStackSupervisior 是针对多个 ActivityStack 进行操作。例如：findTaskLocked()， findActivityLocked()， topRunningActivityLocked(),
ensureActivitiesVisibleLocked()。

## 三 Activity 的生命周期

Activity 的生命周期也是个老生常谈的问题，今天我们从源码的角度去分析 Activity 的生命周期是如何驱动的，以及它是如何变化的。

这里贴一张[android-lifecycle](https://github.com/xxv/android-lifecycle)项目关于 Activity 与 Fragment 生命周期图

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

👉 注：这里提到了主线程 ActivityThread，更准确来说 ActivityThread 不是线程，因为它没有继承 Thread 类或者实现 Runnable 接口，它是运行在应用主线程里的对象，那么应用的主线程
到底是什么呢？从本质上来讲启动启动时创建的进程就是主线程，线程和进程处理是否共享资源外，没有其他的区别，对于 Linux 来说，它们都只是一个 struct 结构体。

上述这个流程的函数调用链如下所示：

```java
ActivityThread.handleLaunchActivity
    ActivityThread.handleConfigurationChanged
        ActivityThread.performConfigurationChanged
            ComponentCallbacks2.onConfigurationChanged

    ActivityThread.performLaunchActivity
        LoadedApk.makeApplication
            Instrumentation.callApplicationOnCreate
                Application.onCreate

        Instrumentation.callActivityOnCreate
            Activity.performCreate
                Activity.onCreate

        Instrumentation.callActivityonRestoreInstanceState
            Activity.performRestoreInstanceState
                Activity.onRestoreInstanceState

    ActivityThread.handleResumeActivity
        ActivityThread.performResumeActivity
            Activity.performResume
                Activity.performRestart
                    Instrumentation.callActivityOnRestart
                        Activity.onRestart

                    Activity.performStart
                        Instrumentation.callActivityOnStart
                            Activity.onStart

                Instrumentation.callActivityOnResume
                    Activity.onResume
```

其他的生命周期在变化时调用流程和上面是一样的，读者可以自己举一反三。

启动新的 Activity 发出的消息是 LAUNCH_ACTIVITY，这些消息定义在 ActivityThread 的内部类 H（Handler）里，一共有 54 个，大部分都是我们熟悉的操作。

```java
public static final int LAUNCH_ACTIVITY         = 100;
public static final int PAUSE_ACTIVITY          = 101;
public static final int PAUSE_ACTIVITY_FINISHING= 102;
public static final int STOP_ACTIVITY_SHOW      = 103;
public static final int STOP_ACTIVITY_HIDE      = 104;
public static final int SHOW_WINDOW             = 105;
public static final int HIDE_WINDOW             = 106;
public static final int RESUME_ACTIVITY         = 107;
public static final int SEND_RESULT             = 108;
public static final int DESTROY_ACTIVITY        = 109;
public static final int BIND_APPLICATION        = 110;
public static final int EXIT_APPLICATION        = 111;
public static final int NEW_INTENT              = 112;
public static final int RECEIVER                = 113;
public static final int CREATE_SERVICE          = 114;
public static final int SERVICE_ARGS            = 115;
public static final int STOP_SERVICE            = 116;

public static final int CONFIGURATION_CHANGED   = 118;
public static final int CLEAN_UP_CONTEXT        = 119;
public static final int GC_WHEN_IDLE            = 120;
public static final int BIND_SERVICE            = 121;
public static final int UNBIND_SERVICE          = 122;
public static final int DUMP_SERVICE            = 123;
public static final int LOW_MEMORY              = 124;
public static final int ACTIVITY_CONFIGURATION_CHANGED = 125;
public static final int RELAUNCH_ACTIVITY       = 126;
public static final int PROFILER_CONTROL        = 127;
public static final int CREATE_BACKUP_AGENT     = 128;
public static final int DESTROY_BACKUP_AGENT    = 129;
public static final int SUICIDE                 = 130;
public static final int REMOVE_PROVIDER         = 131;
public static final int ENABLE_JIT              = 132;
public static final int DISPATCH_PACKAGE_BROADCAST = 133;
public static final int SCHEDULE_CRASH          = 134;
public static final int DUMP_HEAP               = 135;
public static final int DUMP_ACTIVITY           = 136;
public static final int SLEEPING                = 137;
public static final int SET_CORE_SETTINGS       = 138;
public static final int UPDATE_PACKAGE_COMPATIBILITY_INFO = 139;
public static final int TRIM_MEMORY             = 140;
public static final int DUMP_PROVIDER           = 141;
public static final int UNSTABLE_PROVIDER_DIED  = 142;
public static final int REQUEST_ASSIST_CONTEXT_EXTRAS = 143;
public static final int TRANSLUCENT_CONVERSION_COMPLETE = 144;
public static final int INSTALL_PROVIDER        = 145;
public static final int ON_NEW_ACTIVITY_OPTIONS = 146;
public static final int CANCEL_VISIBLE_BEHIND = 147;
public static final int BACKGROUND_VISIBLE_BEHIND_CHANGED = 148;
public static final int ENTER_ANIMATION_COMPLETE = 149;
public static final int START_BINDER_TRACKING = 150;
public static final int STOP_BINDER_TRACKING_AND_DUMP = 151;
public static final int MULTI_WINDOW_MODE_CHANGED = 152;
public static final int PICTURE_IN_PICTURE_MODE_CHANGED = 153;
public static final int LOCAL_VOICE_INTERACTION_STARTED = 154;
```

我们前面说到 Activity 的生命周期是由 ActivityStack 来驱动的，应用的 Activity 在切换时，ActivityStack 会对相应的任务战 TaskRecord 进行调整，以前的 Activity 要出栈
销毁或者移动到后台，要显示的 Activity 添加到栈顶。伴随着对任务栈的操作，Activity 的生命周期也在不断的变化。

在 ActivityStack 里与 Activity 生命周期变化有关的函数主要有以下几个：

- startActivityLocked()
- resumeTopActivityLocked()
- completeResumeLocked()
- startPausingLocked()
- completePauseLocked()
- stopActivityLocked()
- activityPausedLocked()
- finishActivityLocked()
- activityDestroyedLocked()

## 四 Activity 的启动模式

> 启动模式会影响 Activity 的启动行为，默认情况下，启动一个 Activity 就是创建一个实例，然后进入回退栈，但是我们可以通过启动模式来改变这种行为，实现不同的交互效果。

那么有哪些设置会影响这种行为呢？🤔

首先是<activity>标签里的参数：

- taskAffinity：指定 Activity 所在的任务栈，<appliocation>和<activity>标签均可设置，如果没有设置 taskAffinity 就是当前包名。
- launchMode：Activity 的启动模式。
- allowTaskReparenting：当启动 Activity 的任务栈接下来转至前台时，Activity 是否能从该任务栈转移至其他任务栈，“true”表示它可以转移，“false”表示它仍须留在启动它的任务栈。
- clearTaskOnLaunch：是否每当从主屏幕重新启动任务时都从中移除根 Activity 之外的所有 Activity，“true”表示始终将任务清除到只剩其根 Activity；“false”表示不做清除。 默认值为“false”。
  该属性只对启动新任务的 Activity（根 Activity）有意义；对于任务中的所有其他 Activity，均忽略该属性。
- alwaysRetainTaskState：系统是否始终保持 Activity 所在任务的状态，“true”表示保持，“false”表示允许系统在特定情况下将任务重置到其初始状态。默认值为“false”。
  该属性只对任务的根 Activity 有意义；对于所有其他 Activity，均忽略该属性。
- finishOnTaskLaunch：每当用户再次启动其任务（在主屏幕上选择任务）时，是否应关闭（完成）现有 Activity 实例，“true”表示应关闭，“false”表示不应关闭。默认值为“false”。

注：更多和标签相关的参数可以参见[activity-element](https://developer.android.com/guide/topics/manifest/activity-element.html#aff)。

然后是 Intent 里的标志位：

- FLAG_ACTIVITY_NEW_TASK：每当用户再次启动其任务（在主屏幕上选择任务）时，是否应关闭（完成）现有 Activity 实例 —“true”表示应关闭，“false”表示不应关闭。 默认值为“false”。
- FLAG_ACTIVITY_SINGLE_TOP：如果正在启动的 Activity 是当前 Activity（位于返回栈的顶部），则 现有实例会接收对 onNewIntent() 的调用，而不是创建 Activity 的新实例。
  正如前文所述，这会产生与 "singleTop"launchMode 值相同的行为。
- FLAG_ACTIVITY_CLEAR_TOP：如果正在启动的 Activity 已在当前任务中运行，则会销毁当前任务顶部的所有 Activity，并通过 onNewIntent() 将此 Intent 传递给 Activity 已恢复的
  实例（现在位于顶部），而不是启动该 Activity 的新实例。

什么情况下需要设置 FLAG_ACTIVITY_NEW_TASK 标志位呢？🤔

一般说来，主要有以下四种情况：

- 调用者不是 Activity Context；
- 调用者 Activity 调用 single Instance；
- 目标 Activity 设置的有 single Instance 或者 single task；
- 调用者处于 finish 状态；

启动模式一共有四种：

- standard：多实例模式，每次启动都会有创建一个实例，默认会进入启动它的那个 Activity 所属的任务栈的栈顶，读者以前可能使用过 Application Context 去启动 Activity，这是情况下会报错，就是因为
  Application Context 没有所谓的任务栈，解决的方式就是给它添加一个 FLAG_ACTIVITY_NEW_TASK 的标志位，创建一个新的任务栈。
- singleTop：栈顶复用模式，如果新启动的 Activity 已经位于任务栈顶，则不会创建新的实例，而是回调原来 Activity 实例的 onNewIntent()方法，如果新启动的 Activity 没有位于任务栈顶，则会创建
  新的 Activity 实例。
- singleTask：栈内复用模式，如果新启动的 Activity 已经位于任务栈内，则不会创建新的实例，而是回调原来 Activity 实例的 onNewIntent()方法并且**清除它之上的 Activity（这里需要注意一下：任务栈里
  的 Activity 是永远不会重排序的，所以它会清楚上方所有的 Activity 来让自己回到栈顶）**，如果新启动的 Activity
  没有位于任务栈，则新建一个 Activity 实例。
- singleInstance：单实例模式，和 singleTask 相似，但是 singleTask 可以在多个栈里拥有多个实例，而 singleInstance 在多个栈里只能有唯一实例，这个一般用在特殊场景里，例如电话界面。

我们再来总结一下这些启动模式的区别：

- standard 和 singleTop 的行为很相近，但遇到相同的栈顶 Activity 实例，standard 会再次新建一个，而 singleTop 不会;
- singleTop 和 singleTask 在找到目标的 Activity 实例时，会调用其 onNewIntent()方法; singleTop 可以存在多个相同的 Activity 实例，而 singleTask 仅存在一个;
- singleTask 和 singleInstance 都只会存在一个相同的 Activity 实例，singleTask 任务中可以有其他不同的 Activity 实例，而 singleInstance 栈中仅有一个 Activity 实例。

我们前面说过，ActivityRecord 里有个 ActivityInfo 成员变量用来描述 Activity 的相关信息，ActivityInfo 是在解析 AndroidManifest.xml 里的<activity>标签获取的。
它里面的 launchMode 对应上面启动模式。

```java
public static final int LAUNCH_MULTIPLE = 0;
public static final int LAUNCH_SINGLE_TOP = 1;
public static final int LAUNCH_SINGLE_TASK = 2;
public static final int LAUNCH_SINGLE_INSTANCE = 3;
```

## 五 Activity 的通信方式

Activity 之间也经常需要传递数据，这个一般通过以下方式来完成。

原始 Activity

```java
startActivityForResult(intent, requestCode, resultCode);

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
}
```

目标 Activity

```java
setResult(resultCode, intent);
```

我们来看下 setResult()方法的实现。

```java
public final void setResult(int resultCode, Intent data) {
    synchronized (this) {
        mResultCode = resultCode;
        mResultData = data;
    }
}
```

就是个简单的赋值操作，这说明会有方法来去这个变量的值，什么时候来取？🤔 根据平时的开发经验，Activity finsh()或者 onBackPress()来取，将这两个值通过
onActivityResult(int requestCode, int resultCode, Intent data)返回给原始 Activity。

我们来梳理一下整个流程。
