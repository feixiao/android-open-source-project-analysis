## Android 窗口管理框架篇：Android 窗口管理框架概述

![](../../../art/app/ui/android_ui_system.png)

从上图可以看出，Android 的显示系统分为 3 层：

- **UI 框架层**：负责管理窗口中 View 组件的布局与绘制以及响应用户输入事件
- **WindowManagerService** 层：负责管理窗口 Surface 的布局与次序
- **SurfaceFlinger** 层：将 WindowManagerService 管理的窗口按照一定的次序显示在屏幕上

在 Android 显示框架里有这么几个角色：

- Activity：应用视图的容器。
- Window：应用窗口的抽象表示，它的实际表现是 View。
- View：实际显示的应用视图。
- WindowManagerService：用来创建、管理和销毁 Window。

后续的分析思路是这样的，我们先分析最上层的 View，然后依次是 Window、WindowManagerService。这样可以由浅入深，便于理解。至于 Activity 我们会放在 Android 组件框架里分析。
