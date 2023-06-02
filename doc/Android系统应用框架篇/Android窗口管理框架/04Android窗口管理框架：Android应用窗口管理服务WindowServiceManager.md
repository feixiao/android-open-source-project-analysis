## Android 窗口管理框架：Android 应用窗口的管理服务 WindowServiceManager

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

**文章目录**

- 一 Window 的添加流程
- 二 Window 的删除流程
- 三 Window 的更新流程

WindowManagerService 是位于 Framework 层的窗口管理服务，它的职责是管理系统中的所有窗口，也就是 Window，关于 Window 的介绍，我们在文章[03Android 显示框架：Android 应用视图的管理者 Window](./doc/Android系统应用框架篇/Android显示框架/03Android显示框架：Android应用视图管理者Window.md)已经
详细分析过，通俗来说，Window 就是手机上一块显示区域，也就是 Android 中的绘制画布 Surface，添加一个 Window 的过程，也就是申请分配一块 Surface 的过程。而整个流程的管理者正是 WindowManagerService。

Window 在 WindowManagerService 的管理下，有序的显示在屏幕上，构成了多姿多彩的用户界面，关于 Android 的整个窗口系统，可以用下图来表示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/window_mansger_service_class.png"/>

Android 窗口管理框架从窗口的创建到 UI 的绘制主要涉及以下角色：

- WindowManager：应用与窗口管理服务 WindowManagerService 交互的接口
- WindowManagerService：窗口管理服务，继承于 IWindowManager.Stub，是 Binder 的服务端，该服务运行在一个单独的进程中，因此 WindowManager 与 WindowManagerService 的交互也是一个 IPC 的过程。
- SurfaceFlinger：SurfaceFlinger 服务运行在 Android 系统的 System 进程中，它负责管理 Android 系统的帧缓冲区（Frame Buffer)，Android 设备的显示屏被抽象为一个
  帧缓冲区，而 Android 系统中的 SurfaceFlinger 服务就是通过向这个帧缓冲区写入内容来绘制应用程序的用户界面的。
- Surface：每个显示界面的窗口都是一个 Surface。
- PhoneWindowManager：实现了窗口的各种策略。
- Choreographer：用于控制窗口动画、屏幕旋转等操作。
- DisplayContent：用于描述多屏输出相关信息。
- WindowState：描述窗口的状态信息以及和 WindowManagerService 进行通信，一般一个窗口对应一个 WindowState。
- WindowToken：窗口 Token，用来做 Binder 通信。
- Session：通信对象，App 进程通过建立 Session 代理对象和 Session 对象通信，进而和 WindowManagerService 建立连接。

前面说到窗口的管理服务 WindowManagerService 运行在 System Server 进程，所以应用进程与 WindowManagerService 的交互是一个 IPC 的过程，具体流程如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/window_mansger_service_structure.png"/>

如上图所示，Activity 持有一个 Window，负责 UI 的展示与用户交互，里保存了很多重要的信息，如下所示：

- mWindow：PhoneWindow 对象，继承于 Window，是窗口对象。
- mWindowManager：WindowManagerImpl 对象，实现 WindowManager 接口。
- mMainThread：Activity 对象，并非真正的线程，是运行在主线程里的对象。
- mUIThread：Thread 对象，主线程。
- mHandler：Handler 对象，主线程 Handler。
- mDecor：View 对象，用来显示 Activity 里的视图。

ViewRootImpl 负责管理 DecorView 与 WindowManagerService 的交互，每次调用 WindowManager.addView()添加窗口时，都会创建一个 ViewRootImpl 对象，它内部也保存了一些重要信息，如下所示：

- mWindowSession：IWindowSession 对象，Session 的代理对象，用来和 Session 进行通信，同一进程里的所有 ViewRootImpl 对象只对应同一个 Session 代理对象。
- mWindow：IWindow.Stubd 对象，每个窗口对应一个该对象。

👉 注：每个 Activity 对应一个 Window，每个 Window 对应一个 ViewRootImpl。

理解了以上内容，我们来看看 Window 的添加、更新和移除流程。

窗口的操作被定义 WindowManager 中，WindowManager 是一个接口，继承于 ViewManager，实现类是 WindowManagerImpl，实际上我们常用的功能，也是定义在 ViewManager 里的。

```java
public interface ViewManager{
    //添加View
    public void addView(View view, ViewGroup.LayoutParams params);
    //更新View
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    //删除View
    public void removeView(View view);
}
```

WindowManager 可以通过 Context 来获取，WindowManager 也会和其他服务一样在开机时注册到 ContextImpl 里的 map 容器里，然后通过他们的 key 来获取。

```java
windowManager = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
```

WindowManager 的实现类是 WindowManagerImpl，在 WindowManagerImpl 内部实际的功能是有 WindowManagerGlobal 来完成的，我们直接来分析它里面这三个方法的实现。

## 一 Window 的添加流程

```java
public final class WindowManagerGlobal {

     public void addView(View view, ViewGroup.LayoutParams params,
                Display display, Window parentWindow) {
            //校验参数的合法性
            ...

            //ViewRootImpl封装了View与WindowManager的交互力促
            ViewRootImpl root;
            View panelParentView = null;

            synchronized (mLock) {
                // Start watching for system property changes.
                if (mSystemPropertyUpdater == null) {
                    mSystemPropertyUpdater = new Runnable() {
                        @Override public void run() {
                            synchronized (mLock) {
                                for (int i = mRoots.size() - 1; i >= 0; --i) {
                                    mRoots.get(i).loadSystemProperties();
                                }
                            }
                        }
                    };
                    SystemProperties.addChangeCallback(mSystemPropertyUpdater);
                }

                int index = findViewLocked(view, false);
                if (index >= 0) {
                    if (mDyingViews.contains(view)) {
                        // Don't wait for MSG_DIE to make it's way through root's queue.
                        mRoots.get(index).doDie();
                    } else {
                        throw new IllegalStateException("View " + view
                                + " has already been added to the window manager.");
                    }
                    // The previous removeView() had not completed executing. Now it has.
                }

                // If this is a panel window, then find the window it is being
                // attached to for future reference.
                if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                        wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                    final int count = mViews.size();
                    for (int i = 0; i < count; i++) {
                        if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                            panelParentView = mViews.get(i);
                        }
                    }
                }

                //通过上下文构建ViewRootImpl
                root = new ViewRootImpl(view.getContext(), display);

                view.setLayoutParams(wparams);

                //mViews存储着所有Window对应的View对象
                mViews.add(view);
                //mRoots存储着所有Window对应的ViewRootImpl对象
                mRoots.add(root);
                //mParams存储着所有Window对应的WindowManager.LayoutParams对象
                mParams.add(wparams);
            }

            // do this last because it fires off messages to start doing things
            try {
                //调用ViewRootImpl.setView()方法完成Window的添加并更新界面
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                synchronized (mLock) {
                    final int index = findViewLocked(view, false);
                    if (index >= 0) {
                        removeViewLocked(index, true);
                    }
                }
                throw e;
            }
        }

}
```

在这个方法里有三个重要的成员变量：

- mViews 存储着所有 Window 对应的 View 对象
- mRoots 存储着所有 Window 对应的 ViewRootImpl 对象
- mParams 存储着所有 Window 对应的 WindowManager.LayoutParams 对象

这里面提到了一个我们不是很熟悉的类 ViewRootImpl，它其实就是一个封装类，封装了 View 与 WindowManager 的交互方式，它是 View 与 WindowManagerService 通信的桥梁。
最后也是调用 ViewRootImpl.setView()方法完成 Window 的添加并更新界面。

我们来看看这个方法的实现。

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.HardwareDrawCallbacks {

     public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
            synchronized (this) {
                if (mView == null) {
                    mView = view;

                    //参数校验与预处理
                    ...

                    // Schedule the first layout -before- adding to the window
                    // manager, to make sure we do the relayout before receiving
                    // any other events from the system.

                    //1. 调用requestLayout()完成界面异步绘制的请求
                    requestLayout();
                    if ((mWindowAttributes.inputFeatures
                            & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                        mInputChannel = new InputChannel();
                    }
                    mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                            & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
                    try {
                        mOrigWindowType = mWindowAttributes.type;
                        mAttachInfo.mRecomputeGlobalAttributes = true;
                        collectViewAttributes();
                        //2. 创建WindowSession并通过WindowSession请求WindowManagerService来完成Window添加的过程
                        //这是一个IPC的过程。
                        res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                                getHostVisibility(), mDisplay.getDisplayId(),
                                mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                                mAttachInfo.mOutsets, mInputChannel);
                    } catch (RemoteException e) {
                        mAdded = false;
                        mView = null;
                        mAttachInfo.mRootView = null;
                        mInputChannel = null;
                        mFallbackEventHandler.setView(null);
                        unscheduleTraversals();
                        setAccessibilityFocus(null, null);
                        throw new RuntimeException("Adding window failed", e);
                    } finally {
                        if (restore) {
                            attrs.restore();
                        }
                    }
                    ...
            }
        }
}
```

这个方法主要做了两件事：

1. 调用 requestLayout()完成界面异步绘制的请求, requestLayout()会去调用 scheduleTraversals()来完成 View 的绘制，scheduleTraversals()方法将一个 TraversalRunnable 提交到工作队列中执行 View 的绘制。而
   TraversalRunnable 最终调用了 performTraversals()方法来完成实际的绘制操作。提到 performTraversals()方法我们已经很熟悉了，在文章[02Android 显示框架：Android 应用视图的载体 View](./doc/Android系统应用框架篇/Android显示框架/02Android显示框架：Android应用视图载体View.md)中
   我们已经详细的分析过它的实现。
2. 创建 WindowSession 并通过 WindowSession 请求 WindowManagerService 来完成 Window 添加的过程这是一个 IPC 的过程，WindowManagerService 作为实际的窗口管理者，窗口的创建、删除和更新都是由它来完成的，它同时还负责了窗口的层叠排序和大小计算
   等工作。

注：在文章[02Android 显示框架：Android 应用视图的载体 View](./doc/Android系统应用框架篇/Android显示框架/02Android显示框架：Android应用视图载体View.md)中
我们已经详细的分析过 performTraversals()方法的实现，这里我们再简单提一下：

1. 获取 Surface 对象，用于图形绘制。
2. 调用 performMeasure()方法测量视图树各个 View 的大小。
3. 调用 performLayout()方法计算视图树各个 View 的位置，进行布局。
4. 调用 performMeasure()方法对视图树的各个 View 进行绘制。

既然提到 WindowManager 与 WindowManagerService 的跨进程通信，我们再讲一下它们的通信流程。Android 的各种服务都是基于 C/S 结构来设计的，系统层提供服务，应用层使用服务。WindowManager 也是一样，它与
WindowManagerService 的通信是通过 WindowSession 来完成的。

1. 首先调用 ServiceManager.getService("window")获取 WindowManagerService，该方法返回的是 IBinder 对象，然后调用 IWindowManager.Stub.asInterface()方法将 WindowManagerService 转换为一个 IWindowManager 对象。
2. 然后调用 openSession()方法与 WindowManagerService 建立一个通信会话，方便后续的跨进程通信。这个通信会话就是后面我们用到的 WindowSession。

基本上所有的 Android 系统服务都是基于这种方式实现的，它是一种基于 AIDL 实现的 IPC 的过程。关于 AIDL 读者可自行查阅资料。

```java
public final class WindowManagerGlobal {

    public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
                    InputMethodManager imm = InputMethodManager.getInstance();
                    //获取WindowManagerService对象，并将它转换为IWindowManager类型
                    IWindowManager windowManager = getWindowManagerService();
                    //调用openSession()方法与WindowManagerService建立一个通信会话，方便后续的
                    //跨进程通信。
                    sWindowSession = windowManager.openSession(
                            new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                    ValueAnimator.setDurationScale(scale);
                                }
                            },
                            imm.getClient(), imm.getInputContext());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowSession;
        }
    }

    public static IWindowManager getWindowManagerService() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowManagerService == null) {
                //调用ServiceManager.getService("window")获取WindowManagerService，该方法返回的是IBinder对象
                //，然后调用IWindowManager.Stub.asInterface()方法将WindowManagerService转换为一个IWindowManager对象
                sWindowManagerService = IWindowManager.Stub.asInterface(
                        ServiceManager.getService("window"));
                try {
                    sWindowManagerService = getWindowManagerService();
                    ValueAnimator.setDurationScale(sWindowManagerService.getCurrentAnimatorScale());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowManagerService;
        }
    }
 }
```

## 二 Window 的删除流程

Window 的删除流程也是在 WindowManagerGlobal 里完成的。

```java
public final class WindowManagerGlobal {

   public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            //1. 查找待删除View的索引
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            //2. 调用removeViewLocked()完成View的删除, removeViewLocked()方法
            //继续调用ViewRootImpl.die()方法来完成View的删除。
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }

    private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();

        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }

}
```

我们再来看看 ViewRootImpl.die()方法的实现。

```java
public final class ViewRootImpl implements ViewParent,

        View.AttachInfo.Callbacks, ThreadedRenderer.HardwareDrawCallbacks {
      boolean die(boolean immediate) {
            // Make sure we do execute immediately if we are in the middle of a traversal or the damage
            // done by dispatchDetachedFromWindow will cause havoc on return.

            //根据immediate参数来判断是执行异步删除还是同步删除
            if (immediate && !mIsInTraversal) {
                doDie();
                return false;
            }

            if (!mIsDrawing) {
                destroyHardwareRenderer();
            } else {
                Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
                        "  window=" + this + ", title=" + mWindowAttributes.getTitle());
            }
            //如果是异步删除，则发送一个删除View的消息MSG_DIE就会直接返回
            mHandler.sendEmptyMessage(MSG_DIE);
            return true;
        }

        void doDie() {
            checkThread();
            if (LOCAL_LOGV) Log.v(mTag, "DIE in " + this + " of " + mSurface);
            synchronized (this) {
                if (mRemoved) {
                    return;
                }
                mRemoved = true;
                if (mAdded) {
                    //调用dispatchDetachedFromWindow()完成View的删除
                    dispatchDetachedFromWindow();
                }

                if (mAdded && !mFirst) {
                    destroyHardwareRenderer();

                    if (mView != null) {
                        int viewVisibility = mView.getVisibility();
                        boolean viewVisibilityChanged = mViewVisibility != viewVisibility;
                        if (mWindowAttributesChanged || viewVisibilityChanged) {
                            // If layout params have been changed, first give them
                            // to the window manager to make sure it has the correct
                            // animation info.
                            try {
                                if ((relayoutWindow(mWindowAttributes, viewVisibility, false)
                                        & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
                                    mWindowSession.finishDrawing(mWindow);
                                }
                            } catch (RemoteException e) {
                            }
                        }

                        mSurface.release();
                    }
                }

                mAdded = false;
            }
            //刷新数据，将当前移除View的相关信息从我们上面说过了三个列表：mRoots、mParms和mViews中移除。
            WindowManagerGlobal.getInstance().doRemoveView(this);
        }

        void dispatchDetachedFromWindow() {
                //1. 回调View的dispatchDetachedFromWindow方法，通知该View已从Window中移除
                if (mView != null && mView.mAttachInfo != null) {
                    mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(false);
                    mView.dispatchDetachedFromWindow();
                }

                mAccessibilityInteractionConnectionManager.ensureNoConnection();
                mAccessibilityManager.removeAccessibilityStateChangeListener(
                        mAccessibilityInteractionConnectionManager);
                mAccessibilityManager.removeHighTextContrastStateChangeListener(
                        mHighContrastTextManager);
                removeSendWindowContentChangedCallback();

                destroyHardwareRenderer();

                setAccessibilityFocus(null, null);

                mView.assignParent(null);
                mView = null;
                mAttachInfo.mRootView = null;

                mSurface.release();

                if (mInputQueueCallback != null && mInputQueue != null) {
                    mInputQueueCallback.onInputQueueDestroyed(mInputQueue);
                    mInputQueue.dispose();
                    mInputQueueCallback = null;
                    mInputQueue = null;
                }
                if (mInputEventReceiver != null) {
                    mInputEventReceiver.dispose();
                    mInputEventReceiver = null;
                }

                //调用WindowSession.remove()方法，这同样是一个IPC过程，最终调用的是
                //WindowManagerService.removeWindow()方法来移除Window。
                try {
                    mWindowSession.remove(mWindow);
                } catch (RemoteException e) {
                }

                // Dispose the input channel after removing the window so the Window Manager
                // doesn't interpret the input channel being closed as an abnormal termination.
                if (mInputChannel != null) {
                    mInputChannel.dispose();
                    mInputChannel = null;
                }

                mDisplayManager.unregisterDisplayListener(mDisplayListener);

                unscheduleTraversals();
            }

}
```

我们再来总结一下 Window 的删除流程：

1. 查找待删除 View 的索引
2. 调用 removeViewLocked()完成 View 的删除, removeViewLocked()方法继续调用 ViewRootImpl.die()方法来完成 View 的删除。
3. ViewRootImpl.die()方法根据 immediate 参数来判断是执行异步删除还是同步删除，如果是异步删除则则发送一个删除 View 的消息 MSG_DIE 就会直接返回。
   如果是同步删除，则调用 doDie()方法。
4. doDie()方法调用 dispatchDetachedFromWindow()完成 View 的删除，在该方法里首先回调 View 的 dispatchDetachedFromWindow 方法，通知该 View 已从 Window 中移除，
   然后调用 WindowSession.remove()方法，这同样是一个 IPC 过程，最终调用的是 WindowManagerService.removeWindow()方法来移除 Window。

## 三 Window 的更新流程

```java
public final class WindowManagerGlobal {

    public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;

        //更新View的LayoutParams参数
        view.setLayoutParams(wparams);

        synchronized (mLock) {
            //查找Viewd的索引，更新mParams里的参数
            int index = findViewLocked(view, true);
            ViewRootImpl root = mRoots.get(index);
            mParams.remove(index);
            mParams.add(index, wparams);
            //调用ViewRootImpl.setLayoutParams()完成重新布局的工作。
            root.setLayoutParams(wparams, false);
        }
    }
}
```

我们再来看看 ViewRootImpl.setLayoutParams()方法的实现。

```java
public final class ViewRootImpl implements ViewParent,

   void setLayoutParams(WindowManager.LayoutParams attrs, boolean newView) {
           synchronized (this) {
               //参数预处理
               ...

                //如果是新View，调用requestLayout()进行重新绘制
               if (newView) {
                   mSoftInputMode = attrs.softInputMode;
                   requestLayout();
               }

               // Don't lose the mode we last auto-computed.
               if ((attrs.softInputMode & WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST)
                       == WindowManager.LayoutParams.SOFT_INPUT_ADJUST_UNSPECIFIED) {
                   mWindowAttributes.softInputMode = (mWindowAttributes.softInputMode
                           & ~WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST)
                           | (oldSoftInputMode & WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST);
               }

               mWindowAttributesChanged = true;
               //如果不是新View，调用requestLayout()进行重新绘制
               scheduleTraversals();
           }
       }
}
```

Window 的更新流程也和其他流程相似：

1. 更新 View 的 LayoutParams 参数，查找 Viewd 的索引，更新 mParams 里的参数。
2. 调用 ViewRootImpl.setLayoutParams()方法完成重新布局的工作，在 setLayoutParams()方法里最终会调用 scheduleTraversals()
   进行解码重绘制，scheduleTraversals()后续的流程就是 View 的 measure、layout 和 draw 流程了，这个我们在上面已经说过了。

通过上面分析，我们了解了 Window 的添加、删除和更新流程。那么我们来思考一个问题：既然说是 Window 的添加、删除和更新，那为什么方法名是 addView、updateViewLayout、removeView，Window 的本质是什么？🤔

> 事实上 Window 是一个抽象的概念，也就是说它并不是实际存在的，它以 View 的形式存在，每个 Window 都对应着一个 View 和一个 ViewRootImpl，Window 与 View 通过 ViewRootImpl 来建立联系。推而广之，我们可以理解
> WindowManagerService 实际管理的也不是 Window，而是 View，管理在当前状态下哪个 View 应该在最上层显示，SurfaceFlinger 绘制也同样是 View。

好了本篇文章的内容到这里就讲完了，在这篇文章中我们侧重分析 Android 窗口服务 Client 这一侧的实现，事实上更多的内容是在 Server 这一侧，也就是 WindowManagerService。只不过在日常的开发中我们较少
接触到 WindowManagerService，它属于系统的内部服务，就暂时不做进一步的展开。总体上来说，本系列文章的目的还是在于更好的服务应用层的开发者，等到关于 Android 应用层 Framework 实现原理分析完成以后，
我们再进一步深入，去分析系统层 Framework 的实现。
