## Android 界面开发：View 自定义实践概览

**文章目录**

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

**View 自定义通常有哪些手段？🤔**

- 继承 View 重写 onDraw()方法，这种方式通常用来实现一些特殊的绘制效果。
- 继承 ViewGroup 实现一些特殊的 Layout，这种方式通常用来实现一些系统之外的特殊的布局效果。
- 继承特定的 View，例如 ImageView、TextView，这种方式通常用于功能的扩展。

View 自定义通常需要处理哪些问题？🤔

- 让 View 支持 wrap_content 以及 padding，这个问题文章[02Android 显示框架：Android 应用视图的载体 View](./doc/Android系统应用框架篇/Android显示框架/02Android显示框架：Android应用视图载体View.md)中已经做了详细的阐述。
- View 带有滑嵌套时，需要处理好滑动冲突。
- View 里的线程和动画需要及时的停止，另外 View 内部提供了 postXXX()系列方法，无需再用 Handler 去做线程切换。

一个标准的自定义 View 模板

自定义属性

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>

    <declare-styleable name="StandardView">
        <attr name="color" format="color" />
    </declare-styleable>

</resources>
```
