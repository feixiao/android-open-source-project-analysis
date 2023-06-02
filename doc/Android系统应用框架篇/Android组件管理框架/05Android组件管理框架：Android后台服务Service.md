# Android 组件管理框架：Android 后台服务 Service

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

**文章目录**

- 一 Service 生命周期
- 二 Service 启动流程
- 三 Service 绑定流程
- 四 Service 与 IntentService

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

本篇文章开始来分析 Service 相关原理，Service 在开发中使用的相对较少，它主要用来处理后台任务。我们来看看官方对它的定义，如下所示：

> Service 是一个可以在后台执行长时间运行操作而不提供用户界面的应用组件。服务可由其他应用组件启动，而且即使用户切换到其他应用，服务仍将在后台继续运行。
> 此外，组件可以绑定到服务，以与之进行交互，甚至是执行进程间通信 (IPC)。 例如，服务可以处理网络事务、播放音乐，执行文件 I/O 或与内容提供程序交互，
> 而所有这一切均可在后台进行。

Service 从使用方式上可以分为两种，如下所示：

- startService()

> 当应用组件（如 Activity）通过调用 startService() 启动服务时，服务即处于“启动”状态。一旦启动，服务即可在后台无限期运行，即使启动服务的组件已被销毁也
> 不受影响。 已启动的服务通常是执行单一操作，而且不会将结果返回给调用方。例如，它可能通过网络下载或上传文件。 操作完成后，服务会自行停止运行。

- bindService()

> 当应用组件通过调用 bindService() 绑定到服务时，服务即处于“绑定”状态。绑定服务提供了一个客户端-服务器接口，允许组件与服务进行交互、发送请求、获取结果，甚
> 至是利用进程间通信 (IPC) 跨进程执行这些操作。 仅当与另一个应用组件绑定时，绑定服务才会运行。 多个组件可以同时绑定到该服务，但全部取消绑定后，该服务即会被
> 销毁。

👉 注：我们虽然分开讨论这两种使用方式，但并不意味着 Service 只有启动状态或者绑定状态，状态的确定只依赖于是否实现了一些回调方法：onStartCommand() 允许组件启动服务，
onBind() 允许组件绑定服务。如果同时实现了这两种回调方法，那一个 Service 可以同时处于启动和绑定状态。

- 如果组件通过调用 startService() 启动服务（这会导致对 onStartCommand() 的调用），则服务将一直运行，直到服务使用 stopSelf() 自行停止运行，或由其他组件通过调用 stopService() 停止它为止。
- 如果组件是通过调用 bindService() 来创建服务（且未调用 onStartCommand()，则服务只会在该组件与其绑定时运行。一旦该服务与所有客户端之间的绑定全部取消，系统便会销毁它。

另外，Service 也是运行在主线程里的，所以它和 Activity 一样，如果在里面执行一些耗时操作，也是会引起 ANR 的，所以 Service 里的耗时操作也需要单独开新线程来处理。

我们接着来看看 Service 的生命周期。👇

## 一 Service 生命周期

Service 和 Activity 一样也有自己的生命周期，只不过没有那么复杂，具体说来，如下所示：

- onStartCommand()：当一个组件通过 startService()去启动一个 Service 时，系统会回调该方法，则该 Service 会启动并会在后台一直执行，除非调用了 stopSelf()方法
  或者 stopService()方法才可以停止 Service。
- onBind()：当一个组件通过 bindService()去绑定一个 Service 时，系统会回调该方法，实现该方法时，需要返回一个 IBinder 对象供客户端与服务端进行通信。
- onCreate()：首次创建该服务的时候会回调该方法，如果服务已经运行，则不会回调。
- onDestroy（）：当服务不再使用且被销毁时回调该方法，通常会在该方法里清理一些线程、监听器、接收器等资源。

nStartCommand()的返回值用来表示系统如何在 Service 停止的情况下继续运行 Service，如下所示：

- START_NOT_STICKY：如果系统在 onStartCommand() 返回后终止服务，则除非有挂起 Intent 要传递，否则系统不会重建服务。这是最安全的选项，可以避免在不必要时以及应用
  能够轻松重启所有未完成的作业时运行服务。
- START_STICKY：如果系统在 onStartCommand() 返回后终止服务，则会重建服务并调用 onStartCommand()，但不会重新传递最后一个 Intent。相反，除非有挂起 Intent 要启动服务（在这种情况下，将传递这些 Intent ），否则系统会通过空 Intent 调用 onStartCommand()。这适用
  于不执行命令、但无限期运行并等待作业的媒体播放器（或类似服务）。
- START_REDELIVER_INTENT：如果系统在 onStartCommand() 返回后终止服务，则会重建服务，并通过传递给服务的最后一个 Intent 调用 onStartCommand()。任何挂起 Intent 均依次传递。这适用于主动执行应该立即恢复的作业（例如下载文件）的服务。

## 二 Service 启动流程

应用通过 startService()或者 bindService()方法去启动或者绑定 Service 的过程主要是通过 ActivityManagerService 来完成，Service 启动的过程除了 Service 组件的创建
还包括 Service 所在进程（如果没有创建的话）的创建，具体流程如下图所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/service_create_structure.png" height="400"/>

1. ActivityManagerService 通过 Socket 方式向 Zygote 进程请求生成（fork）新的进程用来承载 Service。
2. Zygote 进程调用 fork()方法创建新的进程，并将 ActivityThread 相关资源加载到新进程。
3. 新进程创建完成以后，ActivityMangerService 通过 Binder 方式向新生成的 ActivityThread 进程请求创建 Service。
4. Service 创建完成以后，ActivityThread 启动 Service。

Service 启动流程序列图如下图所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/service_start_sequence.png"/>

从整个序列图我们还可以看出，Service 的启动流程涉及到 4 个进程，按颜色划分，如下所示：

- 启动者 Activity 所在进程
- 被启动者 Service 所在进程
- ActivityServiceManager 所在进程（system_server 进程）
- Zygote 进程

Service 的启动流程如下所示：

1. Activity 所在进程进程采用 Binder IPC 向 system_server 进程发起 startService 请求；
2. system_server 进程接收到请求后，向 zygote 进程发送创建进程的请求；
3. zygote 进程 fork 出新的子进程 Remote Service 进程；
4. Remote Service 进程，通过 Binder IPC 向 sytem_server 进程发起 attachApplication 请求；
5. system_server 进程在收到请求后，进行一系列准备工作后，再通过 binder IPC 向 remote Service 进程发送 scheduleCreateService 请求；
6. Remote Service 进程的 binder 线程在收到请求后，通过 handler 向主线程发送 CREATE_SERVICE 消息；
7. 主线程在收到 Message 后，通过发射机制创建目标 Service，并回调 Service.onCreate()方法。

从上述序列图可以看出，最终创建 Service 的是 ApplicationThread 里的

```java
public final class ActivityThread {

    private void handleCreateService(CreateServiceData data) {
            //当应用处于后台即将进行GC，而此时又被调回到活动状态，则跳过此次GC。
            unscheduleGcIdler();

            //LoadedApk描述一个呗加载到系统里的APK
            LoadedApk packageInfo = getPackageInfoNoCheck(
                    data.info.applicationInfo, data.compatInfo);
            Service service = null;
            try {
                java.lang.ClassLoader cl = packageInfo.getClassLoader();
                // 1. 通过反射创建Service对象。
                service = (Service) cl.loadClass(data.info.name).newInstance();
            } catch (Exception e) {
                if (!mInstrumentation.onException(service, e)) {
                    throw new RuntimeException(
                        "Unable to instantiate service " + data.info.name
                        + ": " + e.toString(), e);
                }
            }

            try {
                if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

                // 2. 创建ContextImpl对象。
                ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
                context.setOuterContext(service);

                // 3. 创建Application对象。
                Application app = packageInfo.makeApplication(false, mInstrumentation);
                service.attach(context, this, data.info.name, data.token, app,
                        ActivityManagerNative.getDefault());
                // 4. 调用Service的onCreate()方法。
                service.onCreate();
                mServices.put(data.token, service);
                try {
                    // 5. 调用服务创建完成，执行一些收尾工作。
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            } catch (Exception e) {
                if (!mInstrumentation.onException(service, e)) {
                    throw new RuntimeException(
                        "Unable to create service " + data.info.name
                        + ": " + e.toString(), e);
                }
            }
        }
}
```

整个启动的流程也十分简单，如下所示：

1. 通过反射创建 Service 对象。
2. 创建 ContextImpl 对象。
3. 创建 Application 对象。
4. 调用 Service 的 onCreate()方法。
5. 调用服务创建完成，执行一些收尾工作。

此方法执行完成以后，便走到了 Service 的生命周期方法 onCreate()里了，此时 Service 就被启动起来了。

## 三 Service 绑定流程

Service 绑定流程序列图如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/service_bind_sequence.png"/>

1. ClientActivity 组件向 ActivityManagerService 发送一个绑定 ServerService 组件的进程间通信请求。
2. ActivityManagerService 发现用来运行 ServerService 组件与 ClientActivity 组件运行在同一个进程里，它
   便直接通知该进程将该 erverService 组件启动起来。
3. 该 erverService 组件启动起来以后，ActivityManagerService 就请求它返回一个 Binder 本地对象，以便
   ClientActivity 组件可以通过这个 Binder 对象与 ServerService 组件建立连接。
4. ActivityManagerService 将从 ServerService 组件获得的 Binder 对象返回给调用者 ClientActivity。
5. ClientActivity 获得到 ActivityManagerService 发送给它的 Binder 对象后，它就可以通过这个 BInder 对象
   获得 ServerService 组件的一个访问接口，从而获得 ServerService 的服务，这样便相当于 ServerService 组件
   绑定在 ClientActivity 组件内部了。

## 四 Service 与 IntentService

前面我们说到 Service 也是运行在主线程的，所以 Service 里的耗时操作也会阻塞主线程，通常我们在处理耗时任务的时候会选用 IntentService，它们的区别如下所示：

- Service

> 这是适用于所有服务的基类。扩展此类时，必须创建一个用于执行所有服务工作的新线程，因为默认情况下，服务将使用应用的主线程，这会降低应用正在运行的所有 Activity 的性能。

- IntentService

> 这是 Service 的子类，它内部创建了一个 HandlerThread 来逐一处理所有启动请求，使用的时候只需要实现 onHandleIntent()方法即可，该方法会接收每个启动的请求的 Intent，IntentService
> 可以用来处理耗时操作。

IntentService 相比 Service，多做了以下处理，如下所示：

- 创建默认的工作线程，用于在应用的主线程外执行传递给 onStartCommand() 的所有 Intent。
- 创建工作队列，用于将 Intent 逐一传递给 onHandleIntent() 实现，这样您就永远不必担心多线程问题。
- 在处理完所有启动请求后停止服务，因此无需调用 stopSelf()。
- 提供 onBind() 的默认实现（返回 null）。
- 提供 onStartCommand() 的默认实现，可将 Intent 依次发送到工作队列和 onHandleIntent() 实现。

👉 注：我们在使用 Service 时通常会执行一些耗时的后台任务，为了不阻塞主线程，通常会使用 IntentService。

关于 IntentService 的实现非常简单，我们来看一下，如下所示：

```java
public abstract class IntentService extends Service {
    // 获取消息的Looper，被volatile休书，说明做了线程同步。
    private volatile Looper mServiceLooper;
    // 处理消息的Handler，被volatile休书，说明做了线程同步。
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            // 回调onHandleIntent
            onHandleIntent((Intent)msg.obj);
            // 关闭自己，说明IntentService执行完任务后会关闭自己。
            stopSelf(msg.arg1);
        }
    }

    // 构造函数，name表示的是worker线程的名字
    public IntentService(String name) {
        super();
        mName = name;
    }

    // 设置Intent是否会重新分发，
    // ① 如果为true，则onStartCommand返回START_REDELIVER_INTENT，
    // 说明如果系统在 onStartCommand() 返回后终止服务，则会重建服务，并通过传递给服务的最后一个 Intent
    // 调用 onStartCommand()。任何挂起 Intent 均依次传递。这适用于主动执行应该立即恢复的作业（例如下载文件）的服务。
    // ② 如果为false（默认），则onStartCommand会返回START_NOT_STICKY，说明如果系统在 onStartCommand() 返回后终
    // 止服务，则除非有挂起 Intent 要传递，否则系统不会重建服务。这是最安全的选项，可以避免在不必要时以及应用能够轻松重启所有未完成的作业时运行服务。
    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        // 创建HandlerThread，HandlerThread是一种带有消息循环的线程。
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        // 获取HandlerThread里的Looper
        mServiceLooper = thread.getLooper();
        // 构建该Looper的Handler
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        // 发送消息
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        // 根据mRedelivery判定是否重发INTENT（即重建服务）
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        // 通过startService()启动，不需要提供绑定的Binder代理对象。
        return null;
    }

    // 覆写次方法在Worker线程处理任务
    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}

```

IntentService 整体的实现还是比较简单的。
