# Android 系统应用框架篇：Service 启动流程

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

本篇文章分析 Service 组件在新进程内的启动流程。

启动 Service 组件的流程如下所示：

```
1 向ActivityManagerService发送一个启动Service组件的请求。
2 ActivityManagerService发现用来运行Service组件的进程不存在，它会先保存Service组件的信息，接着再创建一个新的应用进程。
3 新的应用进程创建完成后，就会向ActivityManagerService发送一个启动完成的进程间通信请求，以便ActivityManagerService可
以继续执行启动Service组件的的操作。
4 ActivityManagerService将第2步保存的Service组件信息发送给新床架的应用进程，以便它可以将Service组件启动起来。
```

新进程中启动 Service 组件序列图

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/service_start_sequence.png"/>

#### 1 Activity.startService(Intent service)

当我们在 Activity 里调用 startService(Intent service)方法时，它实际上在调用 ContextWrapper.startService(Intent service)。

#### 2 ContextWrapper.startService(Intent service)

```java
public class ContextWrapper{
    @Override
    public ComponentName startService(Intent service) {
        return mBase.startService(service);
    }
}
```

mBase 的对象类型是 Context，它实际上指向了 Context 的实现类 ContextImpl，所以该方法进一步调用 ContextImpl.startService.

#### 3 ContextImpl.startService(Intent service)

```java
public class ContextImpl{
    @Override
    public ComponentName startService(Intent service) {
        try {
            ComponentName cn = ActivityManagerNative.getDefault().startService(
                mMainThread.getApplicationThread(), service,
                service.resolveTypeIfNeeded(getContentResolver()));
            if (cn != null && cn.getPackageName().equals("!")) {
                throw new SecurityException(
                        "Not allowed to start service " + service
                        + " without permission " + cn.getClassName());
            }
            return cn;
        } catch (RemoteException e) {
            return null;
        }
    }
}
```

ActivityManagerNative.getDefault()获取的是 ActivityManagerService 的一个代理对象，即 ActivityManagerProxy。我们再看看看传递的参数：

```
mMainThread.getApplicationThread(：获取当前应用进程的一个类型为ApplicationThread的Binder本地对象。将它传递给ActivityManagerService，以便
ActivityManagerService知道是谁在请求它启动Service组件。
service：intent对象。
service.resolveTypeIfNeeded(getContentResolver())：返回Intent的MIME类型。
```

该方法进一步调用即 ActivityManagerProxy.startService(IApplicationThread caller, Intent service, String resolvedType)

#### 4 ActivityManagerProxy.startService(IApplicationThread caller, Intent service, String resolvedType)

```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager{

    class ActivityManagerProxy implements IActivityManager{
        public ComponentName startService(IApplicationThread caller, Intent service,
                String resolvedType) throws RemoteException{
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            data.writeInterfaceToken(IActivityManager.descriptor);
            data.writeStrongBinder(caller != null ? caller.asBinder() : null);
            service.writeToParcel(data, 0);
            data.writeString(resolvedType);
            mRemote.transact(START_SERVICE_TRANSACTION, data, reply, 0);
            reply.readException();
            ComponentName res = ComponentName.readFromParcel(reply);
            data.recycle();
            reply.recycle();
            return res;
        }
    }
}

```

将传递金磊的参数封装成 Parcel 对象，然后利用 ActivityManagerProxy 内部的 Binder 对象 mRemote 向 ActivityManagerService 发送一个 START_SERVICE_TRANSACTION 进程间通信请求。
该函数进一步调用 ActivityManagerService.startService(IApplicationThread caller, Intent service, String resolvedType)来处理这个请求。

#### 5 ActivityManagerService.startService(IApplicationThread caller, Intent service, String resolvedType)

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType) {
        // Refuse possible leaked file descriptors
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            ComponentName res = startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }
}
```

这个函数的参数的含义上面第 2 步已经介绍过，它进一步调用 ActivityManagerService.startServiceLocked()方法执行启动 Service 组件的操作。

#### 6 ActivityManagerService.startServiceLocked(IApplicationThread caller, Intent service, String resolvedType, int callingPid, int callingUid)

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    ComponentName startServiceLocked(IApplicationThread caller,
            Intent service, String resolvedType,
            int callingPid, int callingUid) {
        synchronized(this) {
            if (DEBUG_SERVICE) Slog.v(TAG, "startService: " + service
                    + " type=" + resolvedType + " args=" + service.getExtras());

            if (caller != null) {
                final ProcessRecord callerApp = getRecordForAppLocked(caller);
                if (callerApp == null) {
                    throw new SecurityException(
                            "Unable to find app for caller " + caller
                            + " (pid=" + Binder.getCallingPid()
                            + ") when starting service " + service);
                }
            }

            //查找是否有与参数service对应的一个ServiceRecord对象，如果没有则ActivityManagerService就会到PackageManagerService中
            //去获取与参数service对应的一个Service组件信息，并把它封装成一个ServiceRecord对象。
            ServiceLookupResult res =
                retrieveServiceLocked(service, resolvedType,
                        callingPid, callingUid);
            if (res == null) {
                return null;
            }
            if (res.record == null) {
                return new ComponentName("!", res.permission != null
                        ? res.permission : "private to package");
            }
            //每个Service组件都用一个ServiceRecord对象类描述，就行每个Activity组件都用一个ActivityRecord来描述一样。
            ServiceRecord r = res.record;
            int targetPermissionUid = checkGrantUriPermissionFromIntentLocked(
                    callingUid, r.packageName, service);
            if (unscheduleServiceRestartLocked(r)) {
                if (DEBUG_SERVICE) Slog.v(TAG, "START SERVICE WHILE RESTART PENDING: " + r);
            }
            r.startRequested = true;
            r.callStart = false;
            r.lastStartId++;
            if (r.lastStartId < 1) {
                r.lastStartId = 1;
            }
            r.pendingStarts.add(new ServiceRecord.StartItem(r, r.lastStartId,
                    service, targetPermissionUid));
            r.lastActivity = SystemClock.uptimeMillis();
            synchronized (r.stats.getBatteryStats()) {
                r.stats.startRunningLocked();
            }
            //根据上面封装的ServiceRecord对象，调用bringUpServiceLocked()方法进一步启动Service组件的启动操作。
            if (!bringUpServiceLocked(r, service.getFlags(), false)) {
                return new ComponentName("!", "Service process is bad");
            }
            return r.name;
        }
    }
}
```

> ServiceRecord：用来描述 Service 组件信息，每个 Service 组件都用一个 ServiceRecord 对象类描述，就行每个 Activity 组件都用一个 ActivityRecord 来描述一样。

这个函数主要做了两件事情：

```
1 查找是否有与参数service对应的一个ServiceRecord对象，如果没有则ActivityManagerService就会到PackageManagerService中
去获取与参数service对应的一个Service组件信息，并把它封装成一个ServiceRecord对象。
2 根据上面封装的ServiceRecord对象，调用bringUpServiceLocked()方法进一步启动Service组件的启动操作。
```

我们来看 ActivityManagerService.bringUpServiceLocked()方法的实现。

#### 7 ActivityManagerService.bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean whileRestarting)

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

  private final boolean bringUpServiceLocked(ServiceRecord r,
            int intentFlags, boolean whileRestarting) {
        //Slog.i(TAG, "Bring up service:");
        //r.dump("  ");

        if (r.app != null && r.app.thread != null) {
            sendServiceArgsLocked(r, false);
            return true;
        }

        if (!whileRestarting && r.restartDelay > 0) {
            // If waiting for a restart, then do nothing.
            return true;
        }

        if (DEBUG_SERVICE) Slog.v(TAG, "Bringing up " + r + " " + r.intent);

        // We are now bringing the service up, so no longer in the
        // restarting state.
        mRestartingServices.remove(r);

        //获取ServiceRecord里的processName属性
        final String appName = r.processName;
        //然后根据processName属性与Service组件的用户ID去查找ActivityManagerService是否已经存在
        //一个ProcessRecord对象。
        ProcessRecord app = getProcessRecordLocked(appName, r.appInfo.uid);
        if (app != null && app.thread != null) {
            try {
                //如果存在该ProcessRecord对象则在app所描述的应用进程中启动该Service组件
                realStartServiceLocked(r, app);
                return true;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting service " + r.shortName, e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }

        //如果没有查找到该进程，则会去启动一个新的应用进程。
        // Not running -- get it started, and enqueue this service record
        // to be executed when the app comes up.
        if (startProcessLocked(appName, r.appInfo, true, intentFlags,
                "service", r.name, false) == null) {
            Slog.w(TAG, "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": process is bad");
            bringDownServiceLocked(r, true);
            return false;
        }

        if (!mPendingServices.contains(r)) {
            //将该Service组件对象保存在ActivityManagerService的成员变量mPendingService中，表示它是一个
            //正在等待启动的Service组件。
            mPendingServices.add(r);
        }

        return true;
    }
}
```

该函数主要做了两件事情：

```
1 获取ServiceRecord里的processName属性，然后根据processName属性与Service组件的用户ID去查找ActivityManagerService
是否已经存在一个ProcessRecord对象。

如果存在：则在app所描述的应用进程中启动该Service组件
如果不存在：则会去启动一个新的应用进程。

2 将该Service组件对象保存在ActivityManagerService的成员变量mPendingService中，表示它是一个正在等待启动的Service组件。
```

我们分析的是在新进程创建 Service 组件的情况，因此我们接着来看 ActivityManagerService.startProcessLocked()的实现。

#### 8 ActivityManagerService.tartProcessLocked(String processName, ApplicationInfo info, boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName, boolean allowWhileBooting)

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    final ProcessRecord startProcessLocked(String processName,
            ApplicationInfo info, boolean knownToBeDead, int intentFlags,
            String hostingType, ComponentName hostingName, boolean allowWhileBooting) {
        ProcessRecord app = getProcessRecordLocked(processName, info.uid);
        ...
        startProcessLocked(app, hostingType, hostingNameStr);
        return (app.pid != 0) ? app : null;
    }

    private final void startProcessLocked(ProcessRecord app,
            String hostingType, String hostingNameStr) {
            ...
            int pid = Process.start("android.app.ActivityThread",
                    mSimpleProcessManagement ? app.processName : null, uid, uid,
                    gids, debugFlags, null);
            ...
    }
}
```

从这个函数开始就开始创建新的应用进程了，它主要调用 Process 的静态函数 start()来创建一个新的应用进程，这个新进程以 ActivityThread
类的静态成员函数 main()为入口。

#### 9 ActivityThread.main(String[] args)

```java
public final class ActivityThread {
    public static final void main(String[] args) {
            SamplingProfilerIntegration.start();

            Process.setArgV0("<pre-initialized>");

            Looper.prepareMainLooper();
            if (sMainThreadHandler == null) {
                sMainThreadHandler = new Handler();
            }

            ActivityThread thread = new ActivityThread();
            //该方法会去调用ActivityManagerProxy.attachApplication()方法
            thread.attach(false);

            if (false) {
                Looper.myLooper().setMessageLogging(new
                        LogPrinter(Log.DEBUG, "ActivityThread"));
            }

            Looper.loop();

            if (Process.supportsProcesses()) {
                throw new RuntimeException("Main thread loop unexpectedly exited");
            }

            thread.detach();
            String name = (thread.mInitialApplication != null)
                ? thread.mInitialApplication.getPackageName()
                : "<unknown>";
            Slog.i(TAG, "Main thread of " + name + " is now exiting");
        }
}
```

main 函数是新创建进程的入口，该函数会创建一个 ActivityThread 与 ApplicationThread 对象，并调用 ActivityManagerProxy.attachApplication()方
法进一步执行 Service 组件启动操作。

#### 10 ActivityManagerProxy.attachApplication(IApplicationThread app)

```java
class ActivityManagerProxy implements IActivityManager{
    public void attachApplication(IApplicationThread app) throws RemoteException{
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(app.asBinder());
        mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
        reply.readException();
        data.recycle();
        reply.recycle();
    }
}
```

该函数中 ActivityManagerProxy 接受上一步 main 方法创建的 ApplicationThread 对象，并向 ActivityManagerService 发送一个类型为
ATTACH_APPLICATION_TRANSACTION 进程间通信请求，并将 ApplicationThread 对象传递给 ActivityManagerService，以便
ActivityManagerService 可以和这个新创建的进程进行通信。

#### 11 ActivityManagerService.attachApplication(IApplicationThread thread)

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
}
```

该方法进一步调用 ActivityManagerService.attachApplicationLocked()来处理上一步发出的 ATTACH_APPLICATION_TRANSACTION
进程间通信请求。这个方法我们应该很熟悉，我们以前在分析新进程启动 Activity 组件时就走到了这个函数，我们再来看一看它的实现。

#### 12 ActivityManagerService.attachApplicationLocked(IApplicationThread thread, int pid)

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

 private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

        // Find the application record that is being attached...  either via
        // the pid if we are running in multiple processes, or just pull the
        // next app record if we are emulating process with anonymous threads.
        ProcessRecord app;
        //pid指向的前面新创建的应用进程的PID，在第8步中，ActivityManagerService以这个PID
        //为key将一个ProcessRecord存在了mPidsSelfLocked，现在把它取出来保存在变量app中。
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                app = mPidsSelfLocked.get(pid);
            }
        } else if (mStartingProcesses.size() > 0) {
            app = mStartingProcesses.remove(0);
            app.setPid(pid);
        } else {
            app = null;
        }

        ...

        //指向上一步传递过来的ApplicationThread对象，ActivityManagerService以后就可以通过
        //这个ApplicationThread对象同新创建的应用进程通信了。
        app.thread = thread;
        ...

        //处理Service组件
        // Find any services that should be running in this process...
        if (!badApp && mPendingServices.size() > 0) {
            ServiceRecord sr = null;
            try {
                for (int i=0; i<mPendingServices.size(); i++) {
                    sr = mPendingServices.get(i);
                    //检查保存在mPendingServices里的Service组件是否需要在新进程中启动
                    if (app.info.uid != sr.appInfo.uid
                            || !processName.equals(sr.processName)) {
                        continue;
                    }
                    //如果需要在新进程中启动，则将其在mPendingServices移除，并调用
                    //realStartServiceLocked启动该Service组件。
                    mPendingServices.remove(i);
                    i--;
                    realStartServiceLocked(sr, app);
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.w(TAG, "Exception in new application when starting service "
                      + sr.shortName, e);
                badApp = true;
            }
        }
        ...
    }
}
```

注：Activity 组件、Service 组件与 BrocastReceiver 组件启动都是由这个函数来处理的。

该函数会检查保存在 mPendingServices 里的 Service 组件是否需要在新进程中启动，如果果需要在新进程中启动，则将其
在 mPendingServices 移除，并调用 realStartServiceLocked 启动该 Service 组件。

#### 13 ActivityManagerService.realStartServiceLocked(ServiceRecord r, ProcessRecord app)

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

     private final void realStartServiceLocked(ServiceRecord r,
                ProcessRecord app) throws RemoteException {
            ...

            //将ProcessRecord对象app设置为ServiceRecord的成员变量app
            r.app = app;
            ...
            try {
                ...
                app.thread.scheduleCreateService(r, r.serviceInfo);
                ...
            } finally {
                if (!created) {
                    app.services.remove(r);
                    scheduleServiceRestartLocked(r, false);
                }
            }
            ...
}
```

该函数进一步调用 ApplicationThreadProxy.scheduleCreateService()方法执行 Service 组件启动操作。

#### 14 ActivityManagerService.scheduleCreateService(IBinder token, ServiceInfo info)

```java
class ApplicationThreadProxy implements IApplicationThread {

    public final void scheduleCreateService(IBinder token, ServiceInfo info)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        info.writeToParcel(data, 0);
        mRemote.transact(SCHEDULE_CREATE_SERVICE_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }

}
```

该函数调用 ApplicationThreadProxy 内部的一个 Binder 对象向新创建的进程发送一个 SCHEDULE_CREATE_SERVICE_TRANSACTION
进程间通信请求，进一步在新进程中创建 Service 组件。

#### 15 ActivityThread.cheduleCreateService(IBinder token, ServiceInfo info)

```java
public final class ActivityThread {

    public final void scheduleCreateService(IBinder token,
            ServiceInfo info) {
        CreateServiceData s = new CreateServiceData();
        s.token = token;
        s.info = info;

        queueOrSendMessage(H.CREATE_SERVICE, s);
    }

}

```

该方法将要启动的 Service 组件信息封装成一个 CreateServiceData 对象，然后传递给 queueOrSendMessage 方法。

#### 16 ActivityThread.queueOrSendMessage(int what, Object obj, int arg1, int arg2

```java
public final class ActivityThread {

     private final void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {
            synchronized (this) {
                if (DEBUG_MESSAGES) Slog.v(
                    TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                    + ": " + arg1 + " / " + obj);
                Message msg = Message.obtain();
                msg.what = what;
                msg.obj = obj;
                msg.arg1 = arg1;
                msg.arg2 = arg2;
                mH.sendMessage(msg);
            }
        }
}
```

该方法发送一个 CREATE_SERVICE 消息来进一步创建 Service 组件。然后调用 ActivityThread 内部的 Handler 对象来处理消息。

#### 17 H.handleMessage(Message msg)

```java
private final class H extends Handler {
public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + msg.what);
            switch (msg.what) {
                ...
                case CREATE_SERVICE:
                    handleCreateService((CreateServiceData)msg.obj);
                    break;
                ...
              }

}
```

该方法进一步调用 handleCreateService()来创建 Service 组件。

#### 18 ActivityThread.handleCreateService(CreateServiceData data)

```java
public final class ActivityThread {
    private final void handleCreateService(CreateServiceData data) {
            // If we are getting ready to gc after going to the background, well
            // we are back active so skip it.
            unscheduleGcIdler();

            //获取一个用来描述即将要启动Service组件的所在应用的LoadedApk对象。并将它
            //保存在变量packageInfo中。
            LoadedApk packageInfo = getPackageInfoNoCheck(
                    data.info.applicationInfo);
            Service service = null;
            try {
                //获取类加载器，将Service组件加载到内存中，并创建它的一个实例。
                java.lang.ClassLoader cl = packageInfo.getClassLoader();
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

                //创建一个Context对象，作为Service组件运行的上下文环境。
                ContextImpl context = new ContextImpl();
                context.init(packageInfo, null, this);

                //创建一个Application对象，用来描述Service组件所属的应用。
                Application app = packageInfo.makeApplication(false, mInstrumentation);
                context.setOuterContext(service);
                //初始化Service组件
                service.attach(context, this, data.info.name, data.token, app,
                        ActivityManagerNative.getDefault());
                //调用Service的onCreate()方法
                service.onCreate();
                //将新创建的Service组件保存到ActivityThread的成员变量mServices中，其中data.token
                //指向的是ActivityManagerService内部的一个ServiceRecord对象。
                mServices.put(data.token, service);
                try {
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            data.token, 0, 0, 0);
                } catch (RemoteException e) {
                    // nothing to do.
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

该方法真正执行了 Service 组件的创建以及初始化工作，它主要做了以下几件事情：

```
1 获取初始化Service组件所必需的参数：

LoadedApk packageInfo：用来描述已经加载到进程中的应用，通过它可以访问到该应用里的资源。
ContextImpl context：创建一个Context对象，作为Service组件运行的上下文环境。
Application app：创建一个Application对象，用来描述Service组件所属的应用。

2 获取类加载器，将Service组件加载到内存中，并创建它的一个实例。
3 根据上面创建的参数进行Service组件的初始化。
4 调用Service的onCreate()方法。

```

#### 19 Service.onCreate()

这个 onCreate()方法就是我们使用 Service 组件所重写的方法了，用来自定义一些我需要的初始化操作。
