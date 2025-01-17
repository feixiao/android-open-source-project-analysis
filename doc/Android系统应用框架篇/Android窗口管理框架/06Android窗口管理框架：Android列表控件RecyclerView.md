# Android 窗口管理框架：Android 列表控件 RecyclerView

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

**文章目录**

- RecyclerView 创建流程
- RecyclerView 布局策略管理器 LayoutManager
- RecyclerView 视图描述者 ViewHolder
- RecyclerView 视图复用器 Recycler
- RecyclerView 视图适配器 Adapter
- RecyclerView 视图动画 ItemAnimator
- RecyclerView 视图分隔条 ItemDecoration

> A flexible view for providing a limited window into a large data set.

RecyclerView 继承于 ViewGroup，实现了 ScrollingView 与 NestedScrollingChild 接口，它是日常业务开发中使用频度非常高的一个组件，它被 Android 设计出来代替原来的 ListView 组件。RecyclerView 的的
解耦与设计是十分精妙的，应用了适配器、观察者等多种模式。

提到 RecyclerView，就不得不说一下早期使用的 AdapterView（ListView），这里做下它们的简单对比：

- Item 复用方面：RecyclerView 内置了 RecyclerViewPool、多级缓存、ViewHolder，而 AdapterView 需要手动添加 ViewHolder 且复用功能也没 RecyclerView 完善。
- 样式丰富方面：RecyclerView 通过支持水平、垂直和表格列表及其他更复杂形式，而 AdapterView 只支持具体某一种。
- 效果增强方面：RecyclerView 内置了 ItemDecoration 和 ItemAnimator，可以自定义绘制 itemView 之间的一些特殊 UI 或 item 项数据变化时的动画效果，而用 AdapterView 实现时采取的做法是
  将这些特殊 UI 作为 itemView 的一部分，设置可见不可见决定是否展现，且数据变化时的动画效果没有提供，实现较为繁琐。
- 代码内聚方面：RecyclerView 将功能密切相关的类写成内部类（例如：ViewHolder，Adapter），而 AdapterView 则没有。

我们再来看看 RecyclerView 的渲染流程，一般说来，列表展示涉及两个角色：列表的数据 DataSet 以及最终显示在界面上的 RecyclerView，这两者的转换流程如下所示：

RecyclerView 渲染流程图

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/recyclerview_structure.png"/>

Adapter 将数据 DataSet 翻译成 RecyclerView 可以理解的 ViewHolder，Recycler 负责对这些 ViewHolder 进行管理，LayoutManager 从 Recycler 获取这些 ViewHolder，然后在 RecyclerView 里对它们进行布局，在布局
的过程中还可以通过 ItemDecoration、ItemAnimator 为这些 ViewHolder 添加分隔条、转场动画等东西，让整个 RecyclerView 更加具有交互性。

接下来我们分别来看看每个角色的源码实现。

## 一 RecyclerView 创建流程

首先我们还是从一个简单的例子引入，如何实现一个简单的 RecyclerView 列表。

```java
RecyclerView recyclerView = findViewById(R.id.recycler_view);
RecyclerAdapter adapter = new RecyclerAdapter();
List<RecyclerModel> data = new ArrayList<>();
for(int i = 0; i < 30; i++){
    data.add(new RecyclerModel());
}
adapter.setData(data);
recyclerView.setLayoutManager(new LinearLayoutManager(this));
recyclerView.setAdapter(adapter);
```

我们可以从这段简短的代码提炼出三个关键点：

- RecyclerView()
- setLayoutManager()
- setAdapter()

我们首先来分析它的构造函数。

### 1.1 RecyclerView()

```java
public class RecyclerView extends ViewGroup implements ScrollingView, NestedScrollingChild {

     public RecyclerView(Context context, @Nullable AttributeSet attrs, int defStyle) {
            super(context, attrs, defStyle);
            ...
            //设置当前View可以滑动
            setScrollContainer(true);
            //设置当前View可以获取焦点
            setFocusableInTouchMode(true);

            //从ViewConfiguration获取和View相关的信息，例如最小滑动距离TouchSlop等
            final ViewConfiguration vc = ViewConfiguration.get(context);
            mTouchSlop = vc.getScaledTouchSlop();
            mMinFlingVelocity = vc.getScaledMinimumFlingVelocity();
            mMaxFlingVelocity = vc.getScaledMaximumFlingVelocity();
            setWillNotDraw(getOverScrollMode() == View.OVER_SCROLL_NEVER);

            //设置动画结束时的监听器，这里是实现了ItemAnimator.ItemAnimatorListener接口
            //的ItemAnimatorRestoreListener
            mItemAnimator.setListener(mItemAnimatorListener);

            //初始化AdapterHelper，它是一个工具类，用来对Adapter里的操作进行排队和处理
            initAdapterManager();
            //初始化ChildHelper，它是一个工具类，用来向RecyclerView移除和添加子View
            initChildrenHelper();
            // If not explicitly specified this view is important for accessibility.
            if (ViewCompat.getImportantForAccessibility(this)
                    == ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                ViewCompat.setImportantForAccessibility(this,
                        ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_YES);
            }
            mAccessibilityManager = (AccessibilityManager) getContext()
                    .getSystemService(Context.ACCESSIBILITY_SERVICE);
            setAccessibilityDelegateCompat(new RecyclerViewAccessibilityDelegate(this));
            // Create the layoutManager if specified.

            boolean nestedScrollingEnabled = true;

            //从xml文件里读取参数信息
            if (attrs != null) {
                ...
            } else {
                setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            }

             // Re-set whether nested scrolling is enabled so that it is set on all API levels
             setNestedScrollingEnabled(nestedScrollingEnabled);
    }
}
```

RecyclerView 的构造方法很简单，它主要初始化了两个重要的工具类：

- AdapterHelper：用来对 Adapter 里的操作进行排队和处理
- ChildHelper：用来向 RecyclerView 移除和添加子 View

我们接着来看看 setLayoutManager()方法。

### 1.2 setLayoutManager()

```java
public class RecyclerView extends ViewGroup implements ScrollingView, NestedScrollingChild {

    public void setLayoutManager(LayoutManager layout) {
            if (layout == mLayout) {
                return;
            }
            //停止滑动
            stopScroll();
            // TODO We should do this switch a dispatchLayout pass and animate children. There is a good
            // chance that LayoutManagers will re-use views.
            //处理一些清空的操作，准备好重新进行布局
            if (mLayout != null) {
                //停止所有动画
                if (mItemAnimator != null) {
                    mItemAnimator.endAnimations();
                }
                //移除当前LayoutManager里的View，并稍后在mRecycler里复用
                mLayout.removeAndRecycleAllViews(mRecycler);
                //移除Scrap View，Scrap View指的是那些暂时处理分离状态的View
                //后面可能还会重新使用
                mLayout.removeAndRecycleScrapInt(mRecycler);
                //清空mRecycler
                mRecycler.clear();

                if (mIsAttached) {
                    mLayout.dispatchDetachedFromWindow(this, mRecycler);
                }
                mLayout.setRecyclerView(null);
                mLayout = null;
            } else {
                mRecycler.clear();
            }
            //移除所有子View（包括处于hidden状态的View）
            mChildHelper.removeAllViewsUnfiltered();
            mLayout = layout;
            if (layout != null) {
                if (layout.mRecyclerView != null) {
                    throw new IllegalArgumentException("LayoutManager " + layout +
                            " is already attached to a RecyclerView: " + layout.mRecyclerView);
                }
                //从新给mLayout设置当前RecyclerView的引用
                mLayout.setRecyclerView(this);
                if (mIsAttached) {
                    //处理attach到Window
                    mLayout.dispatchAttachedToWindow(this);
                }
            }
            //更新View的缓存数目
            mRecycler.updateViewCacheSize();
            //重新布局
            requestLayout();
        }
}
```

前面我们已经提到 LayoutManager 只负责对 View 进行布局，而承担管理 View 责任的是 Recycler，Recycler 实现了 View 复用的功能，让列表变得流程。再回到这个方法，它主要做了两件事：

- 处理一些清空的操作，准备好重新进行布局。
- 重新赋值 mLayout，并给 mLayout 设置当前 RecyclerView 的引用，然后请求进行重新布局。

我们接着来看看 setAdapter()方法。

### 1.2 setAdapter()

```java
public class RecyclerView extends ViewGroup implements ScrollingView, NestedScrollingChild {

     public void setAdapter(Adapter adapter) {
            //停止当前的布局和滑动
            setLayoutFrozen(false);
            //设置Adapter，这里有两个boolean，第一个表示是否使用当前相同的ViewHolder
            //第二个表示是否移除和复用当前所有的View，当第一个boolean为false时，第二个
            //boolean会被忽略
            setAdapterInternal(adapter, false, true);
            //请求重新进行布局
            requestLayout();
        }

        private void setAdapterInternal(Adapter adapter, boolean compatibleWithPrevious,
                boolean removeAndRecycleViews) {

            //如果mAdapter不空，先释放掉原来数据和View
            if (mAdapter != null) {
                mAdapter.unregisterAdapterDataObserver(mObserver);
                mAdapter.onDetachedFromRecyclerView(this);
            }

            //停止动画以及做一些清空操作，准备重新设置Adapter
            if (!compatibleWithPrevious || removeAndRecycleViews) {
                //停止所有动画
                if (mItemAnimator != null) {
                    mItemAnimator.endAnimations();
                }

                if (mLayout != null) {
                    mLayout.removeAndRecycleAllViews(mRecycler);
                    mLayout.removeAndRecycleScrapInt(mRecycler);
                }
                mRecycler.clear();
            }
            mAdapterHelper.reset();
            final Adapter oldAdapter = mAdapter;

            //重新设置新的Adapter
            mAdapter = adapter;

            //重新注册数据监测DataObserver以及attach到RecyclerView
            if (adapter != null) {
                adapter.registerAdapterDataObserver(mObserver);
                adapter.onAttachedToRecyclerView(this);
            }
            if (mLayout != null) {
                mLayout.onAdapterChanged(oldAdapter, mAdapter);
            }
            //触发Recycler的onAdapterChanged()方法。
            mRecycler.onAdapterChanged(oldAdapter, mAdapter, compatibleWithPrevious);
            mState.mStructureChanged = true;
            //将所有视图标记为无效，因为要开始进行数据刷新和重绘
            markKnownViewsInvalid();
        }
}
```

这个方法的逻辑和 setLayoutManager()有点相似，我们知道当我们设置了 Adapter，列表就可以显示出来了。它也做了两件事情：

- 对以前的 Adapter（如果有的话）做一些清空操作，停止滑动和动画，准备好重新设置 Adapter。
- 重新设置 Adapter，重新注册 AdapterDataObserver 以及关联到 RecyclerView，并触发 Recycler 的 onAdapterChanged()方法，通知
  Adapter 已经发生改变，最终触发重新布局的操作。

以上就是这三个方法的全部内容，通过对三个方法的源码分析，我们抓住了一些关键类类以及他们的关键函数，具体说来：

LayoutManager

- removeAndRecycleAllViews(mRecycler); 移除当前 LayoutManager 里的 View，并稍后在 mRecycler 里复用
- removeAndRecycleScrapInt(mRecycler); 移除 Scrap View，Scrap View 指的是那些暂时处理分离状态的 View 后面可能还会重新使用

Adapter

- unregisterAdapterDataObserver(mObserver); 取消注册 RecyclerViewDataObserver
- onDetachedFromRecyclerView(this); 取消关联到 RecyclerView
- registerAdapterDataObserver(mObserver); 重新注册 RecyclerViewDataObserver
- onAttachedToRecyclerView(this); 重新关联到 RecyclerView

Recycler

- onAdapterChanged(oldAdapter, mAdapter, compatibleWithPrevious); 通知 Adapter 已经发生改变

下面我们就带着对这些方法的疑问逐个去分类每个关键类的作用。

## 二 RecyclerView 布局策略管理器 LayoutManager

> LayoutManager 是一个抽象类。它主要用来测量和布局子 View，滚动子 View 以及决定何时回收用户不可见的子 View。

它有三个子类：

- LinearLayoutManager：线性布局，分为水平方向和竖直方向两种。
- GridLayoutManager：网络布局，分为水平方向和竖直方向两种。
- StaggeredGridLayoutManager：瀑布流布局，分为水平方向和竖直方向两种。

上面提到 LayoutManager 主要用来测量和布局子 View，滚动子 View 以及决定何时回收用户不可见的子 View。那么处理这些内容的关键方法就是我们
重点关注的对象，具体说来：

- onLayoutChildren(Recycler recycler, State state)：对子 View 进行布局。
- scrollHorizontallyBy(int dx, Recycler recycler, State state) :处理水平方向的滑动。
- scrollVerticallyBy(int dy, Recycler recycler, State state)：处理竖直方向的滑动。

关于滑动需要说明一下，RecyclerView 已经处理了和触摸相关的事件，当 RecyclerView 上下滑动的时候，滑动的偏移量会传入这两个方法，就是上面的 dx/dy，这个时候
LayoutManager 的子类完成了以下三件事情：

- 将子 View 移动适当的位置
- 处理子 View 移动后的添加/删除逻辑
- 返回移动后的实际距离，RecyclerView 会根据这个距离判断是否触屏到边界。

好，理解了哪些是关键方法，我们就来分析这三个字类对这些方法的实现。

### 2.1 LinearLayoutManager

#### onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state)

```java
public class LinearLayoutManager extends RecyclerView.LayoutManager implements
        ItemTouchHelper.ViewDropHandler, RecyclerView.SmoothScroller.ScrollVectorProvider {

        @Override
        public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {

            if (DEBUG) {
                Log.d(TAG, "is pre layout:" + state.isPreLayout());
            }
            if (mPendingSavedState != null || mPendingScrollPosition != NO_POSITION) {
                if (state.getItemCount() == 0) {
                    removeAndRecycleAllViews(recycler);
                    return;
                }
            }
            if (mPendingSavedState != null && mPendingSavedState.hasValidAnchor()) {
                mPendingScrollPosition = mPendingSavedState.mAnchorPosition;
            }

            ensureLayoutState();
            mLayoutState.mRecycle = false;
            // resolve layout direction
            resolveShouldLayoutReverse();

            final View focused = getFocusedChild();
            if (!mAnchorInfo.mValid || mPendingScrollPosition != NO_POSITION
                    || mPendingSavedState != null) {
                mAnchorInfo.reset();
                mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
                // calculate anchor position and coordinate
                updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
                mAnchorInfo.mValid = true;
            } else if (focused != null && (mOrientationHelper.getDecoratedStart(focused)
                            >= mOrientationHelper.getEndAfterPadding()
                    || mOrientationHelper.getDecoratedEnd(focused)
                    <= mOrientationHelper.getStartAfterPadding())) {
                // This case relates to when the anchor child is the focused view and due to layout
                // shrinking the focused view fell outside the viewport, e.g. when soft keyboard shows
                // up after tapping an EditText which shrinks RV causing the focused view (The tapped
                // EditText which is the anchor child) to get kicked out of the screen. Will update the
                // anchor coordinate in order to make sure that the focused view is laid out. Otherwise,
                // the available space in layoutState will be calculated as negative preventing the
                // focused view from being laid out in fill.
                // Note that we won't update the anchor position between layout passes (refer to
                // TestResizingRelayoutWithAutoMeasure), which happens if we were to call
                // updateAnchorInfoForLayout for an anchor that's not the focused view (e.g. a reference
                // child which can change between layout passes).
                mAnchorInfo.assignFromViewAndKeepVisibleRect(focused);
            }
            if (DEBUG) {
                Log.d(TAG, "Anchor info:" + mAnchorInfo);
            }

            // LLM may decide to layout items for "extra" pixels to account for scrolling target,
            // caching or predictive animations.
            int extraForStart;
            int extraForEnd;
            final int extra = getExtraLayoutSpace(state);
            // If the previous scroll delta was less than zero, the extra space should be laid out
            // at the start. Otherwise, it should be at the end.
            if (mLayoutState.mLastScrollDelta >= 0) {
                extraForEnd = extra;
                extraForStart = 0;
            } else {
                extraForStart = extra;
                extraForEnd = 0;
            }
            extraForStart += mOrientationHelper.getStartAfterPadding();
            extraForEnd += mOrientationHelper.getEndPadding();
            if (state.isPreLayout() && mPendingScrollPosition != NO_POSITION
                    && mPendingScrollPositionOffset != INVALID_OFFSET) {
                // if the child is visible and we are going to move it around, we should layout
                // extra items in the opposite direction to make sure new items animate nicely
                // instead of just fading in
                final View existing = findViewByPosition(mPendingScrollPosition);
                if (existing != null) {
                    final int current;
                    final int upcomingOffset;
                    if (mShouldReverseLayout) {
                        current = mOrientationHelper.getEndAfterPadding()
                                - mOrientationHelper.getDecoratedEnd(existing);
                        upcomingOffset = current - mPendingScrollPositionOffset;
                    } else {
                        current = mOrientationHelper.getDecoratedStart(existing)
                                - mOrientationHelper.getStartAfterPadding();
                        upcomingOffset = mPendingScrollPositionOffset - current;
                    }
                    if (upcomingOffset > 0) {
                        extraForStart += upcomingOffset;
                    } else {
                        extraForEnd -= upcomingOffset;
                    }
                }
            }
            int startOffset;
            int endOffset;
            final int firstLayoutDirection;
            if (mAnchorInfo.mLayoutFromEnd) {
                firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_TAIL
                        : LayoutState.ITEM_DIRECTION_HEAD;
            } else {
                firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD
                        : LayoutState.ITEM_DIRECTION_TAIL;
            }

            onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
            detachAndScrapAttachedViews(recycler);
            mLayoutState.mInfinite = resolveIsInfinite();
            mLayoutState.mIsPreLayout = state.isPreLayout();
            if (mAnchorInfo.mLayoutFromEnd) {
                // fill towards start
                updateLayoutStateToFillStart(mAnchorInfo);
                mLayoutState.mExtra = extraForStart;
                fill(recycler, mLayoutState, state, false);
                startOffset = mLayoutState.mOffset;
                final int firstElement = mLayoutState.mCurrentPosition;
                if (mLayoutState.mAvailable > 0) {
                    extraForEnd += mLayoutState.mAvailable;
                }
                // fill towards end
                updateLayoutStateToFillEnd(mAnchorInfo);
                mLayoutState.mExtra = extraForEnd;
                mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
                fill(recycler, mLayoutState, state, false);
                endOffset = mLayoutState.mOffset;

                if (mLayoutState.mAvailable > 0) {
                    // end could not consume all. add more items towards start
                    extraForStart = mLayoutState.mAvailable;
                    updateLayoutStateToFillStart(firstElement, startOffset);
                    mLayoutState.mExtra = extraForStart;
                    fill(recycler, mLayoutState, state, false);
                    startOffset = mLayoutState.mOffset;
                }
            } else {
                // fill towards end
                updateLayoutStateToFillEnd(mAnchorInfo);
                mLayoutState.mExtra = extraForEnd;
                fill(recycler, mLayoutState, state, false);
                endOffset = mLayoutState.mOffset;
                final int lastElement = mLayoutState.mCurrentPosition;
                if (mLayoutState.mAvailable > 0) {
                    extraForStart += mLayoutState.mAvailable;
                }
                // fill towards start
                updateLayoutStateToFillStart(mAnchorInfo);
                mLayoutState.mExtra = extraForStart;
                mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
                fill(recycler, mLayoutState, state, false);
                startOffset = mLayoutState.mOffset;

                if (mLayoutState.mAvailable > 0) {
                    extraForEnd = mLayoutState.mAvailable;
                    // start could not consume all it should. add more items towards end
                    updateLayoutStateToFillEnd(lastElement, endOffset);
                    mLayoutState.mExtra = extraForEnd;
                    fill(recycler, mLayoutState, state, false);
                    endOffset = mLayoutState.mOffset;
                }
            }

            // changes may cause gaps on the UI, try to fix them.
            // TODO we can probably avoid this if neither stackFromEnd/reverseLayout/RTL values have
            // changed
            if (getChildCount() > 0) {
                // because layout from end may be changed by scroll to position
                // we re-calculate it.
                // find which side we should check for gaps.
                if (mShouldReverseLayout ^ mStackFromEnd) {
                    int fixOffset = fixLayoutEndGap(endOffset, recycler, state, true);
                    startOffset += fixOffset;
                    endOffset += fixOffset;
                    fixOffset = fixLayoutStartGap(startOffset, recycler, state, false);
                    startOffset += fixOffset;
                    endOffset += fixOffset;
                } else {
                    int fixOffset = fixLayoutStartGap(startOffset, recycler, state, true);
                    startOffset += fixOffset;
                    endOffset += fixOffset;
                    fixOffset = fixLayoutEndGap(endOffset, recycler, state, false);
                    startOffset += fixOffset;
                    endOffset += fixOffset;
                }
            }
            layoutForPredictiveAnimations(recycler, state, startOffset, endOffset);
            if (!state.isPreLayout()) {
                mOrientationHelper.onLayoutComplete();
            } else {
                mAnchorInfo.reset();
            }
            mLastStackFromEnd = mStackFromEnd;
            if (DEBUG) {
                validateChildOrder();
            }
        }
}
```

          // layout algorithm:
            // 1) by checking children and other variables, find an anchor coordinate and an anchor
            //  item position.
            // 2) fill towards start, stacking from bottom
            // 3) fill towards end, stacking from top
            // 4) scroll to fulfill requirements like stack from bottom.
            // create layout state


1. 从子 View 中查找出一个锚点坐标和锚点位置。
2. 从底部开始堆积，直到填充到首端。
3. 从顶部开始堆积，知道填充到尾部。
4.

#### scrollHorizontallyBy()/scrollVerticallyBy()

RecyclerView 在滚动的时候主要涉及两个问题：

- 旧 View 的回收与新 View 的添加
- 其他 View 重新布局，偏移到合适的位置。

整个流程的序列图如下所示：

注意观察图中标记红色和绿色的方法，它分别完成了上面说的两件事。

- recycleByLayoutState()：回收不再显示的 View。
- layoutChunk()：其他 View 重新布局，偏移到合适的位置。。

这两个方法都在 fill()方法里被调用，它是 LinearLayoutManager 最为关键的方法，如下所示：

```java
public class LinearLayoutManager extends RecyclerView.LayoutManager implements
        ItemTouchHelper.ViewDropHandler, RecyclerView.SmoothScroller.ScrollVectorProvider {

      int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
                RecyclerView.State state, boolean stopOnFocusable) {
            // max offset we should set is mFastScroll + available
            final int start = layoutState.mAvailable;
            if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
                // TODO ugly bug fix. should not happen
                if (layoutState.mAvailable < 0) {
                    layoutState.mScrollingOffset += layoutState.mAvailable;
                }
                //回收不再显示的子View
                recycleByLayoutState(recycler, layoutState);
            }
            int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
            LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
            while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
                layoutChunkResult.resetInternal();
                if (VERBOSE_TRACING) {
                    TraceCompat.beginSection("LLM LayoutChunk");
                }
                layoutChunk(recycler, state, layoutState, layoutChunkResult);
                if (VERBOSE_TRACING) {
                    TraceCompat.endSection();
                }
                if (layoutChunkResult.mFinished) {
                    break;
                }
                layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
                /**
                 * Consume the available space if:
                 * * layoutChunk did not request to be ignored
                 * * OR we are laying out scrap children
                 * * OR we are not doing pre-layout
                 */
                if (!layoutChunkResult.mIgnoreConsumed || mLayoutState.mScrapList != null
                        || !state.isPreLayout()) {
                    layoutState.mAvailable -= layoutChunkResult.mConsumed;
                    // we keep a separate remaining space because mAvailable is important for recycling
                    remainingSpace -= layoutChunkResult.mConsumed;
                }

                if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
                    layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
                    if (layoutState.mAvailable < 0) {
                        layoutState.mScrollingOffset += layoutState.mAvailable;
                    }
                    recycleByLayoutState(recycler, layoutState);
                }
                if (stopOnFocusable && layoutChunkResult.mFocusable) {
                    break;
                }
            }
            if (DEBUG) {
                validateChildOrder();
            }
            return start - layoutState.mAvailable;
        }
}
```

这里面有个 LayoutState 类，它保存了与布局相关的状态，它里面相关变量的含义如下所示：

- boolean mRecycle = true; 子 View 是否可以被回收
- nt mOffset; 布局从何处开始
- int mAvailable; 滚动完成后，空出多少空间需要被填充
- int mCurrentPosition; Adapter 上当前的位置
- int mItemDirection; Adapter 遍历数据的方向，有 ITEM_DIRECTION_HEAD 与 ITEM_DIRECTION_TAIL 两种。
- int mLayoutDirection; 布局填充的方向，有 LAYOUT_START 与 LAYOUT_END 两种。
- int mScrollingOffset; 添加一个新 View 后，还可以滑动多少空间。
- boolean mInfinite; 无限滚动，没有新 View 添加限制。

### 2.2 GridLayoutManager

### 2.3 StaggeredGridLayoutManager

## 三 RecyclerView 视图描述者 ViewHolder

> A ViewHolder describes an item view and metadata about its place within the RecyclerView.

ViewHolder 正如它的名字那样，它是一个 Item View 视图容器。

## 四 RecyclerView 视图复用器 Recycler

> A Recycler is responsible for managing scrapped or detached item views for reuse.

Recycler 实现了 Item View 的回收与复用，也就是说它是用来做缓存的，我们首先来关注下 Recycler 里面几个缓存列表：

> Scrap View 指的是那些已经滚动出可视范围，但还没有 detach 掉的 Item View。

- final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>(); 存放已经滚动出可视范围，但还没有 detach 掉的 ViewHolder。
- ArrayList<ViewHolder> mChangedScrap = null; 存放那些被标记为 Scrap View 的 ViewHolder
- final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>(); 存放缓存的 ViewHolder
- private final List<ViewHolder> mUnmodifiableAttachedScrap = Collections.unmodifiableList(mAttachedScrap); 一个不可变列表，用来存放 mAttachedScrap。

此外，Recycler 里还有个缓存池的概念，也就是 RecycledViewPool，它可以让我们在多个 RecyclerView 之间复用 View，大大提高了渲染效率。使用的方式也是创建一个 RecycledViewPool 实例，然后
调用 RecyclerView.setRecyclerPool()方法来进行设置即可，当然 RecyclerView 默认会创建一个 RecycledViewPool。

## 五 RecyclerView 视图适配器 Adapter

## 六 RecyclerView 视图动画 ItemAnimator

## 七 RecyclerView 视图分隔条 ItemDecoration
