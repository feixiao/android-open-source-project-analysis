# Android 界面开发：View 自定义实践布局篇

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

**文章目录**

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

- [01Android 界面开发：View 自定义实践概览](./doc/Android应用开发实践篇/Android界面开发/01Android界面开发：View自定义实践概览.md)
- [02Android 界面开发：View 自定义实践布局篇](./doc/Android应用开发实践篇/Android界面开发/02Android界面开发：View自定义实践布局篇.md)
- [03Android 界面开发：View 自定义实践绘制篇](./doc/Android应用开发实践篇/Android界面开发/03Android界面开发：View自定义实践绘制篇.md)
- [04Android 界面开发：View 自定义实践交互篇](./doc/Android应用开发实践篇/Android界面开发/04Android界面开发：View自定义实践交互篇.md)

在文章[02Android 显示框架：Android 应用视图的载体 View](./doc/Android系统应用框架篇/Android显示框架/02Android显示框架：Android应用视图载体View.md)中我们理解了
View 的测量、布局、绘制、触摸事件处理等内容，今天我们开始我们 View 自定义实践的内容。

View 自定义是开发中最常见的需求，图表等各种复杂的 ui 以及产品经理各种奇怪的需求 😤 都要通过 View 自定义来完成。

View 自定义有三个关键点：

- 布局：决定 View 的摆放位置
- 绘制：决定 View 的具体内容
- 交互：决定 View 与用户的交互体验

这篇文章我们就来分析关于 View 自定义的布局问题。要想彻底掌握 View 自定义的布局，就要理解 View 的布局实现，这个我们在前面的文章分析过源码，这里再来整体总结一下。
