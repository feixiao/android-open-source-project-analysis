# Android 进程框架：线程通信的桥梁 Handler

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

**文章目录**

- 一 消息队列的创建
  - 1.1 建立消息队列
  - 1.2 开启消息循环
- 二 消息的添加
- 三 消息的分发和处理
  - 3.1 消息分发
  - 3.2 消息处理

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

Android 是一个消息驱动型的系统，消息机制在 Android 系统中扮演者重要的角色，与之相关的 Handler 也是我日常中常用的工具。今天我们就来聊一聊这个。

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

有了上面这些分析，相信我们对 Android 的消息机制有了大致的理解，对于这套机制，我们很自然会去思考三个方面的问题：

- 消息队列是如何创建的，它们如何实现消息循环的，消息循环为什么不会导致线程卡死？🤔
- 消息是如何添加到队列中的，它们在队列里是如何排序的？🤔
- 消息是如何被分发的，分发以后又是如何被处理的？🤔

我们一一来看一下。

## 一 消息队列的创建

### 1.1 建立消息队列

消息队列是由 MessageQueue 类来描述的，MessageQueue 是 Android 消息机制 Java 层和 C++层的纽带，其中很多核心方法都交由 native 方法实现。

既然提到对象构建，我们先来看看它的构造函数。

```java
public final class MessageQueue {

    private long mPtr; // used by native code

    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
}
```

可以看到它调用的是 native 方法来完成初始化，这个方法定义在了一个 android_os_MessageQueue 的 C++类类。

```java
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    //构建NativeMessageQueue对象
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);
    //将nativeMessageQueue对象的地址值转成long型返回该Java层
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

可以看到该方法构建了一个 NativeMessageQueue 对象，并将 NativeMessageQueue 对象的地址值转成 long 型返回给 Java 层，这里我们知道实际上是 mPtr 持有了这个
地址值。

NativeMessageQueue 继承域 MessageQueue.cpp 类，我们来看看 NativeMessageQueue 的构造方法。

```java
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {

    //先检查是否已经为当前线程创建过一个Looper对象
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        //创建Looper对象
        mLooper = new Looper(false);
        //为当前线程设置Looper对象
        Looper::setForThread(mLooper);
    }
}
```

可以看到 NativeMessageQueue 构造方法先检查是否已经为当前线程创建过一个 Looper 对象，如果没有，则创建 Looper 对象并为当前线程设置 Looper 对象。

我们再来看看 Looper 的构造方法。

```java
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    int wakeFds[2];
    //创建管道
    int result = pipe(wakeFds);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not create wake pipe.  errno=%d", errno);
    //读端文件描述符
    mWakeReadPipeFd = wakeFds[0];
    //写端文件描述符
    mWakeWritePipeFd = wakeFds[1];
    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake read pipe non-blocking.  errno=%d",
            errno);
    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake write pipe non-blocking.  errno=%d",
            errno);
    //创建一个epoll实例，并将它的文件描述符保存在变量mEpollFd中
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance.  errno=%d", errno);
    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeReadPipeFd;
    //将前面创建的管道读端描述符添加到这个epoll实例中，以便它可以对管道的写操作进行监听
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake read pipe to epoll instance.  errno=%d",
            errno);
}
```

这里面提到两个概念：管道与 epoll 机制。

关于管道

> 管道在本质上也是文件，但它不是普通的文件，它不属于任何文件类型，而且它只存在与内存之中且有固定大小的缓存区，一般为 1 页即 4kb。它分为读端和写端，读端负责从
> 管道读取数据，当数据为空时则阻塞，写端负责向管道写数据，当管道缓存区满时则阻塞。那管道在线程通信中主要用来通知另一个线程。例如：线程 A 准备好了 Message 放入
> 了消息队列，这个时候需要通知线程 B 去处理，这个时候线程 A 就像管道的写端写入数据，管道有了数据之后就回去唤醒线程 B 区处理消息。也正是基于管道来进行线程的休眠与
> 唤醒，才保住了线程中的 loop 循环不会让线程卡死。

关于 epoll 机制

> epoll 机制用来监听多个文件描述符的 IO 读写事件，在 Android 的消息机制用来监听管道的读写 IO 事件。

关于 epool 机制，这里有个[简单易懂的解释](https://www.zhihu.com/question/20122137)。

epoll 一共有三个操作方法：

- epoll_create()：创建一个 epoll 的句柄，size 是指监听的描述符个数
- epoll_ctl()：对需要监听的文件描述符(fd)执行操作，比如将 fd 加入到 epoll 句柄。
- epoll_wait()：返回需要处理的事件数目，如返回 0 表示已超时。

上面 Looper 的构造方法里，我们已经看到了利用 epoll_create()创建一个 epoll 的实例，并利用 epoll_ctl()将管道的读端描述符操作符添加到 epoll 实例中，以便可以对管道的
写操作进行监听，下面我们还可以看到 epoll_wait()的用法。

讲到这里整个消息队列便创建完成了，下面我们接着来看看消息循环和如何开启的。

### 1.2 开启消息循环

消息循环是建立在 Looper 之上的，Looper 可以为线程添加一个消息循环的功能，具体说来，为了给线程添加一个消息循环，我们通常会这么做：

```java
public class LooperThread extends Thread {

    public Handler mHandler;

    @Override
    public void run() {
        Looper.prepare();
        mHandler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                // process incoming messages here
            }
        };
        Looper.loop();
    }
}
```

可以看到先调用 Looper.prepare()初始化一个 Looper，然后调用 Looper.loop()开启循环。

关于 Looper，有两个方法来初始化 prepare()/prepareMainLooper()，它们创建的 Looper 对象都是一样，只是 prepareMainLooper()
创建的 Looper 是给 Android 主线程用的，它还是个静态对象，以便其他线程都可以获取到它，从而可以向主线程发送消息。

```java
public final class Looper {

   // sThreadLocal.get() will return null unless you've called prepare().
   static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
   private static Looper sMainLooper;  // guarded by Looper.class

    public static void prepare() {
          prepare(true);
      }

     private static void prepare(boolean quitAllowed) {
      if (sThreadLocal.get() != null) {
          throw new RuntimeException("Only one Looper may be created per thread");
      }
      sThreadLocal.set(new Looper(quitAllowed));
     }

      //创建主线程的Looper，应用启动的时候会右系统调用，我们一般不需要调用这个方法。
      public static void prepareMainLooper() {
          prepare(false);
          synchronized (Looper.class) {
              if (sMainLooper != null) {
                  throw new IllegalStateException("The main Looper has already been prepared.");
              }
              sMainLooper = myLooper();
          }
      }

      //返回和当前线程相关的Looper
      public static @Nullable Looper myLooper() {
          return sThreadLocal.get();
      }
}

```

指的一提的是这里使用的是 ThreadLocal 来存储新创建的 Looper 对象。

> ThreadLocal 描述的是线程本地存储区，不同的线程不能访问对方的线程本地存储区，当前线程可以对自己的线程本地存储区进行独立的修改和读取。

之所以会采用 ThreadLocal 来存储 Looper，是因为每个具备消息循环能力的线程都有自己独立的 Looper，它们彼此独立，所以需要用线程本地存储区来存储 Looper。

我们在接着来看看 Looper 的构造函数，如下所示：

```java
public final class Looper {

  private Looper(boolean quitAllowed) {
      //创建消息队列
      mQueue = new MessageQueue(quitAllowed);
      //指向当前线程
      mThread = Thread.currentThread();
   }
}
```

Looper 的构造函数也很简单，构造了一个消息队列 MessageQueue，并将成员变量 mThread 指向当前线程，这里构建了一个 MessageQueue 对象，在 MessageQueue 构建
的过程中会在 C++层构建 Looper 对象，这个我们上面已经说过。

Looper 对象创建完成后就可以开启消息循环了，这是由 loop()方法来完成的。

```java
public final class Looper {

     public static void loop() {
        //获取当前线程的Looper
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }

        //获取当前线程的消息队列
        final MessageQueue queue = me.mQueue;

        //确保当前线程处于本地进程中，Handler仅限于处于同一进程间的不同线程的通信。
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        //进入loop主循环方法
        for (;;) {
            //不断的获取下一条消息，这个方法可能会被阻塞
            Message msg = queue.next();
            if (msg == null) {
                //如果没有消息需要处理，则退出当前循环。
                return;
            }

            // 默认为null，可通过setMessageLogging来指定输出，用于debug
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            //处理消息
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            //把message回收到消息池，以便重复利用。
            msg.recycleUnchecked();
        }
     }
}
```

可以看到，这个方法不断重复着以下三件事：

1. 调用 MessageQueue 的 next()方法读取 MessageQueue 的下一条 Message。
2. 把 Message 分发给相应的 target。
3. 再把分发的 Message 回收到消息池，以便重复利用。

如此消息循环便建立起来了。

## 二 消息的添加

在如此开始中，我们通常会调用 handler 的 sendXXX()或者 postXXX()将一个 Message 或者 Runnable，这些方法实际上调用的 MessageQueue 的 enqueueMessage()方法，该方法
会给目标线程的消息队列插入一条消息。

注：如何理解这个"目标线程的消息队列"，首先要明白 Handler、Looper 与 MessageQueue 这三兄弟是全家桶，绑在一起的，你用哪个 Handler，消息就被插入到了这个 Handler 所在线程
的消息队列里。

```java
public final class MessageQueue {

      boolean enqueueMessage(Message msg, long when) {
            //每个消息都必须有个target handler
            if (msg.target == null) {
                throw new IllegalArgumentException("Message must have a target.");
            }

            //每个消息必须没有被使用
            if (msg.isInUse()) {
                throw new IllegalStateException(msg + " This message is already in use.");
            }

            synchronized (this) {
                //正在退出时，回收Message，加入消息池。
                if (mQuitting) {
                    IllegalStateException e = new IllegalStateException(
                            msg.target + " sending message to a Handler on a dead thread");
                    Log.w(TAG, e.getMessage(), e);
                    msg.recycle();
                    return false;
                }

                msg.markInUse();
                msg.when = when;
                //mMessages表示当前需要处理的消息，也就是消息队列头的消息
                Message p = mMessages;
                boolean needWake;

                if (p == null || when == 0 || when < p.when) {
                    // New head, wake up the event queue if blocked.
                    msg.next = p;
                    mMessages = msg;
                    needWake = mBlocked;
                } else {
                    needWake = mBlocked && p.target == null && msg.isAsynchronous();
                    //将消息按照时间顺序插入到消息队列中
                    Message prev;
                    for (;;) {
                        prev = p;
                        p = p.next;
                        if (p == null || when < p.when) {
                            break;
                        }
                        if (needWake && p.isAsynchronous()) {
                            needWake = false;
                        }
                    }
                    msg.next = p; // invariant: p == prev.next
                    prev.next = msg;
                }

                // 唤醒消息队列
                if (needWake) {
                    nativeWake(mPtr);
                }
            }
            return true;
        }
}
```

enqueueMessage()以时间为序将消息插入到消息队列中去，以下三种情况下需要插入到队列头部：

- 消息队列为空
- 要插入的消息的执行时间为 0
- 要插入的消息的执行时间小于消息队列头的消息的执行时间

上面三种情况很容易想到，其他情况以时间为序插入到队列中间。当有新的消息插入到消息队列头时，当前线程就需要去唤醒目标线程（如果它已经睡眠（mBlocked = true）就执行唤醒操作，否则不需要），以便
它可以来处理新插入消息头的消息。

通过这里的分析，你可以发现，消息队列事实上是基于单向链表来实现的，虽然我们总称呼它为"队列"，但它并不是一个队列（不满足先进先出）。

同样利用单向链表这种思路的还有对象池，读者应该有印象，很多文档都提倡通过 Message.obtain()方法获取一个 Message 对象，这是因为 Message 对象会被缓存在消息池中，它主要利用
Message 的 recycle()/obtain()方法进行缓存和获取。

具体说来：

**recycle()**

```java
public final class Message implements Parcelable {

        private static final Object sPoolSync = new Object();
        private static Message sPool;

        public void recycle() {
            //判断消息是否正在使用
            if (isInUse()) {
                if (gCheckRecycle) {
                    throw new IllegalStateException("This message cannot be recycled because it "
                            + "is still in use.");
                }
                return;
            }
            //对于不再使用的消息加入消息池
            recycleUnchecked();
        }

        void recycleUnchecked() {
            //将消息标记为FLAG_IN_USE并清空关于它的其他信息
            flags = FLAG_IN_USE;
            what = 0;
            arg1 = 0;
            arg2 = 0;
            obj = null;
            replyTo = null;
            sendingUid = -1;
            when = 0;
            target = null;
            callback = null;
            data = null;

            synchronized (sPoolSync) {
                //当消息池没有满时，将消息加入消息池
                if (sPoolSize < MAX_POOL_SIZE) {
                    //将sPool存放在next变量中
                    next = sPool;
                    //sPool引用当前对象
                    sPool = this;
                    //消息池数量自增1
                    sPoolSize++;
                }
            }
        }
}
```

**obtain()**

```java
public final class Message implements Parcelable {

    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {

                //sPool当前持有的消息对象将作为结果返回
                Message m = sPool;
                //将m的后继重新赋值给sPool，这其实一个链表的删除操作
                sPool = m.next;
                //m的后继置为空
                m.next = null;
                //清除 in-use 标志位
                m.flags = 0;
                //消息池大小自减1
                sPoolSize--;
                return m;
            }
        }
        //当对象池为空时，直接创建新的Message()对象。
        return new Message();
    }
}
```

这里面有个巧妙的设计，这也给我们如何设计一个对象池提供了一个很好的思路，它是以单向链表具体说来：

1. 在类中定义一个该类的静态对象 sPool 以及它的后继对象 next。
2. 当对象加入对象池时，将该对象加入到链表中。
3. 当对象从对象池中取出时，返回 sPool 当前持有的对象即可，并将 sPool 从当前链表中移除。

好了。消息池就聊这么多，我们接着来看消息的分发和处理。

## 三 消息的分发与处理

### 3.1 消息分发

消息的分发是建立在消息循环之上的，在不断的循环中拉取队列里的消息，消息循环的建立流程我们上面已经分析过，通过分析得知，loop()方法
不断调用 MessageQueue 的 next()读取消息队列里的消息，从而进行消息的分发。

我们来看看 next()方法的实现。

```java
public final class MessageQueue {

    Message next() {
           final long ptr = mPtr;
           //当前消息循环已退出，直接返回。
           if (ptr == 0) {
               return null;
           }

           //pendingIdleHandlerCount保存的是注册到消息队列中空闲Handler个个数
           int pendingIdleHandlerCount = -1;
           //nextPollTimeoutMillisb表示当前无消息到来时，当前线程需要进入睡眠状态的
           //时间，0表示不进入睡眠状态，-1表示进入无限等待的睡眠状态，直到有人将它唤醒
           int nextPollTimeoutMillis = 0;
           for (;;) {
               if (nextPollTimeoutMillis != 0) {
                   Binder.flushPendingCommands();
               }

               //nativePollOnce是阻塞操作，用来检测当前线程的消息队列中是否有消息需要处理
               nativePollOnce(ptr, nextPollTimeoutMillis);

               //查询下一条需要执行的消息
               synchronized (this) {
                   final long now = SystemClock.uptimeMillis();
                   Message prevMsg = null;
                   //mMessages代表了当前线程需要处理的消息
                   Message msg = mMessages;

                   //查询第一个可以处理的消息（msg.target == null表示没有处理Handler，无法进行处理，忽略掉）
                   if (msg != null && msg.target == null) {
                       do {
                           prevMsg = msg;
                           msg = msg.next;
                       } while (msg != null && !msg.isAsynchronous());
                   }
                   if (msg != null) {
                       //如果消息的执行时间大于当前时间，则计算线程需要睡眠等待的时间
                       if (now < msg.when) {
                           //当异步消息的触发时间大于当前时间，则使者下一次轮询的超时时长
                           nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                       }
                       //如果消息的执行时间小于当前时间，则说明该消息需要立即执行，则将该消息返回，并从消息队列中
                       //将该消息移除
                       else {
                           mBlocked = false;
                           if (prevMsg != null) {
                               prevMsg.next = msg.next;
                           } else {
                               mMessages = msg.next;
                           }
                           msg.next = null;
                           if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                           //设置消息的使用状态，即flags |= FLAG_IN_USE。
                           msg.markInUse();
                           return msg;
                       }
                   } else {
                       // 如果没有更多消息需要处理，则将nextPollTimeoutMillis置为-1，让当前线程进入无限睡眠状态，直到
                       //被其他线程唤醒。
                       nextPollTimeoutMillis = -1;
                   }

                   //所有消息都已经被处理，准备退出。
                   if (mQuitting) {
                       dispose();
                       return null;
                   }

                   //pendingIdleHandlerCount指的是等待执行的Handler的数量，mIdleHandlers是一个空闲Handler列表
                   if (pendingIdleHandlerCount < 0
                           && (mMessages == null || now < mMessages.when)) {
                       pendingIdleHandlerCount = mIdleHandlers.size();
                   }
                   if (pendingIdleHandlerCount <= 0) {
                       //当没有空闲的Handler需要执行时进入阻塞状态，mBlocked设置为true
                       mBlocked = true;
                       continue;
                   }

                   //mPendingIdleHandler是一个IdleHandler数组
                   if (mPendingIdleHandlers == null) {
                       mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                   }
                   mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
               }

               //只有在第一次循环时，才会去执行idle handlers，执行完成后重置pendingIdleHandlerCount为0
               for (int i = 0; i < pendingIdleHandlerCount; i++) {
                   final IdleHandler idler = mPendingIdleHandlers[i];
                   //释放Handler的引用
                   mPendingIdleHandlers[i] = null;

                   boolean keep = false;
                   try {
                       keep = idler.queueIdle();
                   } catch (Throwable t) {
                       Log.wtf(TAG, "IdleHandler threw exception", t);
                   }

                   if (!keep) {
                       synchronized (this) {
                           mIdleHandlers.remove(idler);
                       }
                   }
               }

               //执行完成后，重置pendingIdleHandlerCount为0，以保证不会再次重复运行。
               pendingIdleHandlerCount = 0;

               //当调用一个空闲Handler时，新的消息可以立即被分发，因此无需再设置超时时间。
               nextPollTimeoutMillis = 0;
           }
       }
}
```

**首先要明确一个概念，MessageQueue 是利用对象间的后继关联（每个对象都知道自己的直接后继）实现的链表，其中它的成员变量 mMessages 变量指的是当前需要被处理消息。**

next()方法主要用来从消息队列里循环获取消息，这分为两步：

1. 调用 nativePollOnce(ptr, nextPollTimeoutMillis)方法检测当前线程的消息队列中是否有消息需要处理（注意不是在这里取消息）。它是一个阻塞操作，可能会引起
   线程睡眠，下面我们会说。
2. 查询当前需要处理的消息，返回并将其从消息队列中移除。

这个查询当前需要处理的消息可以分为三步：

1. 找到当前的消息队列头 mMessages，如果它为空就说明整个消息队列为空，就将 nextPollTimeoutMillis 置为-1，当前线程进入无限睡眠等待，知道别的线程将其唤醒。如果
   它不为空，则进入步骤 2.
2. 如果消息队列头的执行之间大于当前时间，则说明线程需要等待该消息的执行，线程进入睡眠。否则进入步骤 3.
3. 查找到了当前需要被处理的消息，将该消息从消息队列里移除，并返回该消息。

可以看到这里调用的是 native 方法 nativePollOnce()来检查当前线程是否有消息需要处理，调用该方法时，线程有可能进入睡眠状态，具体由 nextPollTimeoutMillis 参数决定。0 表示不进入睡眠状态，-1 表示
进入无限等待的睡眠状态，直到有人将它唤醒。

我们接着来看看 nativePollOnce()方法的实现。

nativePollOnce()方法是个 native 方法，它按照调用链：android_os_MessageQueue#nativePollOnce() -> NativeMessageQueue#pollOnce() -> Looper#pollOnce() -> Looper#pollInner()
最终完成了消息的拉取。可见实现功能的还是在 Looper.cpp 里。

我们来看一下实现。

```java
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        ...
        //内部不断调用pollInner()方法检查是否有新消息需要处理
        result = pollInner(timeoutMillis);
    }
}

int Looper::pollInner(int timeoutMillis) {
    ...
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];

    //调用epoll_wait()函数监听前面注册在mEpollFd实例里的管道文件描述符中的读写事件。如果这些管道
    //文件描述符没有发生读写事件，则当前线程会在epoll_wait()方法里进入睡眠，睡眠事件由timeoutMillis决定
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    ...

    //epoll_wait返回后，检查哪一个管道文件描述符发生了读写事件
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        //如果fd是当前线程关联的管道读端描述符且读写事件类型是EPOLLIN
        //就说明当前线程关联的一个管道写入了新的数据，这个时候就会调用
        //awoken()去唤醒线程
        if (fd == mWakeReadPipeFd) {
            if (epollEvents & EPOLLIN) {
                //此时已经唤醒线程，读取清空管道数据
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
            }
        }
        ...
     }
     ...
    return result;
}
```

可以看到这个方法做了两件事情：

1. 调用 epoll_wait()函数监听前面注册在 mEpollFd 实例里的管道文件描述符中的读写事件。如果这些管道文件描述符没有发生读写事件，则当前线程
   会在 epoll_wait()方法里进入睡眠，睡眠事件由 timeoutMillis 决定。
2. 如果 fd 是当前线程关联的管道读端描述符且读写事件类型是 EPOLLIN 就说明当前线程关联的一个管道写入了新的数据，这个时候就会调用 awoken()去唤醒线程。

至此，消息完成了分发。从上面的 loop()方法我们可以知道，消息完成分发后会接着调用 Handler 的 dispatchMessage()方法来处理消息。

我们接着来聊一聊 Handler。

### 3.1 消息处理

Handler 主要用来发送和处理消息，它会和自己的 Thread 以及 MessageQueue 相关联，当创建一个 Hanlder 时，它就会被绑定到创建它的线程上，它会向
这个线程的消息队列分发 Message 和 Runnable。

一般说来，Handler 主要有两个用途：

- 调度 Message 和 Runnable，延时执行任务。
- 进行线程的切换，请求别的线程完成相关操作。

我们先来看看 Handler 的构造函数。

```java
public class Handler {

    //无参构造方法，最常用。
    public Handler() {
        this(null, false);
    }

    public Handler(Callback callback) {
        this(callback, false);
    }

    public Handler(Looper looper) {
        this(looper, null, false);
    }

    public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);
    }

    public Handler(Callback callback, boolean async) {
        //匿名类、本地类都必须声明为static，否则会警告可能出现内存泄漏，这个提示我们应该很熟悉了。
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        //获取当前线程的Looper
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        //获取当前线程的消息队列
        mQueue = mLooper.mQueue;
        //回调方法，这个Callback里面其实只有个handleMessage()方法，我们实现这个
        //接口，就不用去用匿名内部类那样的方式来创建Handler了。
        mCallback = callback;
        //设置消息是否为异步处理方式
        mAsynchronous = async;
    }
}
```

对于构造方法而言，我们最常用的是无参构造方法，它没有 Callback 回调，且消息处理方式为同步处理，从这里我们也可以看出你在哪个线程里创建了 Handler，就默认使用当前线程的 Looper。

从上面的 loop()方法中，我们知道 Looper 会调用 MessageQueue 的 dispatchMessage()方法进行消息的处理，我们来看看这个方法的实现。

```java
public class Handler {
        public void dispatchMessage(Message msg) {
            //当Message存在回调方法时，优先调用Message的回调方法message.callback.run()
            if (msg.callback != null) {
                //实际调用的就是message.callback.run();
                handleCallback(msg);
            } else {
                //如果我们设置了Callback回调，优先调用Callback回调。
                if (mCallback != null) {
                    if (mCallback.handleMessage(msg)) {
                        return;
                    }
                }
                //如果我们没有设置了Callback回调，则回调自身的Callback方法。
                handleMessage(msg);
            }
        }
}
```

可以看到整个消息分发流程如下所示：

1. 当 Message 存在回调方法时，优先调用 Message 的回调方法 message.callback.run()。
2. 果我们设置了 Callback 回调，优先调用 Callback 回调。
3. 如果我们没有设置了 Callback 回调，则回调自身的 Callback 方法。

由此我们也可以得知方法调用的优先级，从高到低依次为：

- message.callback.run()
- Handler.mCallback.handleMessage()
- Handler.handleMessage()

大部分代码都是以匿名内部类的形式实现了 Handler，所以一般会走到第三个流程。

可以看到所以发送消息的方法最终都是调用 MessageQueue 的 enqueueMessage()方法来实现，这个我们上面在分析 MessageQueue 的时候已经说过，这里就不再赘述。

理解了上面的内容，相信读者已经对 Android 的消息机制有了大致的了解，我们趁热打铁来聊一聊实际业务开发中遇到的一些问题。

> 在日常的开发中，我们通常在子线程中执行耗时任务，主线程更新 UI，更新的手段也多种多样，如 Activity#runOnUIThread()、View#post()等等，它们之间有何区别呢？如果我的代码了
> 既没有 Activity 也没有 View，我该如何将代码切换回主线程呢？🤔

我们一一来分析。

首先，Activity 里的 Handler 直接调用的就是默认的无参构造方法。可以看到在上面的构造方法里调用 Looper.myLooper()去获取当前线程的 Looper，对于 Activity 而言当前线程就是主线程（UI 线程），那主线程
的 Looper 是什么时候创建的呢？🤔

在[03Android 组件框架：Android 视图容器 Activity](./doc/Android系统应用框架篇/Android组件框架/03Android组件框架：Android视图容器Activity.md)一文
里我们就分析过，ActivityThread 的 main()函数作为应用的入口，会去初始化 Looper，并开启消息循环。

```java
public final class ActivityThread {
      public static void main(String[] args) {
          ...
          Looper.prepareMainLooper();
          ...
          if (sMainThreadHandler == null) {
              sMainThreadHandler = thread.getHandler();
          }
          ...
          Looper.loop();
          throw new RuntimeException("Main thread loop unexpectedly exited");
      }
}
```

主线程的 Looper 已经准备就绪了，我们再调用 Handler 的构造函数去构建 Handler 对象时就会默认使用这个 Handler，如下所示：

```java
public class Activity {

   final Handler mHandler = new Handler();

   public final void runOnUiThread(Runnable action) {
          if (Thread.currentThread() != mUiThread) {
              mHandler.post(action);
          } else {
              action.run();
          }
      }
}
```

```java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {

    public boolean post(Runnable action) {

        //当View被添加到window时会添加一些附加信息，这里面就包括Handler
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }

        //Handler等相关信息还没有被关联到Activity，先建立一个排队队列。
        //这其实就相当于你去银行办事，银行没开门，你们在门口排队等着一样。
        getRunQueue().post(action);
        return true;
    }
}
```

这里面也是利用 attachInfo.mHandler 来处理消息，它事实上是一个 Handler 的子类 ViewRootHandler，同样的它也是使用 Looper.prepareMainLooper()构建出来的 Looper。

所以你可以看出 Activity#runOnUIThread()、View#post()这两种方式并没有本质上的区别，最终还都是通过 Handler 来发送消息。那么对于那些既不在 Activity 里、也不在 View 里的代码
当我们想向主线程发送消息或者将某段代码（通常都是接口的回调方法，在这些方法里需要更新 UI）post 到主线程中执行，就可以按照以下方式进行：

```java
Handler handler = new Handler(Looper.getMainLooper());
handler.post(new Runnable() {
    @Override
    public void run() {
        //TODO refresh ui
    }
})
```

好了，到这里整个 Android 的消息机制我们都已经分析完了，如果对底层的管道这些东西感觉比较模糊，可以先理解 Java 层的实现。
