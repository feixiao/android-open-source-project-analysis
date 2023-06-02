# Android 组件管理框架：Android 广播接收者 BroadcastReceiver

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

**文章目录**

- 一 广播的注册流程
- 二 广播的发送流程

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

Android 里的广播机制用来做进程或者线程通信，它基于 Binder 实现，使用广播的过程分为发送广播和接收广播两个过程，BroadcastReceiver 作为四大组件之一，就是用来接收广播的。

发送的广播可以分为三种，如下所示：

- 普通广播：通过 Context 的 sendBroastcast()发送，可以并行处理。
- 有序广播：通过 Context 的 sendOrderedBroastcast()发送，可以串行处理。
- 粘性广播：通过 Context 的 sendStickyBroastcast()发送，粘性广播发出后会一直等待对应的 Receiver 处理，如果对应的 Receiver 被销毁，下次重建的时候会自动接收到广播消息。Android 5.0 以后处于安全性
  考虑已经废除了这个广播。

广播接收者可以分为两种，如下所示：

- 静态广播接收者：在 AndroidManifest.xml 文件里用标签注册 BroadcastReceiver。
- 动态广播接收者：在代理里通过 Context 的 registerReceiver()注册广播，不再需要的使用通过 unregisterReceiver()解除注册。
