# Android 显示框架：Android 应用视图的载体 View

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

**文章目录**

- 一 View 的生命周期
- 二 View 的测量流程
- 三 View 的布局流程
- 四 View 的绘制流程
- 五 View 事件分发机制

> This class represents the basic building block for user interface components. A Viewoccupies a rectangular area on the screen and is
> responsible for drawing and event handling.

View 是屏幕上的一块矩形区域，负责界面的绘制与触摸事件的处理，它是一种界面层控件的抽象，所有的控件都继承自 View。

View 是 Android 显示框架中较为复杂的一环，首先是它的生命周期会随着 Activity 的生命周期进行变化，掌握 View 的生命周期对我们自定义 View 有着重要的意义。另一个方面 View 从 ViewRoot.performTraversals()开始
经历 measure、layout、draw 三个流程最终显示在用户面前，用户在点击屏幕时，点击事件随着 Activity 传入 Window，最终由 ViewGroup/View 进行分发处理。今天我们就围绕着这些主题进行展开分析。

## 一 View 生命周期

在 View 中有诸多回调方法，它们在 View 的不同生命周期阶段调用，常用的有以下方法。

我们写一个简单的自定义 View 来观察 View 与 Activity 的生命周期变化。

```java
public class CustomView extends View {

    private static final String TAG = "View";

    public CustomView(Context context) {
        super(context);
        Log.d(TAG, "CustomView()");
    }

    public CustomView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        Log.d(TAG, "CustomView()");
    }

    public CustomView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        Log.d(TAG, "CustomView()");
    }

    /**
     * View在xml文件里加载完成时调用
     */
    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        Log.d(TAG, "View onFinishInflate()");
    }

    /**
     * 测量View及其子View大小时调用
     */
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        Log.d(TAG, "View onMeasure()");
    }

    /**
     * 布局View及其子View大小时调用
     */
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed, left, top, right, bottom);
        Log.d(TAG, "View onLayout() left = " + left + " top = " + top + " right = " + right + " bottom = " + bottom);
    }

    /**
     * View大小发生改变时调用
     */
    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        Log.d(TAG, "View onSizeChanged() w = " + w + " h = " + h + " oldw = " + oldw + " oldh = " + oldh);
    }

    /**
     * 绘制View及其子View大小时调用
     */
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Log.d(TAG, "View onDraw()");
    }

    /**
     * 物理按键事件发生时调用
     */
    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        Log.d(TAG, "View onKeyDown() event = " + event.getAction());
        return super.onKeyDown(keyCode, event);
    }

    /**
     * 物理按键事件发生时调用
     */
    @Override
    public boolean onKeyUp(int keyCode, KeyEvent event) {
        Log.d(TAG, "View onKeyUp() event = " + event.getAction());
        return super.onKeyUp(keyCode, event);
    }

    /**
     * 触摸事件发生时调用
     */
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.d(TAG, "View onTouchEvent() event =  " + event.getAction());
        return super.onTouchEvent(event);
    }

    /**
     * View获取焦点或者失去焦点时调用
     */
    @Override
    protected void onFocusChanged(boolean gainFocus, int direction, @Nullable Rect previouslyFocusedRect) {
        super.onFocusChanged(gainFocus, direction, previouslyFocusedRect);
        Log.d(TAG, "View onFocusChanged() gainFocus = " + gainFocus);
    }

    /**
     * View所在窗口获取焦点或者失去焦点时调用
     */
    @Override
    public void onWindowFocusChanged(boolean hasWindowFocus) {
        super.onWindowFocusChanged(hasWindowFocus);
        Log.d(TAG, "View onWindowFocusChanged() hasWindowFocus = " + hasWindowFocus);
    }

    /**
     * View被关联到窗口时调用
     */
    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        Log.d(TAG, "View onAttachedToWindow()");
    }

    /**
     * View从窗口分离时调用
     */
    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        Log.d(TAG, "View onDetachedFromWindow()");
    }

    /**
     * View的可见性发生变化时调用
     */
    @Override
    protected void onVisibilityChanged(@NonNull View changedView, int visibility) {
        super.onVisibilityChanged(changedView, visibility);
        Log.d(TAG, "View onVisibilityChanged() visibility = " + visibility);
    }

    /**
     * View所在窗口的可见性发生变化时调用
     */
    @Override
    protected void onWindowVisibilityChanged(int visibility) {
        super.onWindowVisibilityChanged(visibility);
        Log.d(TAG, "View onWindowVisibilityChanged() visibility = " + visibility);
    }
}
```

Activity 与 View 的生命周期变化一目了然。

Activity create

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/view_lifecycle_create.png"/>

Activity pause

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/view_lifecycle_pause.png"/>

Activity resume

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/view_lifecycle_resume.png"/>

Activity destory

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/view_lifecycle_destory.png"/>

我们来总结一下 View 的声明周期随着 Activity 生命周期变化的情况。

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/view_lifecycle.png"/>

我们了解这些生命周期方法有什么作用呢？🤔

其实这些方法在我们自定义 View 的时候发挥着很大的作用，我们来举几种应用场景。

场景 1：在 Activity 启动时获取 View 的宽高，但是在 onCreate、onStart 和 onResume 均无法获取正确的结果。这是因为在 Activity 的这些方法里，Viewed 绘制可能还没有完成，我们可以在 View 的生命周期方法里获取。

```java
@Override
public void onWindowFocusChanged(boolean hasFocus) {
    super.onWindowFocusChanged(hasFocus);
    if(hasFocus){
        int width = view.getMeasuredWidth();
        int height = view.getMeasuredHeight();
    }
}
```

场景 2：在 Activity 生命周期发生变化时，View 也要做响应的处理，典型的有 VideoView 保存进度和恢复进度。

```java
@Override
protected void onVisibilityChanged(@NonNull View changedView, int visibility) {
    super.onVisibilityChanged(changedView, visibility);
    //TODO do something if activity lifecycle changed if necessary
    //Activity onResume()
    if(visibility == VISIBLE){

    }
    //Activity onPause()
    else {

    }
}

@Override
public void onWindowFocusChanged(boolean hasWindowFocus) {
    super.onWindowFocusChanged(hasWindowFocus);

    //TODO do something if activity lifecycle changed if necessary
    //Activity onResume()
    if (hasWindowFocus) {
    }
    //Activity onPause()
    else {
    }
}
```

场景 3：释放线程、资源

```java
@Override
protected void onDetachedFromWindow() {
    super.onDetachedFromWindow();
    //TODO release resources, thread, animation
}
```

## 二 View 的测量流程

View 是一个矩形区域，它有自己的位置、大小与边距。

View 位置

> View 位置：有左上角坐标(getLeft(), getTop())决定，该坐标是以它的父 View 的左上角为坐标原点，单位是 pixels。

View 大小

> View 大小：View 的大小有两对值来表示。getMeasuredWidth()/getMeasuredHeight()这组值表示了该 View 在它的父 View 里期望的大小值，在 measure()方法完成后可获得。
> getWidth()/getHeight()这组值表示了该 View 在屏幕上的实际大小，在 draw()方法完成后可获得。

View 内边距

> View 内边距：View 的内边距用 padding 来表示，它表示 View 的内容距离 View 边缘的距离。通过 getPaddingXXX()方法获取。需要注意的是我们在自定义 View 的时候需要单独处理
> padding，否则它不会生效，这一块的内容我们会在 View 自定义实践系列的文章中展开。

View 外边距

> View 内边距：View 的外边距用 margin 来表示，它表示 View 的边缘离它相邻的 View 的距离。

> Measure 过程决定了 View 的宽高，该过程完成后，通常都可以通过 getMeasuredWith()/getMeasuredHeight()获得宽高。

理解了上面这些概念，我们接下来来看看详细的测量流程。

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/measure_sequence.png" height="500"/>

View 的测量流程看似复杂，实际遵循着简单的逻辑。

在做测量的时候，measure()方法被父 View 调用，在 measure()中做一些准备和优化工作后，调用 onMeasure()来进行实际的自我测量。对于 onMeasure()，View 和 ViewGroup 有所区别：

- View：View 在 onMeasure() 中会计算出自己的尺寸然后保存；
- ViewGroup：ViewGroup 在 onMeasure()中会调用所有子 View 的 measure()让它们进行自我测量，并根据子 View 计算出的期望尺寸来计算出它们的实际尺寸和位置然后保存。同时，它也会
  根据子 View 的尺寸和位置来计算出自己的尺寸然后保存.

在介绍测量流程之前，我们先来介绍下 MeasureSpec，它用来把测量要求从父 View 传递给子 View。我们知道 View 的大小最终由子 View 的 LayoutParams 与父 View 的测量要求公共决定，测量要求指的
就是这个 MeasureSpec，它是一个 32 位 int 值。

- 高 2 位：SpecMode，测量模式
- 低 30 位：SpecSize，在特定测量模式下的大小

测量模式有三种：

```java
public static class MeasureSpec {

    private static final int MODE_SHIFT = 30;
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

    //父View不对子View做任何限制，需要多大给多大，这种情况一般用于系统内部，表示一种测量的状态
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;

    //父View已经检测出View所需要的精确大小，这个时候View的最终大小就是SpecSize所指定的值，它对应LayoutParams中的match_parent和具体数值这两种模式
    public static final int EXACTLY     = 1 << MODE_SHIFT;

    //父View给子VIew提供一个最大可用的大小，子View去自适应这个大小。
    public static final int AT_MOST     = 2 << MODE_SHIFT;
}
```

日常开发中我们接触最多的不是 MeasureSpec 而是 LayoutParams，在 View 测量的时候，LayoutParams 会和父 View 的 MeasureSpec 相结合被换算成 View 的 MeasureSpec，进而决定 View 的大小。

View 的 MeasureSpec 计算源码如下所示：

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {

     public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
            int specMode = MeasureSpec.getMode(spec);
            int specSize = MeasureSpec.getSize(spec);

            int size = Math.max(0, specSize - padding);

            int resultSize = 0;
            int resultMode = 0;

            switch (specMode) {
            // Parent has imposed an exact size on us
            case MeasureSpec.EXACTLY:
                if (childDimension >= 0) {
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    // Child wants to be our size. So be it.
                    resultSize = size;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    // Child wants to determine its own size. It can't be
                    // bigger than us.
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;

            // Parent has imposed a maximum size on us
            case MeasureSpec.AT_MOST:
                if (childDimension >= 0) {
                    // Child wants a specific size... so be it
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    // Child wants to be our size, but our size is not fixed.
                    // Constrain child to not be bigger than us.
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    // Child wants to determine its own size. It can't be
                    // bigger than us.
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;

            // Parent asked to see how big we want to be
            case MeasureSpec.UNSPECIFIED:
                if (childDimension >= 0) {
                    // Child wants a specific size... let him have it
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    // Child wants to be our size... find out how big it should
                    // be
                    resultSize = 0;
                    resultMode = MeasureSpec.UNSPECIFIED;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    // Child wants to determine its own size.... find out how
                    // big it should be
                    resultSize = 0;
                    resultMode = MeasureSpec.UNSPECIFIED;
                }
                break;
            }
            return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
        }

}
```

该方法用来获取子 View 的 MeasureSpec，由参数我们就可以知道子 View 的 MeasureSpec 由父容器的 spec，父容器中已占用的的空间大小
padding，以及子 View 自身大小 childDimension 共同来决定的。

通过上述方法，我们可以总结出普通 View 的 MeasureSpec 的创建规则。

- 当 View 采用固定宽高的时候，不管父容器的 MeasureSpec 是什么，resultSize 都是指定的宽高，resultMode 都是 MeasureSpec.EXACTLY。
- 当 View 的宽高是 match_parent，当父容器是 MeasureSpec.EXACTLY，则 View 也是 MeasureSpec.EXACTLY，并且其大小就是父容器的剩余空间。当父容器是 MeasureSpec.AT_MOST
  则 View 也是 MeasureSpec.AT_MOST，并且大小不会超过父容器的剩余空间。
- 当 View 的宽高是 wrap_content 时，不管父容器的模式是 MeasureSpec.EXACTLY 还是 MeasureSpec.AT_MOST，View 的模式总是 MeasureSpec.AT_MOST，并且大小都不会超过父类的剩余空间。

了解了 MeasureSpec 的概念之后，我就就可以开始分析测量流程了。

- 对于顶级 View（DecorView）其 MeasureSpec 由窗口的尺寸和自身的 LayoutParams 共同确定的。
- 对于普通 View 其 MeasureSpec 由父容器的 Measure 和自身的 LayoutParams 共同确定的。

View 的绘制会先调用 View 的 measure()方法，measure()方法用来测量 View 的大小，实际的测量工作是由 View 的 onMeasure()来完成的。我们来看看
onMeasure(int widthMeasureSpec, int heightMeasureSpec)方法的实现。

**关键点 1：View.onMeasure(int widthMeasureSpec, int heightMeasureSpec)**

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {

       protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
           setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                   getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
       }

       //设置View宽高的测量值
       protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);

       //measureSpec指的是View测量后的大小
       public static int getDefaultSize(int size, int measureSpec) {
           int result = size;
           int specMode = MeasureSpec.getMode(measureSpec);
           int specSize =  MeasureSpec.getSize(measureSpec);

           switch (specMode) {
           //MeasureSpec.UNSPECIFIED一般用来系统的内部测量流程
           case MeasureSpec.UNSPECIFIED:
               result = size;
               break;
           //我们主要关注着两种情况，它们返回的是View测量后的大小
           case MeasureSpec.AT_MOST:
           case MeasureSpec.EXACTLY:
               result = specSize;
               break;
           }
           return result;
       }

       //如果View没有设置背景，那么返回android:minWidth这个属性的值，这个值可以为0
       //如果View设置了背景，那么返回android:minWidth和背景最小宽度两者中的最大值。
       protected int getSuggestedMinimumHeight() {
           int suggestedMinHeight = mMinHeight;

           if (mBGDrawable != null) {
               final int bgMinHeight = mBGDrawable.getMinimumHeight();
               if (suggestedMinHeight < bgMinHeight) {
                   suggestedMinHeight = bgMinHeight;
               }
           }

           return suggestedMinHeight;
       }
}
```

View 的 onMeasure()方法实现比较简单，它调用 setMeasuredDimension()方法来设置 View 的测量大小，测量的大小通过 getDefaultSize()方法来获取。

如果我们直接继承 View 来自定义 View 时，需要重写 onMeasure()方法，并设置 wrap_content 时的大小。为什么呢？🤔

通过上面的描述我们知道，当 LayoutParams 为 wrap_content 时，SpecMode 为 AT_MOST，而在

关于 getDefaultSize(int size, int measureSpec) 方法需要说明一下，通过上面的描述我们知道 etDefaultSize()方法中 AT_MOST 与 EXACTLY 模式下，返回的
都是 specSize，这个 specSize 是父 View 当前可以使用的大小，如果不处理，那 wrap_content 就相当于 match_parent。

如何处理？🤔

```java
@Override
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
      super.onMeasure(widthMeasureSpec, heightMeasureSpec);
      Log.d(TAG, "widthMeasureSpec = " + widthMeasureSpec + " heightMeasureSpec = " + heightMeasureSpec);

      //指定一组默认宽高，至于具体的值是多少，这就要看你希望在wrap_cotent模式下
      //控件的大小应该设置多大了
      int mWidth = 200;
      int mHeight = 200;

      int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
      int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);

      int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
      int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);

      if (widthSpecMode == MeasureSpec.AT_MOST && heightMeasureSpec == MeasureSpec.AT_MOST) {
          setMeasuredDimension(mWidth, mHeight);
      } else if (widthSpecMode == MeasureSpec.AT_MOST) {
          setMeasuredDimension(mWidth, heightSpecSize);
      } else if (heightSpecMode == MeasureSpec.AT_MOST) {
          setMeasuredDimension(widthSpecSize, mHeight);
      }
  }
```

注：你可以自己尝试一下自定义一个 View，然后不重写 onMeasure()方法，你会发现只有设置 match_parent 和 wrap_content 效果是一样的，事实上 TextView、ImageView
等系统组件都在 wrap_content 上有自己的处理，可以去翻一翻源码。

看完了 View 的 measure 过程，我们再来看看 ViewGroup 的 measure 过程。ViewGroup 继承于 View，是一个抽象类，它并没有重写 onMeasure()方法，因为不同布局类型的测量
流程各不相同，因此 onMeasure()方法由它的子类来实现。

我们来看个 FrameLayout 的 onMeasure()方法的实现。

**关键点 2：FrameLayout.onMeasure(int widthMeasureSpec, int heightMeasureSpec)**

View.onMeasure()方法的具体实现一般是由其子类来完成的，对于应用窗口的顶级视图 DecorView 来说，它继承于 FrameLayout，我们来看看 FrameLayout.onMeasure()
方法的实现。

```java
public class FrameLayout extends ViewGroup {

       @Override
       protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
           final int count = getChildCount();

           int maxHeight = 0;
           int maxWidth = 0;

           // Find rightmost and bottommost child
           for (int i = 0; i < count; i++) {
               final View child = getChildAt(i);
               if (mMeasureAllChildren || child.getVisibility() != GONE) {
                   measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                   maxWidth = Math.max(maxWidth, child.getMeasuredWidth());
                   maxHeight = Math.max(maxHeight, child.getMeasuredHeight());
               }
           }

           // Account for padding too
           maxWidth += mPaddingLeft + mPaddingRight + mForegroundPaddingLeft + mForegroundPaddingRight;
           maxHeight += mPaddingTop + mPaddingBottom + mForegroundPaddingTop + mForegroundPaddingBottom;

           // Check against our minimum height and width
           maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
           maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

           // Check against our foreground's minimum height and width
           final Drawable drawable = getForeground();
           if (drawable != null) {
               maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
               maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
           }

           setMeasuredDimension(resolveSize(maxWidth, widthMeasureSpec),
                   resolveSize(maxHeight, heightMeasureSpec));
       }

      public static int resolveSize(int size, int measureSpec) {
           int result = size;
           int specMode = MeasureSpec.getMode(measureSpec);
           int specSize =  MeasureSpec.getSize(measureSpec);
           switch (specMode) {
           case MeasureSpec.UNSPECIFIED:
               result = size;
               break;
           case MeasureSpec.AT_MOST:
               result = Math.min(size, specSize);
               break;
           case MeasureSpec.EXACTLY:
               result = specSize;
               break;
           }
           return result;
       }
}
```

可以看到该方法主要做了以下事情：

1. 调用 measureChildWithMargins()去测量每一个子 View 的大小，找到最大高度和宽度保存在 maxWidth/maxHeigth 中。
2. 将上一步计算的 maxWidth/maxHeigth 加上 padding 值，mPaddingLeft，mPaddingRight，mPaddingTop ，mPaddingBottom 表示当前内容区域的左右上下四条边分别到当前视图的左右上下四条边的距离，
   mForegroundPaddingLeft ，mForegroundPaddingRight，mForegroundPaddingTop ，mForegroundPaddingBottom 表示当前视图的各个子视图所围成的区域的左右上下四条边到当前视图前景区域的
   左右上下四条边的距离，经过计算获得最终宽高。
3. 当前视图是否设置有最小宽度和高度。如果设置有的话，并且它们比前面计算得到的宽度 maxWidth 和高度 maxHeight 还要大，那么就将它们作为当前视图的宽度和高度值。
4. 当前视图是否设置有前景图。如果设置有的话，并且它们比前面计算得到的宽度 maxWidth 和高度 maxHeight 还要大，那么就将它们作为当前视图的宽度和高度值。
5. 经过以上的计算，就得到了正确的宽高，先调用 resolveSize()方法，获取 MeasureSpec，接着调用父类的 setMeasuredDimension()方法将它们作为当前视图的大小。

我们再来看看 resolveSize(int size, int measureSpec)方法是如果获取 MeasureSpec 的？

这个方法的两个参数：int size：前面计算出的最大宽/高，int measureSpec 父视图指定的 MeasureSpec，它们按照：

- MeasureSpec.UNSPECIFIED: 取 size
- MeasureSpec.AT_MOST: 取 size, specSize 的最小值
- MeasureSpec.EXACTLY: 取 specSize

来生成最后的大小。

以上便是 Measure 的整个流程，该流程完成以后，我们可以通过 getMeasuredWidth()与 getMeasuredHeight()来获得 View 的宽高。但是在某些情况下，系统需要经过多次 Measure 才能确定
最终的宽高，因此在 onMeasure()方法中拿到的宽高很可能是不正确的，比较好的做法是在 onLayout()方法中获取 View 的宽高。

## 三 View 的布局流程

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/layout_sequence.png" height="500"/>

在进行布局的时候，layout()方法被父 View 调用，在 layout()中它会保存父 View 传进来的自己的位置和尺寸，并且调用 onLayout()来进行实际的内部布局。对于 onLayout()，View 和 ViewGroup 有所区别：

- View：由于没有子 View，所以 View 的 onLayout() 什么也不做。
- ViewGroup：ViewGroup 在 onLayout()中会调用自己的所有子 View 的 layout()方法，把它们的尺寸和位置传给它们，让它们完成自我的内部布局。

layout()方法用来确定 View 本身的位置，onLayout()方法用来确定子元素的位置。

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {

   public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        //1 调用setFrame()设置View四个顶点ed位置
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {

            //2 调用onLayout()确定View子元素的位置
            onLayout(changed, l, t, r, b);

            if (shouldDrawRoundScrollbar()) {
                if(mRoundScrollbarRenderer == null) {
                    mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
                }
            } else {
                mRoundScrollbarRenderer = null;
            }

            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }
}
```

**关键点 1：View.invalidate()**

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {

    public void invalidate() {
        if (ViewDebug.TRACE_HIERARCHY) {
            ViewDebug.trace(this, ViewDebug.HierarchyTraceType.INVALIDATE);
        }

        //检查mPrivateFlags的DRAWN位与HAS_BOUNDS是否被置1，说明上一次请求执行的UI绘制已经完成了，这个时候才能执行新的UI绘制操作
        if ((mPrivateFlags & (DRAWN | HAS_BOUNDS)) == (DRAWN | HAS_BOUNDS)) {
            //将mPrivateFlags的DRAWN位与HAS_BOUNDS是否被置0
            mPrivateFlags &= ~DRAWN & ~DRAWING_CACHE_VALID;
            final ViewParent p = mParent;
            final AttachInfo ai = mAttachInfo;
            if (p != null && ai != null) {
                final Rect r = ai.mTmpInvalRect;
                r.set(0, 0, mRight - mLeft, mBottom - mTop);
                // Don't call invalidate -- we don't want to internally scroll
                // our own bounds
                p.invalidateChild(this, r);
            }
        }
    }
}
```

该方法检查 mPrivateFlags 的 DRAWN 位与 HAS_BOUNDS 是否被置 1，说明上一次请求执行的 UI 绘制已经完成了，这个时候才能执行新的 UI 绘制操作，在执行新的 UI 绘制操作之前，还会将
这两个标志位置 0，然后调用 ViewParent.invalidateChild()方法来完成绘制操作，这个 ViewParent 指向的是 ViewRoot 对象。

**关键点 2：FrameLayout.onLayout(boolean changed, int left, int top, int right, int bottom)**

onLayout 的实现依赖于具体的布局，所以 View/ViewGroup 并没有实现这个方法，我们来看看 FrameLayout 的实现。

```java
public class FrameLayout extends ViewGroup {

    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
            final int count = getChildCount();

            final int parentLeft = mPaddingLeft + mForegroundPaddingLeft;
            final int parentRight = right - left - mPaddingRight - mForegroundPaddingRight;

            final int parentTop = mPaddingTop + mForegroundPaddingTop;
            final int parentBottom = bottom - top - mPaddingBottom - mForegroundPaddingBottom;

            mForegroundBoundsChanged = true;

            for (int i = 0; i < count; i++) {
                final View child = getChildAt(i);
                if (child.getVisibility() != GONE) {
                    final LayoutParams lp = (LayoutParams) child.getLayoutParams();

                    final int width = child.getMeasuredWidth();
                    final int height = child.getMeasuredHeight();

                    int childLeft = parentLeft;
                    int childTop = parentTop;

                    final int gravity = lp.gravity;

                    if (gravity != -1) {
                        final int horizontalGravity = gravity & Gravity.HORIZONTAL_GRAVITY_MASK;
                        final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

                        switch (horizontalGravity) {
                            case Gravity.LEFT:
                                childLeft = parentLeft + lp.leftMargin;
                                break;
                            case Gravity.CENTER_HORIZONTAL:
                                childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                                        lp.leftMargin - lp.rightMargin;
                                break;
                            case Gravity.RIGHT:
                                childLeft = parentRight - width - lp.rightMargin;
                                break;
                            default:
                                childLeft = parentLeft + lp.leftMargin;
                        }

                        switch (verticalGravity) {
                            case Gravity.TOP:
                                childTop = parentTop + lp.topMargin;
                                break;
                            case Gravity.CENTER_VERTICAL:
                                childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                                        lp.topMargin - lp.bottomMargin;
                                break;
                            case Gravity.BOTTOM:
                                childTop = parentBottom - height - lp.bottomMargin;
                                break;
                            default:
                                childTop = parentTop + lp.topMargin;
                        }
                    }

                    child.layout(childLeft, childTop, childLeft + width, childTop + height);
                }
            }
        }
}
```

我们先来解释一下这个函数里的变量的含义。

- int left, int top, int right, int bottom: 描述的是当前视图的外边距，即它与父窗口的边距。
- mPaddingLeft，mPaddingTop，mPaddingRight，mPaddingBottom: 描述的当前视图的内边距。

通过这些参数，我们就可以得到当前视图的子视图所能布局在的区域。

接着，该方法就会遍历它的每一个子 View，并获取它的左上角的坐标位置：childLeft，childTop。这两个位置信息会根据 gravity 来进行计算。
最后会调用子 View 的 layout()方法循环布局操作，直到所有的布局都完成为止。

## View 的绘制流程

> Draw 过程最终将 View 绘制在屏幕上。

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/draw_sequence.png" height="500"/>

绘制从 ViewRoot.draw()开始，它首先会创建一块画布，接着再在画布上绘制 Android 上的 UI，再把画布的内容交给 SurfaceFlinger 服务来渲染。

**关键点 1：ViewRoot.draw(boolean fullRedrawNeeded)**

```java
public final class ViewRoot extends Handler implements ViewParent,
        View.AttachInfo.Callbacks {

    private void draw(boolean fullRedrawNeeded) {
            //surface用来操作应用窗口的绘图表面
            Surface surface = mSurface;
            if (surface == null || !surface.isValid()) {
                return;
            }

            if (!sFirstDrawComplete) {
                synchronized (sFirstDrawHandlers) {
                    sFirstDrawComplete = true;
                    for (int i=0; i<sFirstDrawHandlers.size(); i++) {
                        post(sFirstDrawHandlers.get(i));
                    }
                }
            }

            scrollToRectOrFocus(null, false);

            if (mAttachInfo.mViewScrollChanged) {
                mAttachInfo.mViewScrollChanged = false;
                mAttachInfo.mTreeObserver.dispatchOnScrollChanged();
            }

            int yoff;
            //计算窗口是否处于滚动状态
            final boolean scrolling = mScroller != null && mScroller.computeScrollOffset();
            if (scrolling) {
                yoff = mScroller.getCurrY();
            } else {
                yoff = mScrollY;
            }
            if (mCurScrollY != yoff) {
                mCurScrollY = yoff;
                fullRedrawNeeded = true;
            }
            //描述窗口是否正在请求大小缩放
            float appScale = mAttachInfo.mApplicationScale;
            boolean scalingRequired = mAttachInfo.mScalingRequired;

            //描述窗口的脏区域，即需要重新绘制的区域
            Rect dirty = mDirty;
            if (mSurfaceHolder != null) {
                // The app owns the surface, we won't draw.
                dirty.setEmpty();
                return;
            }

            //用来描述是否需要用OpenGL接口来绘制UI，当应用窗口flag等于WindowManager.LayoutParams.MEMORY_TYPE_GPU
            //则表示需要用OpenGL接口来绘制UI
            if (mUseGL) {
                if (!dirty.isEmpty()) {
                    Canvas canvas = mGlCanvas;
                    if (mGL != null && canvas != null) {
                        mGL.glDisable(GL_SCISSOR_TEST);
                        mGL.glClearColor(0, 0, 0, 0);
                        mGL.glClear(GL_COLOR_BUFFER_BIT);
                        mGL.glEnable(GL_SCISSOR_TEST);

                        mAttachInfo.mDrawingTime = SystemClock.uptimeMillis();
                        mAttachInfo.mIgnoreDirtyState = true;
                        mView.mPrivateFlags |= View.DRAWN;

                        int saveCount = canvas.save(Canvas.MATRIX_SAVE_FLAG);
                        try {
                            canvas.translate(0, -yoff);
                            if (mTranslator != null) {
                                mTranslator.translateCanvas(canvas);
                            }
                            canvas.setScreenDensity(scalingRequired
                                    ? DisplayMetrics.DENSITY_DEVICE : 0);
                            mView.draw(canvas);
                            if (Config.DEBUG && ViewDebug.consistencyCheckEnabled) {
                                mView.dispatchConsistencyCheck(ViewDebug.CONSISTENCY_DRAWING);
                            }
                        } finally {
                            canvas.restoreToCount(saveCount);
                        }

                        mAttachInfo.mIgnoreDirtyState = false;

                        mEgl.eglSwapBuffers(mEglDisplay, mEglSurface);
                        checkEglErrors();

                        if (SHOW_FPS || Config.DEBUG && ViewDebug.showFps) {
                            int now = (int)SystemClock.elapsedRealtime();
                            if (sDrawTime != 0) {
                                nativeShowFPS(canvas, now - sDrawTime);
                            }
                            sDrawTime = now;
                        }
                    }
                }

                //如果窗口处于滚动状态，则应用窗口需要马上进行下一次全部重绘，调用scheduleTraversals()方法
                if (scrolling) {
                    mFullRedrawNeeded = true;
                    scheduleTraversals();
                }
                return;
            }

            //是否需要全部重绘，如果是则将窗口的脏区域设置为整个窗口区域，表示整个窗口曲云都需要重绘
            if (fullRedrawNeeded) {
                mAttachInfo.mIgnoreDirtyState = true;
                dirty.union(0, 0, (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
            }

            if (DEBUG_ORIENTATION || DEBUG_DRAW) {
                Log.v(TAG, "Draw " + mView + "/"
                        + mWindowAttributes.getTitle()
                        + ": dirty={" + dirty.left + "," + dirty.top
                        + "," + dirty.right + "," + dirty.bottom + "} surface="
                        + surface + " surface.isValid()=" + surface.isValid() + ", appScale:" +
                        appScale + ", width=" + mWidth + ", height=" + mHeight);
            }

            if (!dirty.isEmpty() || mIsAnimating) {
                Canvas canvas;
                try {
                    int left = dirty.left;
                    int top = dirty.top;
                    int right = dirty.right;
                    int bottom = dirty.bottom;
                    //调用Surface.lockCanvas()来创建画布
                    canvas = surface.lockCanvas(dirty);

                    if (left != dirty.left || top != dirty.top || right != dirty.right ||
                            bottom != dirty.bottom) {
                        mAttachInfo.mIgnoreDirtyState = true;
                    }

                    // TODO: Do this in native
                    canvas.setDensity(mDensity);
                } catch (Surface.OutOfResourcesException e) {
                    Log.e(TAG, "OutOfResourcesException locking surface", e);
                    // TODO: we should ask the window manager to do something!
                    // for now we just do nothing
                    return;
                } catch (IllegalArgumentException e) {
                    Log.e(TAG, "IllegalArgumentException locking surface", e);
                    // TODO: we should ask the window manager to do something!
                    // for now we just do nothing
                    return;
                }

                try {
                    if (!dirty.isEmpty() || mIsAnimating) {
                        long startTime = 0L;

                        if (DEBUG_ORIENTATION || DEBUG_DRAW) {
                            Log.v(TAG, "Surface " + surface + " drawing to bitmap w="
                                    + canvas.getWidth() + ", h=" + canvas.getHeight());
                            //canvas.drawARGB(255, 255, 0, 0);
                        }

                        if (Config.DEBUG && ViewDebug.profileDrawing) {
                            startTime = SystemClock.elapsedRealtime();
                        }

                        // If this bitmap's format includes an alpha channel, we
                        // need to clear it before drawing so that the child will
                        // properly re-composite its drawing on a transparent
                        // background. This automatically respects the clip/dirty region
                        // or
                        // If we are applying an offset, we need to clear the area
                        // where the offset doesn't appear to avoid having garbage
                        // left in the blank areas.
                        if (!canvas.isOpaque() || yoff != 0) {
                            canvas.drawColor(0, PorterDuff.Mode.CLEAR);
                        }

                        dirty.setEmpty();
                        mIsAnimating = false;
                        mAttachInfo.mDrawingTime = SystemClock.uptimeMillis();
                        mView.mPrivateFlags |= View.DRAWN;

                        if (DEBUG_DRAW) {
                            Context cxt = mView.getContext();
                            Log.i(TAG, "Drawing: package:" + cxt.getPackageName() +
                                    ", metrics=" + cxt.getResources().getDisplayMetrics() +
                                    ", compatibilityInfo=" + cxt.getResources().getCompatibilityInfo());
                        }
                        int saveCount = canvas.save(Canvas.MATRIX_SAVE_FLAG);
                        try {
                            canvas.translate(0, -yoff);
                            if (mTranslator != null) {
                                mTranslator.translateCanvas(canvas);
                            }
                            canvas.setScreenDensity(scalingRequired
                                    ? DisplayMetrics.DENSITY_DEVICE : 0);
                            mView.draw(canvas);
                        } finally {
                            mAttachInfo.mIgnoreDirtyState = false;
                            canvas.restoreToCount(saveCount);
                        }

                        if (Config.DEBUG && ViewDebug.consistencyCheckEnabled) {
                            mView.dispatchConsistencyCheck(ViewDebug.CONSISTENCY_DRAWING);
                        }

                        if (SHOW_FPS || Config.DEBUG && ViewDebug.showFps) {
                            int now = (int)SystemClock.elapsedRealtime();
                            if (sDrawTime != 0) {
                                nativeShowFPS(canvas, now - sDrawTime);
                            }
                            sDrawTime = now;
                        }

                        if (Config.DEBUG && ViewDebug.profileDrawing) {
                            EventLog.writeEvent(60000, SystemClock.elapsedRealtime() - startTime);
                        }
                    }

                } finally {
                    //UI绘制完成后，调用urface.unlockCanvasAndPost(canvas)S来请求SurfaceFlinger进行UI的渲染
                    surface.unlockCanvasAndPost(canvas);
                }
            }

            if (LOCAL_LOGV) {
                Log.v(TAG, "Surface " + surface + " unlockCanvasAndPost");
            }

            if (scrolling) {
                mFullRedrawNeeded = true;
                scheduleTraversals();
            }
        }
}
```

这个函数主要做了以下事情：

1. 调用 Scroller.computeScrollOffset()方法计算应用是否处于滑动状态，并获得应用窗口在 Y 轴上的即时滑动位置 yoff。
2. 根据 AttachInfo 里描述的数据，判断窗口是否需要缩放。
3. 根据成员变量 React mDirty 的描述来判断窗口脏区域的大小，脏区域指的是需要全部重绘的窗口区域。
4. 根据成员变量 boolean mUserGL 判断是否需要用 OpenGL 接口来绘制 UI，当应用窗口 flag 等于 WindowManager.LayoutParams.MEMORY_TYPE_GPU 则表示需要用 OpenGL 接口来绘制 UI.
5. 如果不是用 OpenGL 来绘制，则用 Surface 来绘制，先调用 Surface.lockCanvas()来创建画布，UI 绘制完成后，再调用 urface.unlockCanvasAndPost(canvas)S 来请求 SurfaceFlinger 进行 UI 的渲染

注：这里的 Surface 对象对应了 C++层里的 Surface 对象，真正的功能在 C++层，关于 C++层的实现，我们会在后续的文章进一步分析。

**关键点 2：View.draw(Canvas canvas)**

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {

    public void draw(Canvas canvas) {
            if (ViewDebug.TRACE_HIERARCHY) {
                ViewDebug.trace(this, ViewDebug.HierarchyTraceType.DRAW);
            }

            final int privateFlags = mPrivateFlags;
            //dirtyOpaque用来描述当前绘制，它有两种情况：1 检查DIRTY_OPAQUE为是否为1，如果是则说明当前视图某个子视图请求了一个不透明的UI绘制操作，此时当前
            //视图会被子视图覆盖 2 如果mAttachInfo.mIgnoreDirtyState = true则表示忽略该标志位
            final boolean dirtyOpaque = (privateFlags & DIRTY_MASK) == DIRTY_OPAQUE &&
                    (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);

            //将DIRTY_MASK与DRAWN置为1，表示开始绘制
            mPrivateFlags = (privateFlags & ~DIRTY_MASK) | DRAWN;


            /*
             * Draw traversal performs several drawing steps which must be executed
             * in the appropriate order:
             *
             *      1. Draw the background
             *      2. If necessary, save the canvas' layers to prepare for fading
             *      3. Draw view's content
             *      4. Draw children
             *      5. If necessary, draw the fading edges and restore layers
             *      6. Draw decorations (scrollbars for instance)
             */

            // Step 1, draw the background, if needed
            int saveCount;

            if (!dirtyOpaque) {
                //绘制当前视图的背景
                final Drawable background = mBGDrawable;
                if (background != null) {
                    final int scrollX = mScrollX;
                    final int scrollY = mScrollY;

                    if (mBackgroundSizeChanged) {
                        background.setBounds(0, 0,  mRight - mLeft, mBottom - mTop);
                        mBackgroundSizeChanged = false;
                    }

                    if ((scrollX | scrollY) == 0) {
                        background.draw(canvas);
                    } else {
                        canvas.translate(scrollX, scrollY);
                        background.draw(canvas);
                        canvas.translate(-scrollX, -scrollY);
                    }
                }
            }

            //检查是否可以跳过第2步和第5步，也就是绘制变量，FADING_EDGE_HORIZONTAL == 1表示处于水平
            //滑动状态，则需要绘制水平边框渐变效果，FADING_EDGE_VERTICAL == 1表示处于垂直滑动状态，则
            //需要绘制垂直边框渐变效果。
            // skip step 2 & 5 if possible (common case)
            final int viewFlags = mViewFlags;
            boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
            boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
            if (!verticalEdges && !horizontalEdges) {

                //窗口内容不透明才开始绘制，透明的时候就无需绘制了
                // Step 3, draw the content
                if (!dirtyOpaque) onDraw(canvas);

                // Step 4, draw the children
                dispatchDraw(canvas);

                // Step 6, draw decorations (scrollbars)
                onDrawScrollBars(canvas);

                // we're done...
                return;
            }

            /*
             * Here we do the full fledged routine...
             * (this is an uncommon case where speed matters less,
             * this is why we repeat some of the tests that have been
             * done above)
             */
            //检查失修需要保存参数canvas所描述的一块画布的堆栈状态，并且创建额外的图层来绘制当前视图
            //在滑动时的边框渐变效果
            boolean drawTop = false;
            boolean drawBottom = false;
            boolean drawLeft = false;
            boolean drawRight = false;

            float topFadeStrength = 0.0f;
            float bottomFadeStrength = 0.0f;
            float leftFadeStrength = 0.0f;
            float rightFadeStrength = 0.0f;

            // Step 2, save the canvas' layers
            int paddingLeft = mPaddingLeft;
            int paddingTop = mPaddingTop;

            final boolean offsetRequired = isPaddingOffsetRequired();
            if (offsetRequired) {
                paddingLeft += getLeftPaddingOffset();
                paddingTop += getTopPaddingOffset();
            }

            //表示当前视图可以用来绘制的内容区域，这个区域已经将内置的和扩展的内边距排除之外
            int left = mScrollX + paddingLeft;
            int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
            int top = mScrollY + paddingTop;
            int bottom = top + mBottom - mTop - mPaddingBottom - paddingTop;

            if (offsetRequired) {
                right += getRightPaddingOffset();
                bottom += getBottomPaddingOffset();
            }

            final ScrollabilityCache scrollabilityCache = mScrollCache;
            int length = scrollabilityCache.fadingEdgeLength;

            // clip the fade length if top and bottom fades overlap
            // overlapping fades produce odd-looking artifacts
            if (verticalEdges && (top + length > bottom - length)) {
                length = (bottom - top) / 2;
            }

            // also clip horizontal fades if necessary
            if (horizontalEdges && (left + length > right - length)) {
                length = (right - left) / 2;
            }

            if (verticalEdges) {
                topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
                drawTop = topFadeStrength >= 0.0f;
                bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
                drawBottom = bottomFadeStrength >= 0.0f;
            }

            if (horizontalEdges) {
                leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
                drawLeft = leftFadeStrength >= 0.0f;
                rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
                drawRight = rightFadeStrength >= 0.0f;
            }

            saveCount = canvas.getSaveCount();

            int solidColor = getSolidColor();
            if (solidColor == 0) {
                final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

                if (drawTop) {
                    canvas.saveLayer(left, top, right, top + length, null, flags);
                }

                if (drawBottom) {
                    canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
                }

                if (drawLeft) {
                    canvas.saveLayer(left, top, left + length, bottom, null, flags);
                }

                if (drawRight) {
                    canvas.saveLayer(right - length, top, right, bottom, null, flags);
                }
            } else {
                scrollabilityCache.setFadeColor(solidColor);
            }

            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            //绘制当前视图的上下左右边框的渐变效果
            // Step 5, draw the fade effect and restore layers
            final Paint p = scrollabilityCache.paint;
            final Matrix matrix = scrollabilityCache.matrix;
            final Shader fade = scrollabilityCache.shader;
            final float fadeHeight = scrollabilityCache.fadingEdgeLength;

            if (drawTop) {
                matrix.setScale(1, fadeHeight * topFadeStrength);
                matrix.postTranslate(left, top);
                fade.setLocalMatrix(matrix);
                canvas.drawRect(left, top, right, top + length, p);
            }

            if (drawBottom) {
                matrix.setScale(1, fadeHeight * bottomFadeStrength);
                matrix.postRotate(180);
                matrix.postTranslate(left, bottom);
                fade.setLocalMatrix(matrix);
                canvas.drawRect(left, bottom - length, right, bottom, p);
            }

            if (drawLeft) {
                matrix.setScale(1, fadeHeight * leftFadeStrength);
                matrix.postRotate(-90);
                matrix.postTranslate(left, top);
                fade.setLocalMatrix(matrix);
                canvas.drawRect(left, top, left + length, bottom, p);
            }

            if (drawRight) {
                matrix.setScale(1, fadeHeight * rightFadeStrength);
                matrix.postRotate(90);
                matrix.postTranslate(right, top);
                fade.setLocalMatrix(matrix);
                canvas.drawRect(right - length, top, right, bottom, p);
            }

            canvas.restoreToCount(saveCount);

            //绘制当前视图的滚动条
            // Step 6, draw decorations (scrollbars)
            onDrawScrollBars(canvas);
        }
}
```

该方法主要完成了以下事情：

1. 绘制当前视图的背景
2. 保存当前画布的状态，并且在当前画布创建额外的突出，以便接下来可以绘制视图在滑动时的边框渐变效果。
3. 绘制当前视图的内容
4. 绘制当前视图的子视图
5. 绘制当前视图在滑动时的边框渐变效果
6. 绘制当前视图的滚动条

**关键点 2：ViewGroup.dispatchDraw(Canvas canvas)**

dispatchDraw 用来循环绘制子 View 视图。

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {

    protected void dispatchDraw(Canvas canvas) {
            //当前视图的子视图个数
            final int count = mChildrenCount;
            final View[] children = mChildren;
            int flags = mGroupFlags;

            if ((flags & FLAG_RUN_ANIMATION) != 0 && canAnimate()) {
                final boolean cache = (mGroupFlags & FLAG_ANIMATION_CACHE) == FLAG_ANIMATION_CACHE;

                for (int i = 0; i < count; i++) {
                    final View child = children[i];
                    if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {
                        final LayoutParams params = child.getLayoutParams();
                        attachLayoutAnimationParameters(child, params, i, count);
                        bindLayoutAnimation(child);
                        if (cache) {
                            child.setDrawingCacheEnabled(true);
                            child.buildDrawingCache(true);
                        }
                    }
                }

                final LayoutAnimationController controller = mLayoutAnimationController;
                if (controller.willOverlap()) {
                    mGroupFlags |= FLAG_OPTIMIZE_INVALIDATE;
                }

                controller.start();

                //检查是否需要显示动画，即FLAG_RUN_ANIMATION == 1
                mGroupFlags &= ~FLAG_RUN_ANIMATION;
                mGroupFlags &= ~FLAG_ANIMATION_DONE;

                if (cache) {
                    mGroupFlags |= FLAG_CHILDREN_DRAWN_WITH_CACHE;
                }

                //通知动画监听者动画开始显示了
                if (mAnimationListener != null) {
                    mAnimationListener.onAnimationStart(controller.getAnimation());
                }
            }

            int saveCount = 0;
            //如果CLIP_TO_PADDING_MASK != 1，则说明参数canvas描述的是画布的剪裁区域，该剪裁区域不包含当前视图组的内边距
            final boolean clipToPadding = (flags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK;
            if (clipToPadding) {
                saveCount = canvas.save();
                //裁剪画布
                canvas.clipRect(mScrollX + mPaddingLeft, mScrollY + mPaddingTop,
                        mScrollX + mRight - mLeft - mPaddingRight,
                        mScrollY + mBottom - mTop - mPaddingBottom);

            }

            // We will draw our child's animation, let's reset the flag
            mPrivateFlags &= ~DRAW_ANIMATION;
            mGroupFlags &= ~FLAG_INVALIDATE_REQUIRED;

            boolean more = false;
            final long drawingTime = getDrawingTime();

            //如果FLAG_USE_CHILD_DRAWING_ORDER == 0，则说明子视图按照它们在children数组里顺序进行绘制
            //否则需要调用getChildDrawingOrder来判断绘制顺序
            if ((flags & FLAG_USE_CHILD_DRAWING_ORDER) == 0) {
                for (int i = 0; i < count; i++) {
                    final View child = children[i];
                    //如果子视图可见，则开始绘制子视图
                    if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                        more |= drawChild(canvas, child, drawingTime);
                    }
                }
            } else {
                for (int i = 0; i < count; i++) {
                    final View child = children[getChildDrawingOrder(count, i)];
                    if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                        more |= drawChild(canvas, child, drawingTime);
                    }
                }
            }

            //mDisappearingChildren用来保存哪些正在消失的子视图，正在消失的子视图也是需要绘制的
            // Draw any disappearing views that have animations
            if (mDisappearingChildren != null) {
                final ArrayList<View> disappearingChildren = mDisappearingChildren;
                final int disappearingCount = disappearingChildren.size() - 1;
                // Go backwards -- we may delete as animations finish
                for (int i = disappearingCount; i >= 0; i--) {
                    final View child = disappearingChildren.get(i);
                    more |= drawChild(canvas, child, drawingTime);
                }
            }

            if (clipToPadding) {
                canvas.restoreToCount(saveCount);
            }

            // mGroupFlags might have been updated by drawChild()
            flags = mGroupFlags;

            //如果FLAG_INVALIDATE_REQUIRED == 1，则说明需要进行重新绘制
            if ((flags & FLAG_INVALIDATE_REQUIRED) == FLAG_INVALIDATE_REQUIRED) {
                invalidate();
            }

            //通知动画监听者，动画已经结束
            if ((flags & FLAG_ANIMATION_DONE) == 0 && (flags & FLAG_NOTIFY_ANIMATION_LISTENER) == 0 &&
                    mLayoutAnimationController.isDone() && !more) {
                // We want to erase the drawing cache and notify the listener after the
                // next frame is drawn because one extra invalidate() is caused by
                // drawChild() after the animation is over
                mGroupFlags |= FLAG_NOTIFY_ANIMATION_LISTENER;
                final Runnable end = new Runnable() {
                   public void run() {
                       notifyAnimationListener();
                   }
                };
                post(end);
            }
        }
}
```

dispatchDraw 用来循环绘制子 View 视图，它主要做了以下事情：

1. 检查是否需要显示动画，即 FLAG_RUN_ANIMATION == 1，则开始显示动画，并通知动画监听者动画已经开始。
2. 如果 FLAG_USE_CHILD_DRAWING_ORDER == 0，则说明子视图按照它们在 children 数组里顺序进行绘制否则需要调用 getChildDrawingOrder 来判断绘制顺序，最终调用 drawChild()来完成
   子视图的绘制。
3. 判断是否需要进行重绘以及通知动画监听者动画已经结束。

**关键点 3：ViewGroup.drawChild(Canvas canvas, View child, long drawingTime)**

ViewGroup.drawChild(Canvas canvas, View child, long drawingTime)用来完成子视图的绘制。

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {

    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
            //表示子视图child是否还在显示动画
            boolean more = false;

            //获取子视图的绘制区域以及标志位
            final int cl = child.mLeft;
            final int ct = child.mTop;
            final int cr = child.mRight;
            final int cb = child.mBottom;

            final int flags = mGroupFlags;

            if ((flags & FLAG_CLEAR_TRANSFORMATION) == FLAG_CLEAR_TRANSFORMATION) {
                if (mChildTransformation != null) {
                    mChildTransformation.clear();
                }
                mGroupFlags &= ~FLAG_CLEAR_TRANSFORMATION;
            }

            //获取子视图的变换矩阵transformToApply
            Transformation transformToApply = null;
            //获取子视图的动画
            final Animation a = child.getAnimation();
            boolean concatMatrix = false;

            if (a != null) {
                if (mInvalidateRegion == null) {
                    mInvalidateRegion = new RectF();
                }
                final RectF region = mInvalidateRegion;

                final boolean initialized = a.isInitialized();
                if (!initialized) {
                    a.initialize(cr - cl, cb - ct, getWidth(), getHeight());
                    a.initializeInvalidateRegion(0, 0, cr - cl, cb - ct);
                    child.onAnimationStart();
                }

                if (mChildTransformation == null) {
                    mChildTransformation = new Transformation();
                }
                //如果子视图需要播放动画，则调用getTransformation开始执行动画，如果动画还需要继续执行，则more == true，并且返回子视图的
                //变化矩阵mChildTransformation
                more = a.getTransformation(drawingTime, mChildTransformation);
                transformToApply = mChildTransformation;

                concatMatrix = a.willChangeTransformationMatrix();

                if (more) {
                    if (!a.willChangeBounds()) {
                        if ((flags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE)) ==
                                FLAG_OPTIMIZE_INVALIDATE) {
                            mGroupFlags |= FLAG_INVALIDATE_REQUIRED;
                        } else if ((flags & FLAG_INVALIDATE_REQUIRED) == 0) {
                            // The child need to draw an animation, potentially offscreen, so
                            // make sure we do not cancel invalidate requests
                            mPrivateFlags |= DRAW_ANIMATION;
                            invalidate(cl, ct, cr, cb);
                        }
                    } else {
                        a.getInvalidateRegion(0, 0, cr - cl, cb - ct, region, transformToApply);

                        // The child need to draw an animation, potentially offscreen, so
                        // make sure we do not cancel invalidate requests
                        mPrivateFlags |= DRAW_ANIMATION;

                        final int left = cl + (int) region.left;
                        final int top = ct + (int) region.top;
                        invalidate(left, top, left + (int) region.width(), top + (int) region.height());
                    }
                }
            }
            //如果FLAG_SUPPORT_STATIC_TRANSFORMATIONS == 1，调用getChildStaticTransformation()方法检查子视图是否被设置一个
            //变换矩阵，如果设置了，即hasTransform == true，则mChildTransformation就是子视图需要的变换矩阵
            else if ((flags & FLAG_SUPPORT_STATIC_TRANSFORMATIONS) ==
                    FLAG_SUPPORT_STATIC_TRANSFORMATIONS) {
                if (mChildTransformation == null) {
                    mChildTransformation = new Transformation();
                }
                final boolean hasTransform = getChildStaticTransformation(child, mChildTransformation);
                if (hasTransform) {
                    final int transformType = mChildTransformation.getTransformationType();
                    transformToApply = transformType != Transformation.TYPE_IDENTITY ?
                            mChildTransformation : null;
                    concatMatrix = (transformType & Transformation.TYPE_MATRIX) != 0;
                }
            }

            //设置mPrivateFlags的DRAWN标志位为1，标明它要开始绘制了。
            // Sets the flag as early as possible to allow draw() implementations
            // to call invalidate() successfully when doing animations
            child.mPrivateFlags |= DRAWN;

            if (!concatMatrix && canvas.quickReject(cl, ct, cr, cb, Canvas.EdgeType.BW) &&
                    (child.mPrivateFlags & DRAW_ANIMATION) == 0) {
                return more;
            }

            //调用computeScroll()计算子视图的滑动位置
            child.computeScroll();

            final int sx = child.mScrollX;
            final int sy = child.mScrollY;

            boolean scalingRequired = false;
            Bitmap cache = null;
            //如果FLAG_CHILDREN_DRAWN_WITH_CACHE或者FLAG_CHILDREN_DRAWN_WITH_CACHE为1，则表示它采用缓冲的方式进行
            //绘制，它将自己的UI缓冲在一个Bitmap里，可以调用getDrawingCache()方法来获得这个Bitmap。
            if ((flags & FLAG_CHILDREN_DRAWN_WITH_CACHE) == FLAG_CHILDREN_DRAWN_WITH_CACHE ||
                    (flags & FLAG_ALWAYS_DRAWN_WITH_CACHE) == FLAG_ALWAYS_DRAWN_WITH_CACHE) {
                cache = child.getDrawingCache(true);
                if (mAttachInfo != null) scalingRequired = mAttachInfo.mScalingRequired;
            }

            final boolean hasNoCache = cache == null;

            //设置子视图child的偏移、Alpha通道以及裁剪区域
            final int restoreTo = canvas.save();
            if (hasNoCache) {
                canvas.translate(cl - sx, ct - sy);
            } else {
                canvas.translate(cl, ct);
                if (scalingRequired) {
                    // mAttachInfo cannot be null, otherwise scalingRequired == false
                    final float scale = 1.0f / mAttachInfo.mApplicationScale;
                    canvas.scale(scale, scale);
                }
            }

            float alpha = 1.0f;

            if (transformToApply != null) {
                if (concatMatrix) {
                    int transX = 0;
                    int transY = 0;
                    if (hasNoCache) {
                        transX = -sx;
                        transY = -sy;
                    }
                    // Undo the scroll translation, apply the transformation matrix,
                    // then redo the scroll translate to get the correct result.
                    canvas.translate(-transX, -transY);
                    canvas.concat(transformToApply.getMatrix());
                    canvas.translate(transX, transY);
                    mGroupFlags |= FLAG_CLEAR_TRANSFORMATION;
                }

                alpha = transformToApply.getAlpha();
                if (alpha < 1.0f) {
                    mGroupFlags |= FLAG_CLEAR_TRANSFORMATION;
                }

                if (alpha < 1.0f && hasNoCache) {
                    final int multipliedAlpha = (int) (255 * alpha);
                    if (!child.onSetAlpha(multipliedAlpha)) {
                        canvas.saveLayerAlpha(sx, sy, sx + cr - cl, sy + cb - ct, multipliedAlpha,
                                Canvas.HAS_ALPHA_LAYER_SAVE_FLAG | Canvas.CLIP_TO_LAYER_SAVE_FLAG);
                    } else {
                        child.mPrivateFlags |= ALPHA_SET;
                    }
                }
            } else if ((child.mPrivateFlags & ALPHA_SET) == ALPHA_SET) {
                child.onSetAlpha(255);
            }

            //如果FLAG_CLIP_CHILDREN == 1，则需要设置子视图的裁剪区域
            if ((flags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                if (hasNoCache) {
                    canvas.clipRect(sx, sy, sx + (cr - cl), sy + (cb - ct));
                } else {
                    if (!scalingRequired) {
                        canvas.clipRect(0, 0, cr - cl, cb - ct);
                    } else {
                        canvas.clipRect(0, 0, cache.getWidth(), cache.getHeight());
                    }
                }
            }

            //绘制子视图的UI
            if (hasNoCache) {
                // Fast path for layouts with no backgrounds
                if ((child.mPrivateFlags & SKIP_DRAW) == SKIP_DRAW) {
                    if (ViewDebug.TRACE_HIERARCHY) {
                        ViewDebug.trace(this, ViewDebug.HierarchyTraceType.DRAW);
                    }
                    child.mPrivateFlags &= ~DIRTY_MASK;
                    child.dispatchDraw(canvas);
                } else {
                    child.draw(canvas);
                }
            } else {
                final Paint cachePaint = mCachePaint;
                if (alpha < 1.0f) {
                    cachePaint.setAlpha((int) (alpha * 255));
                    mGroupFlags |= FLAG_ALPHA_LOWER_THAN_ONE;
                } else if  ((flags & FLAG_ALPHA_LOWER_THAN_ONE) == FLAG_ALPHA_LOWER_THAN_ONE) {
                    cachePaint.setAlpha(255);
                    mGroupFlags &= ~FLAG_ALPHA_LOWER_THAN_ONE;
                }
                if (Config.DEBUG && ViewDebug.profileDrawing) {
                    EventLog.writeEvent(60003, hashCode());
                }
                canvas.drawBitmap(cache, 0.0f, 0.0f, cachePaint);
            }

            //恢复画布的堆栈状态，以便在绘制完当前子视图的UI后，可以继续绘制其他子视图的UI
            canvas.restoreToCount(restoreTo);

            if (a != null && !more) {
                child.onSetAlpha(255);
                finishAnimatingView(child, a);
            }

            return more;
        }
}
```

ViewGroup.drawChild(Canvas canvas, View child, long drawingTime)用来完成子视图的绘制，它主要完成了以下事情：

1 获取子视图的绘制区域以及标志位
2 获取子视图的变换矩阵 transformToApply，这个分两种情况：

- 如果子视图需要播放动画，则调用 getTransformation 开始执行动画，如果动画还需要继续执行，则 more == true，并且返回子视图的变化矩阵 mChildTransformation
- 如果 FLAG_SUPPORT_STATIC_TRANSFORMATIONS == 1，调用 getChildStaticTransformation()方法检查子视图是否被设置一个变换矩阵，如果设置了，即 hasTransform == true，则 mChildTransformation 就是子视图需要的变换矩阵

3 如果 FLAG_CHILDREN_DRAWN_WITH_CACHE 或者 FLAG_CHILDREN_DRAWN_WITH_CACHE 为 1，则表示它采用缓冲的方式进行绘制，它将自己的 UI 缓冲在一个 Bitmap 里，可以调用 getDrawingCache()方法来获得这个 Bitmap。
4 设置子视图 child 的偏移、Alpha 通道以及裁剪区域。

5 绘制子视图的 UI，这分为两种情况：

- 如果以非缓冲的方式来绘制，如果 SKIP_DRAW == 1，则说明需要跳过当前子视图而去绘制它自己的子视图，否则先绘制它的视图，再绘制它的子视图。绘制自身通过 draw()函数来
  完成，绘制它的子视图则通过 dispatchDraw()来完成的。
- 如果是以缓冲的方式来绘制，这种情况只需要将上一次的缓冲的 Bitmap 对象 cache 绘制到画布 canvas 上

6 恢复画布的堆栈状态，以便在绘制完当前子视图的 UI 后，可以继续绘制其他子视图的 UI。

**总结**

至此，Android 应用程序窗口的渲染流程就分析完了，我们再来总结一下。

1. 渲染 Android 应用视图的渲染流程，测量流程用来确定视图的大小、布局流程用来确定视图的位置、绘制流程最终将视图绘制在应用窗口上。
2. Android 应用程序窗口 UI 首先是使用 Skia 图形库 API 来绘制在一块画布上，实际地是绘制在这块画布里面的一个图形缓冲区中，这个图形缓冲区最终会被交给 SurfaceFlinger 服
   务，而 SurfaceFlinger 服务再使用 OpenGL 图形库 API 来将这个图形缓冲区渲染到硬件帧缓冲区中。

## 五 View 事件分发机制

在介绍 View 的事件分发机制之前，我们要先了解两个概念。

- MotionEvent：Android 中用来表示各种事件的对象，例如 ACTION_DOWN、ACTION_MOVE 等，我们还可以通过它获取事件发生的坐标，getX/getY 获取相对于当前 View 左上角的坐标，getRawX/getRawY 获取相对于屏幕左上角的坐标。
- TouchSlop：系统所能识别的最小滑动距离，通过 ViewConfiguration.get(context).getScaledTouchSlop()方法获取。

现在我们再来看看 View 里的事件分发机制，概括来说，可以用下面代码表示：

```java
public boolean dispatchTouchEvent(MotionEvent event){
    boolean consume = false;
    //父View决定是否拦截事件
    if(onInterceptTouchEvent(event)){
        //父View调用onTouchEvent(event)消费事件
        consume = onTouchEvent(event);
    }else{
        //调用子View的dispatchTouchEvent(event)方法继续分发事件
        consume = child.dispatchTouchEvent(event);
    }
    return consume;
}
```

我们再来具体看看各个场景中的事件分发。

### 5.1 Activity 的事件分发

当点击事件发生时，事件最先传递给 Activity，Activity 会首先将事件将诶所属的 Window 进行处理，即调用 superDispatchTouchEvent()方法。

通过观察 superDispatchTouchEvent()方法的调用链，我们可以发现事件的传递顺序：

- PhoneWinodw.superDispatchTouchEvent()
- DecorView.dispatchTouchEvent(event)
- ViewGroup.dispatchTouchEvent(event)

事件一层层传递到了 ViewGroup 里，关于 ViewGroup 对事件的处理，我们下面会说，如果 superDispatchTouchEvent()方法返回 false，即没有
处理该事件，则会继续调用 Activity 的 onTouchEvent(ev)方法来处理该事件。可见 Activity 的 onTouchEvent(ev)在事件处理的优先级是最低的。

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback {

        public boolean dispatchTouchEvent(MotionEvent ev) {
            if (ev.getAction() == MotionEvent.ACTION_DOWN) {
                onUserInteraction();
            }
            if (getWindow().superDispatchTouchEvent(ev)) {
                return true;
            }
            return onTouchEvent(ev);
        }
}
```

### 5.2 ViewGroup 的事件分发

ViewGroup 作为 View 容器，它需要考虑自己的子 View 是否处理了该事件，具体说来：

- 如果 ViewGroup 拦截了事件，即它的 onInterceptTouchEvent()返回 true，则该事件由 ViewGroup 处理，如果 ViewGroup 调用了 setOnTouchListener()则该接口的 onTouch()方法会被调用
  否则会调用 onTouchEvent()方法。
- 如果 ViewGroup 没有拦截事件，则该事件会传递给它的子 View，子 View 的 dispatchTouchEvent()会被调用，View.dispatchTouchEvent()的处理流程前面我们已经分析过。

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {

    public boolean dispatchTouchEvent(MotionEvent ev) {
            ...
            boolean handled = false;
            if (onFilterTouchEventForSecurity(ev)) {
                final int action = ev.getAction();
                final int actionMasked = action & MotionEvent.ACTION_MASK;

                //每当有ACTION_DOWN事件进来的时候，都重置成初始状态
                if (actionMasked == MotionEvent.ACTION_DOWN) {
                    // Throw away all previous state when starting a new touch gesture.
                    // The framework may have dropped the up or cancel event for the previous gesture
                    // due to an app switch, ANR, or some other state change.
                    cancelAndClearTouchTargets(ev);
                    resetTouchState();
                }

                // Check for interception.
                final boolean intercepted;
                //MotionEvent.ACTION_DOWN事件总是会被ViewGroup拦截
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || mFirstTouchTarget != null) {
                    //1. 判断是否允许ViewGroup拦截除了ACTION_DOWN以外的其他事件，通过requestDisallowInterceptTouchEvent()方法设置
                    //FLAG_DISALLOW_INTERCEPT标志位来完成的
                    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                    if (!disallowIntercept) {
                        //2. 通过onInterceptTouchEvent(ev)方法判断是否拦截该事件
                        intercepted = onInterceptTouchEvent(ev);
                        ev.setAction(action); // restore action in case it was changed
                    } else {
                        intercepted = false;
                    }
                } else {
                    //如果mFirstTouchTarget == null，即没有接受
                    intercepted = true;
                }

                //ViewGroup以链表的形式存储它的子View，mFirstTouchTarget表示链表中第一个
                //被点击的子View
                if (intercepted || mFirstTouchTarget != null) {
                    ev.setTargetAccessibilityFocus(false);
                }

                // Check for cancelation.
                final boolean canceled = resetCancelNextUpFlag(this)
                        || actionMasked == MotionEvent.ACTION_CANCEL;

                // Update list of touch targets for pointer down, if needed.
                final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
                TouchTarget newTouchTarget = null;
                boolean alreadyDispatchedToNewTouchTarget = false;
                if (!canceled && !intercepted) {

                    ...

                    if (actionMasked == MotionEvent.ACTION_DOWN
                            || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                            || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                        final int actionIndex = ev.getActionIndex(); // always 0 for down
                        final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                                : TouchTarget.ALL_POINTER_IDS;

                        // Clean up earlier touch targets for this pointer id in case they
                        // have become out of sync.
                        removePointersFromTouchTargets(idBitsToAssign);

                        final int childrenCount = mChildrenCount;
                        if (newTouchTarget == null && childrenCount != 0) {
                            final float x = ev.getX(actionIndex);
                            final float y = ev.getY(actionIndex);
                            // Find a child that can receive the event.
                            // Scan children from front to back.
                            final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                            final boolean customOrder = preorderedList == null
                                    && isChildrenDrawingOrderEnabled();

                            //3. 当ViewGroup不再拦截事件时，事件会向下分发给它的子View进行处理。
                            final View[] children = mChildren;
                            for (int i = childrenCount - 1; i >= 0; i--) {
                                final int childIndex = getAndVerifyPreorderedIndex(
                                        childrenCount, i, customOrder);
                                final View child = getAndVerifyPreorderedView(
                                        preorderedList, children, childIndex);

                                 //4. 判断子View是否能够接受点击事件，判断标准有两点：① 子View是否可以获取焦点
                                 //② 点击的坐标是否落在了子View的区域内
                                if (childWithAccessibilityFocus != null) {
                                    if (childWithAccessibilityFocus != child) {
                                        continue;
                                    }
                                    childWithAccessibilityFocus = null;
                                    i = childrenCount - 1;
                                }

                                if (!canViewReceivePointerEvents(child)
                                        || !isTransformedTouchPointInView(x, y, child, null)) {
                                    ev.setTargetAccessibilityFocus(false);
                                    continue;
                                }

                                newTouchTarget = getTouchTarget(child);
                                if (newTouchTarget != null) {
                                    // Child is already receiving touch within its bounds.
                                    // Give it the new pointer in addition to the ones it is handling.
                                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                                    break;
                                }

                                resetCancelNextUpFlag(child);
                                //5. dispatchTransformedTouchEvent()方法会去调用子View的dispatchTouchEvent()方法来处理事件
                                if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                    // Child wants to receive touch within its bounds.
                                    mLastTouchDownTime = ev.getDownTime();
                                    if (preorderedList != null) {
                                        // childIndex points into presorted list, find original index
                                        for (int j = 0; j < childrenCount; j++) {
                                            if (children[childIndex] == mChildren[j]) {
                                                mLastTouchDownIndex = j;
                                                break;
                                            }
                                        }
                                    } else {
                                        mLastTouchDownIndex = childIndex;
                                    }
                                    mLastTouchDownX = ev.getX();
                                    mLastTouchDownY = ev.getY();
                                    newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                    alreadyDispatchedToNewTouchTarget = true;
                                    break;
                                }

                                // The accessibility focus didn't handle the event, so clear
                                // the flag and do a normal dispatch to all children.
                                ev.setTargetAccessibilityFocus(false);
                            }
                            if (preorderedList != null) preorderedList.clear();
                        }

                        ...
                    }
                }
    、
            ...
            return handled;
        }
}
```

### 5.3 View 的事件分发

View 没有子元素，无法向下传递事件，它只能自己处理事件，所以 View 的事件传递比较简单。

如果外界设置了 OnTouchListener 且 OnTouchListener.onTouch(this, event)返回 true，则表示该方法消费了该事件，则 onTouchEvent(event)不再被调用。
可见 OnTouchListener 的优先级高于 onTouchEvent(event)，这样是为了便于外界处理事件。

```java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {

     public boolean dispatchTouchEvent(MotionEvent event) {
            ...
            if (onFilterTouchEventForSecurity(event)) {
                if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                    result = true;
                }
                //如果外界设置了OnTouchListener且OnTouchListener.onTouch(this, event)返回true，则
                //表示该方法消费了该事件，则onTouchEvent(event)不再被调用
                ListenerInfo li = mListenerInfo;
                //如果外界调用了setOnTouchListener()方法且
                if (li != null && li.mOnTouchListener != null
                        && (mViewFlags & ENABLED_MASK) == ENABLED
                        && li.mOnTouchListener.onTouch(this, event)) {
                    result = true;
                }

                if (!result && onTouchEvent(event)) {
                    result = true;
                }
            }
            ...
            return result;
        }
}
```

我们再来看看 View 里的 onTouchEvent(event)方法的处理。

```java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {

     public boolean onTouchEvent(MotionEvent event) {
            final float x = event.getX();
            final float y = event.getY();
            final int viewFlags = mViewFlags;
            final int action = event.getAction();

            //1. View的disable属性不会影响onTouchEvent()方法的返回值，哪怕View是disable的，只要
            //View的clickable或者longClickable为true，onTouchEvent()方法还是会返回true
            if ((viewFlags & ENABLED_MASK) == DISABLED) {
                if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                    setPressed(false);
                }
                // A disabled view that is clickable still consumes the touch
                // events, it just doesn't respond to them.
                return (((viewFlags & CLICKABLE) == CLICKABLE
                        || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                        || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
            }
            if (mTouchDelegate != null) {
                if (mTouchDelegate.onTouchEvent(event)) {
                    return true;
                }
            }
            //2. 只要clickable或者longClickable为true，onTouchEvent()方法就会消费这个事件
            if (((viewFlags & CLICKABLE) == CLICKABLE ||
                    (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                    (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
                switch (action) {
                    case MotionEvent.ACTION_UP:
                        boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                            // take focus if we don't have it already and we should in
                            // touch mode.
                            boolean focusTaken = false;
                            if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                                focusTaken = requestFocus();
                            }

                            if (prepressed) {
                                // The button is being released before we actually
                                // showed it as pressed.  Make it show the pressed
                                // state now (before scheduling the click) to ensure
                                // the user sees it.
                                setPressed(true, x, y);
                           }

                            if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                                // This is a tap, so remove the longpress check
                                removeLongPressCallback();

                                // Only perform take click actions if we were in the pressed state
                                if (!focusTaken) {
                                    // Use a Runnable and post this rather than calling
                                    // performClick directly. This lets other visual state
                                    // of the view update before click actions start.
                                    if (mPerformClick == null) {
                                        mPerformClick = new PerformClick();
                                    }
                                    //3. 如果View设置了OnClickListener，则performClick()会调用它的onClick方法
                                    if (!post(mPerformClick)) {
                                        performClick();
                                    }
                                }
                            }

                            if (mUnsetPressedState == null) {
                                mUnsetPressedState = new UnsetPressedState();
                            }

                            if (prepressed) {
                                postDelayed(mUnsetPressedState,
                                        ViewConfiguration.getPressedStateDuration());
                            } else if (!post(mUnsetPressedState)) {
                                // If the post failed, unpress right now
                                mUnsetPressedState.run();
                            }

                            removeTapCallback();
                        }
                        mIgnoreNextUpEvent = false;
                        break;

                    case MotionEvent.ACTION_DOWN:
                        ...
                        break;

                    case MotionEvent.ACTION_CANCEL:
                        ...
                        break;

                    case MotionEvent.ACTION_MOVE:
                        ...
                        break;
                }

                return true;
            }

            return false;
        }
}
```

关于 onTouchEvent(MotionEvent event)，有两点需要说明一下：

1. View 的 disable 属性不会影响 onTouchEvent()方法的返回值，哪怕 View 是 disable 的，只要
   View 的 clickable 或者 longClickable 为 true，onTouchEvent()方法还是会返回 true。
2. 只要 clickable 或者 longClickable 为 true，onTouchEvent()方法就会消费这个事件
3. 如果 View 设置了 OnClickListener，则 performClick()会调用它的 onClick 方法。

上面我们提到了 viewFlags 里的 CLICKABLE 与 LONG_CLICKABLE，也就是 xml 或者代码里可以设置的 clickable 与 longClickable，View 的 LONG_CLICKABLE 默认为
true，CLICKABLE 默认为 false，值得一提的是 setOnClickListener()方法和 setOnLongClickListener()会将这两个值设置为 true。

通过对源码的分析，我们已经掌握了各种场景下事件分发的规律，我们再来总结一下 View 事件分发的相关结论。

- 事件的传递是按照 Activity -> Window -> View 的顺序进行的
- 一般情况下，一个事件序列只能由一个 View 拦截并消耗，一旦一个 View 拦截了该事件，则该事件序列的后续事件都会交由该 View 来处理。
- ViewGroup 默认不拦截任何事件
- View 没有 onInterceptTouchEvent()方法，一但有点击事件传递给它，它的 ouTouchEvent()方法就会被调用。
