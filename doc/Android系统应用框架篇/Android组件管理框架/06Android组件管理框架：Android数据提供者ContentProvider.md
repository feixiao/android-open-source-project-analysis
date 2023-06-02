# Android 组件管理框架：Android 后台服务 Service

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

Content Provider 用于提供数据统一访问的格式，封装底层的具体实现，对于调用者而言，无需失效数据的来源，例如：数据库、文件或者
网络，只需要使用 Content Provider 提供的接口就可以进行数据的增删改查操作。

Content Provider 作为 Android 四大组件之一，没有复杂的生命周期，只有简单的 onCreate()过程。底层基于 Binder 实现，可以用来实现
跨进程数据交互与共享。
