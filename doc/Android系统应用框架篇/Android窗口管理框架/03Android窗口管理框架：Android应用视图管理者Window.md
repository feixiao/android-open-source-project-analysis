## Android 窗口管理框架：Android 应用视图的管理者 Window

**文章目录**

- 一 窗口类型
- 二 窗口参数
- 三 窗口模式
- 四 窗口回调
- 五 窗口实现

从这篇文章开始，我们来分析和 Window 以及 WindowManager 相关的内容，

> Abstract base class for a top-level window look and behavior policy.

> Window 在 Android 是一个窗口的概念，日常开发中我们和它接触的不多，我们更多接触的是 View，但是 View 都是通过 Window 来呈现的，Window 是 View 的直接管理者。
> 而 WindowManager 承担者管理 Window 的责任。

### 一 窗口类型

![](../../../art/app/ui/window_layer.png)

Window 在 Android 中有三种类型：

- 应用 Window：z-index 在 1~99 之间，它往往对应着一个 Activity。
- 子 Window：z-index 在 1000~1999 之间，它往往不能独立存在，需要依附在父 Window 上，例如 Dialog 等。
- 系统 Window：z-index 在 2000~2999 之间，它往往需要声明权限才能创建，例如 Toast、状态栏、系统音量条、错误提示框都是系统 Window。

> z-index 是 Android 窗口的层级的概念，z-index 越大的窗口越居于顶层，

z-index 对应着 WindowManager.LayoutParams 里的 type 参数，具体说来。

应用 Window

- public static final int FIRST_APPLICATION_WINDOW = 1;//1
- public static final int TYPE_BASE_APPLICATION = 1;//窗口的基础值，其他的窗口值要大于这个值
- public static final int TYPE_APPLICATION = 2;//普通的应用程序窗口类型
- public static final int TYPE_APPLICATION_STARTING = 3;//应用程序启动窗口类型，用于系统在应用程序窗口启动前显示的窗口。
- public static final int TYPE_DRAWN_APPLICATION = 4;
- public static final int LAST_APPLICATION_WINDOW = 99;//2

子 Window

- public static final int FIRST_SUB_WINDOW = 1000;//子窗口类型初始值
- public static final int TYPE_APPLICATION_PANEL = FIRST_SUB_WINDOW;
- public static final int TYPE_APPLICATION_MEDIA = FIRST_SUB_WINDOW + 1;
- public static final int TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW + 2;
- public static final int TYPE_APPLICATION_ATTACHED_DIALOG = FIRST_SUB_WINDOW + 3;
- public static final int TYPE_APPLICATION_MEDIA_OVERLAY = FIRST_SUB_WINDOW + 4;
- public static final int TYPE_APPLICATION_ABOVE_SUB_PANEL = FIRST_SUB_WINDOW + 5;
- public static final int LAST_SUB_WINDOW = 1999;//子窗口类型结束值

系统 Window

- public static final int FIRST_SYSTEM_WINDOW = 2000;//系统窗口类型初始值
- public static final int TYPE_STATUS_BAR = FIRST_SYSTEM_WINDOW;//系统状态栏窗口
- public static final int TYPE_SEARCH_BAR = FIRST_SYSTEM_WINDOW+1;//搜索条窗口
- public static final int TYPE_PHONE = FIRST_SYSTEM_WINDOW+2;//通话窗口
- public static final int TYPE_SYSTEM_ALERT = FIRST_SYSTEM_WINDOW+3;//系统 ALERT 窗口
- public static final int TYPE_KEYGUARD = FIRST_SYSTEM_WINDOW+4;//锁屏窗口
- public static final int TYPE_TOAST = FIRST_SYSTEM_WINDOW+5;//TOAST 窗口

### 二 窗口参数

在 WindowManager 里定义了一个 LayoutParams 内部类，它描述了窗口的参数信息，主要包括：

- public int x：窗口 x 轴坐标
- public int y：窗口 y 轴坐标
- public int type：窗口类型
- public int flags：窗口属性
- public int softInputMode：输入法键盘模式

关于窗口属性，它控制着窗口的行为，举几个常见的：

- FLAG_ALLOW_LOCK_WHILE_SCREEN_ON 只要窗口可见，就允许在开启状态的屏幕上锁屏
- FLAG_NOT_FOCUSABLE 窗口不能获得输入焦点，设置该标志的同时，FLAG_NOT_TOUCH_MODAL 也会被设置
- FLAG_NOT_TOUCHABLE 窗口不接收任何触摸事件
- FLAG_NOT_TOUCH_MODAL 在该窗口区域外的触摸事件传递给其他的 Window,而自己只会处理窗口区域内的触摸事件
- FLAG_KEEP_SCREEN_ON 只要窗口可见，屏幕就会一直亮着
- FLAG_LAYOUT_NO_LIMITS 允许窗口超过屏幕之外
- FLAG_FULLSCREEN 隐藏所有的屏幕装饰窗口，比如在游戏、播放器中的全屏显示
- FLAG_SHOW_WHEN_LOCKED 窗口可以在锁屏的窗口之上显示
- FLAG_IGNORE_CHEEK_PRESSES 当用户的脸贴近屏幕时（比如打电话），不会去响应此事件
- FLAG_TURN_SCREEN_ON 窗口显示时将屏幕点亮

关于窗口类型，它对应着窗口的层级，上面我们也提到过了。

它的构造函数也主要是针对这几个参数的。

```java
 public static class LayoutParams extends ViewGroup.LayoutParams implements Parcelable {
            public LayoutParams() {
                super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
                type = TYPE_APPLICATION;
                format = PixelFormat.OPAQUE;
            }

            public LayoutParams(int _type) {
                super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
                type = _type;
                format = PixelFormat.OPAQUE;
            }

            public LayoutParams(int _type, int _flags) {
                super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
                type = _type;
                flags = _flags;
                format = PixelFormat.OPAQUE;
            }

            public LayoutParams(int _type, int _flags, int _format) {
                super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
                type = _type;
                flags = _flags;
                format = _format;
            }

            public LayoutParams(int w, int h, int _type, int _flags, int _format) {
                super(w, h);
                type = _type;
                flags = _flags;
                format = _format;
            }

            public LayoutParams(int w, int h, int xpos, int ypos, int _type,
                    int _flags, int _format) {
                super(w, h);
                x = xpos;
                y = ypos;
                type = _type;
                flags = _flags;
                format = _format;
            }
 }
```

### 三 窗口模式

关于窗口模式我们就比较熟悉了，我们会在 AndroidManifest.xml 里 Activity 的标签下设置 android:windowSoftInputMode="adjustNothing"，来控制输入键盘显示行为。

可选的有 6 个参数，源码里也有 6 个值与之对应：

- SOFT_INPUT_STATE_UNSPECIFIED：没有指定软键盘输入区域的显示状态。
- SOFT_INPUT_STATE_UNCHANGED：不要改变软键盘输入区域的显示状态。
- SOFT_INPUT_STATE_HIDDEN：在合适的时候隐藏软键盘输入区域，例如，当用户导航到当前窗口时。
- SOFT_INPUT_STATE_ALWAYS_HIDDEN：当窗口获得焦点时，总是隐藏软键盘输入区域。
- SOFT_INPUT_STATE_VISIBLE：在合适的时候显示软键盘输入区域，例如，当用户导航到当前窗口时。
- SOFT_INPUT_STATE_ALWAYS_VISIBLE：当窗口获得焦点时，总是显示软键盘输入区域。

当然，我们也可以通过代码设置键盘模式。

```java
getWindow().setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE);
```

### 四 窗口回调

Window 里定义了一个 Callback 接口，Activity 实现了 Window.Callback 接口，将 Activity 关联给 Window，Window 就可以将一些事件交由 Activity 处理。

```java
 public interface Callback {

        //键盘事件分发
        public boolean dispatchKeyEvent(KeyEvent event);

        //触摸事件分发
        public boolean dispatchTouchEvent(MotionEvent event);

        //轨迹球事件分发
        public boolean dispatchTrackballEvent(MotionEvent event);

        //可见性事件分发
        public boolean dispatchPopulateAccessibilityEvent(AccessibilityEvent event);

        //创建Panel View
        public View onCreatePanelView(int featureId);

        //创建menu
        public boolean onCreatePanelMenu(int featureId, Menu menu);

        //画板准备好时回调
        public boolean onPreparePanel(int featureId, View view, Menu menu);

        //menu打开时回调
        public boolean onMenuOpened(int featureId, Menu menu);

        //menu item被选择时回调
        public boolean onMenuItemSelected(int featureId, MenuItem item);

        //Window Attributes发生变化时回调
        public void onWindowAttributesChanged(WindowManager.LayoutParams attrs);

        //Content View发生变化时回调
        public void onContentChanged();

        //窗口焦点发生变化时回调
        public void onWindowFocusChanged(boolean hasFocus);

        //Window被添加到WIndowManager时回调
        public void onAttachedToWindow();

        //Window被从WIndowManager中移除时回调
        public void onDetachedFromWindow();

         */
        //画板关闭时回调
        public void onPanelClosed(int featureId, Menu menu);

        //用户开始执行搜索操作时回调
        public boolean onSearchRequested();
    }
```

### 五 窗口实现

Window 是一个抽象类，它的唯一实现类是 PhoneWindow，PhoneWindow 里包含了以下内容：

- private DecorView mDecor：DecorView 是 Activity 中的顶级 View，它本质上是一个 FrameLayout，一般说来它内部包含标题栏和内容栏（com.android.internal.R.id.content）。
- ViewGroup mContentParent：窗口内容视图，它是 mDecor 本身或者是它的子 View。
- private ImageView mLeftIconView：左上角图标
- private ImageView mRightIconView：右上角图标
- private ProgressBar mCircularProgressBar：圆形 loading 条
- private ProgressBar mHorizontalProgressBar：水平 loading 条
- 其他的一些和转场动画相关的 Transition 与 listener

看到这些，大家有没有觉得很熟悉，这就是我们日常开发中经常见到的东西，它在 PhoneWindow 里被创建。另外，我们在 Activity 里经常调用的方法，它的实际实现也是
在 PhoneWindow 里，我们分别来看一看。

### setContentView()

这是一个我们非常熟悉的方法，只不过我们通常是在 Activity 里进行调用，但是它的实际实现是在 PhoneWindow 里。

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {

    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            //1. 如果没有DecorView则创建它，并将创建好的DecorView赋值给mContentParent
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            //2. 将Activity传入的布局文件生成View并添加到mContentParent中
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            //3. 回调Window.Callback里的onContentChanged()方法，这个Callback也被Activity
            //所持有，因此它实际回调的是Activity里的onContentChanged()方法，通知Activity
            //视图已经发生改变。
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
}
```

这个方法主要做了两件事情：

1. 如果没有 DecorView 则创建它，并将创建好的 DecorView 赋值给 mContentParent
2. 将 Activity 传入的布局文件生成 View 并添加到 mContentParent 中
3. 回调 Window.Callback 里的 onContentChanged()方法，这个 Callback 也被 Activity 所持有，因此它实际回调的是 Activity 里的 onContentChanged()方法，通知 Activity 视图已经发生改变。

创建 DecorView 是通过 installDecor()方法完成的，它的逻辑也非常简单，就是创建了一个 ViewGroup 然后返回给了 mDecor 和 mContentParent。

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {

 public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;

  private void installDecor() {
         mForceDecorInstall = false;
         if (mDecor == null) {
             //生成DecorView
             mDecor = generateDecor(-1);
             mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
             mDecor.setIsRootNamespace(true);
             if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                 mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
             }
         } else {
             mDecor.setWindow(this);
         }
         if (mContentParent == null) {
             mContentParent = generateLayout(mDecor);
             ...
             } else {
                ...
             }
            ...
         }
     }

 protected ViewGroup generateLayout(DecorView decor) {
        //读取并设置主题颜色、状态栏颜色等信息
        ...
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        //设置窗口参数等信息
        ...
        return contentParent;
    }
}
```

通过以上这些流程，DecorView 已经被创建并初始化完毕，Activity 里的布局文件也被成功的添加到 PhoneWindow 的 mContentParent（实际上就是 DecorView，它是 Activity 的顶层 View）
中，但是这个时候 DecorView 还没有真正的被 WindowManager 添加到 Window 中，它还无法接受用户的输入信息和焦点事件，这个时候就相当于走到了 Activity 的 onCreate()流程，界面还
未展示给用户。

直到走到 Activity 的 onResume()方法，它会调用 Activity 的 makeVisiable()方法，DecorView 才真正的被用户所看到。

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback {

    void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
}
```

通常以上的分析，我们理解了 setContentView 的工作原理，另外还有 addContentView、clearContentView，正如它们的名字那样，setContentView 是替换 View，addContentView 是添加 View。实现原理相同。

好了，以上便是本篇文章的全部内容，下一篇文章我们来分析 WindowManager 的内容，分析 Window 的添加、移除和更新的流程。

### 附录

文章末尾给大家提供一个 WindowUtils 工具类。

```java
import android.animation.ValueAnimator;
import android.app.Activity;
import android.content.Context;
import android.content.res.Configuration;
import android.view.Surface;
import android.view.Window;
import android.view.WindowManager;

public final class WindowUtils {

    /**
     * Don't let anyone instantiate this class.
     */
    private WindowUtils() {
        throw new Error("Do not need instantiate!");
    }

    /**
     * 获取当前窗口的旋转角度
     *
     * @param activity activity
     * @return  int
     */
    public static int getDisplayRotation(Activity activity) {
        switch (activity.getWindowManager().getDefaultDisplay().getRotation()) {
            case Surface.ROTATION_0:
                return 0;
            case Surface.ROTATION_90:
                return 90;
            case Surface.ROTATION_180:
                return 180;
            case Surface.ROTATION_270:
                return 270;
            default:
                return 0;
        }
    }

    /**
     * 当前是否是横屏
     *
     * @param context  context
     * @return  boolean
     */
    public static final boolean isLandscape(Context context) {
        return context.getResources().getConfiguration().orientation == Configuration.ORIENTATION_LANDSCAPE;
    }

    /**
     * 当前是否是竖屏
     *
     * @param context  context
     * @return   boolean
     */
    public static final boolean isPortrait(Context context) {
        return context.getResources().getConfiguration().orientation == Configuration.ORIENTATION_PORTRAIT;
    }
    /**
     *  调整窗口的透明度  1.0f,0.5f 变暗
     * @param from  from>=0&&from<=1.0f
     * @param to  to>=0&&to<=1.0f
     * @param context  当前的activity
     */
    public static void dimBackground(final float from, final float to, Activity context) {
        final Window window = context.getWindow();
        ValueAnimator valueAnimator = ValueAnimator.ofFloat(from, to);
        valueAnimator.setDuration(500);
        valueAnimator.addUpdateListener(
                new ValueAnimator.AnimatorUpdateListener() {
                    @Override
                    public void onAnimationUpdate(ValueAnimator animation) {
                        WindowManager.LayoutParams params
                                = window.getAttributes();
                        params.alpha = (Float) animation.getAnimatedValue();
                        window.setAttributes(params);
                    }
                });
        valueAnimator.start();
    }
}
```
