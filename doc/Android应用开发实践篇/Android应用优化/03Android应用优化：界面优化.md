# Android 应用优化：界面优化

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

## 一 顿检测

我们可以利用 BlockCanary 去检查造成 UI 卡顿的地方，如下所示：

BlockCanary：https://github.com/markzhai/AndroidPerformanceMonitor

BlockCanary 检查 UI 卡顿的原理如下图所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/practice/performance/blockcanary_structure.png"/>

## 二 卡顿优化

Android 界面优化主要解决界面卡顿的问题，Android 系统每隔 16ms 就会发送一个 VSYNC 信号，触发 UI 渲染，如果绘制操作超过了 16ms，就会引起掉帧，也就是会导致姐们卡顿。

导致界面卡顿的原因主要是过度绘制，绘制了多余的 UI，开发者选项里有检测过度绘制的工具，如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/practice/performance/overdraw_level.png" width="250"/>

1. 移除不必要的 backgroud。
2. 自定义 View 的时候 clipReact 减少重叠区域的绘制。
3. 利用<merge>等标签减少 View 的层级。
4. 利用<ViewStub>在需要的时候再去加载 View。
