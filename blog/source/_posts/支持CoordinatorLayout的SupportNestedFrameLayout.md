---
title: 支持CoordinatorLayout的SupportNestedFrameLayout
date: 2017-10-10 20:11:28
categories: Android
keywords: CoordinatorLayout, NestedScroll, SupportNestedFrameLayout
description: 支持CoordinatorLayout的SupportNestedFrameLayout
tags: [Android, CoordinatorLayout, NestedScroll]
---
### 遇到的问题
最近在需求中遇到了一个问题，先贴出大致布局代码：

```
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"   
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <android.support.design.widget.AppBarLayout
        android:id="@+id/appBarLayout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:fitsSystemWindows="true">
        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="?attr/colorPrimary"
            app:popupTheme="@style/ThemeOverlay.AppCompat.Light"
            app:layout_scrollFlags="scroll|snap" />
        <android.support.design.widget.TabLayout
            android:id="@+id/tl_vp_tab"
            android:layout_width="match_parent"
            android:layout_height="54dp"
            app:tabMode="scrollable"
            app:tabPaddingStart="11dp"
            app:tabPaddingEnd="11dp"
            app:tabMinWidth="0dp"
            app:tabIndicatorHeight="3dp"
            app:tabIndicatorColor="#FF22A9A0"
            app:tabSelectedTextColor="#FF222222"
            app:tabTextAppearance="@style/CourseTabTextAppearance"/>
    </android.support.design.widget.AppBarLayout>
    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_behavior="@string/appbar_scrolling_view_behavior">
        <android.support.v4.view.ViewPager
            android:id="@+id/vp_content"
            android:layout_width="match_parent"
            android:layout_height="match_parent"/>
    </FrameLayout>
</android.support.design.widget.CoordinatorLayout>
```
`viewPager`里是`Fragment`，`Fragment`的布局就是一个`Recyclerview`。到这里的布局都与`CoordinatorLayout`协作很好，但是，有点小小的需求就是，当数据为空的**空提示布局**和网络错误的**网络错误布局**出现的时候，并不能与`CoordinatorLayout`协作。换句话说，就是当点击并试图拖动上拉的区域是空布局或者是错误布局时，无法拖动，也无法将`AppbarLayout`隐藏。

**原因是啥？** 因为空布局不像`recyclerview`一样实现`Nested`相关接口，所以无法和`CoordinatorLayout`协作啊。

**解决方法是啥？**写个实现`Nested`相关接口的布局，将正常态，空态，异常态三种状态包裹进来不就完啦！

于是，话题展开：如何写一个支持`CoordinatorLayout`和Nested的`FrameLayout`。

### 需求分析
这个中间层的`FrameLayout`要做的目标比较明确：

* 1. 当子view实现Nested相关接口时，直接把子view的各种事件传到上层。
* 2. 当子view并没有实现相关接口时，自己处理。

**分析结果：** 这个`FrameLayout`需要继承`NestedScrollingChild`, `NestedScrollingParent`这俩接口。

### 写前准备
**先来说说Nested相关的一些知识：**

`NestedScrolling` 提供了一套父 View 和子 View 滑动交互机制。要完成这样的交互，父 View 需要实现 `NestedScrollingParent` 接口，而子 View 需要实现 `NestedScrollingChild` 接口。

```
NestedScrollingChild (接口) 
NestedScrollingChildHelper

NestedScrollingParent (接口) 
NestedScrollingParentHelper
```

像`recyclerview`就实现了`NestedScrollingChild`接口，`CoordinatorLayout` 类实现了`NestedScrollingParent`接口。

**`NestedScrollingChild`类**

```
//开始、停止嵌套滚动
public boolean startNestedScroll(int axes); public void stopNestedScroll();
//触摸滚动相关
public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow);
public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow);
//惯性滚动相关 
public boolean dispatchNestedPreFling(float velocityX, float velocityY);
public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed);
```

首先来说 `NestedScrollingChild`。如果你有一个可以滑动的 View，需要被用来作为嵌入滑动的子 View，就必须实现本接口。在此 View 中，包含一个 `NestedScrollingChildHelper` 辅助类。`NestedScrollingChild` 接口的实现，基本上就是调用本 Helper 类的对应的函数即可，因为 Helper 类中已经实现好了 Child 和 Parent 交互的逻辑。原来的 View 的处理 Touch 事件，并实现滑动的逻辑大体上不需要改变。

需要做的就是，如果要准备开始滑动了，child需要告诉 Parent:要准备进入滑动状态了。调用 `startNestedScroll()`。 
child在滑动之前，先问一下 Parent 是否需要滑动，也就是调用 `dispatchNestedPreScroll()`。 
如果 Parent 滑动了一定距离，child 需要重新计算一下 Parent 滑动后剩下给child的滑动距离余量。 
然后，child进行余下的滑动。 
最后，如果滑动距离还有剩余，child 就再问一下，Parent 是否需要在继续滑动你剩下的距离，也就是调用 `dispatchNestedScroll()`。

**`NestedScrollingParent`类**

```
/当开启、停止嵌套滚动时被调用
public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);
public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes);
public void onStopNestedScroll(View target);
//当触摸嵌套滚动时被调用
public void onNestedPreScroll(View target, int dx, int dy, int[] consumed);
public void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed);
//当惯性嵌套滚动时被调用
public boolean onNestedPreFling(View target, float velocityX, float velocityY);
public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed);
```
作为一个可以嵌入 `NestedScrollingChild` 的父 View，需要实现 `NestedScrollingParent`，这个接口方法和 `NestedScrollingChild` 大致有一一对应的关系。同样，也有一个 `NestedScrollingParentHelper` 辅助类来帮助你实现和 `Child` 交互的逻辑。滑动动作是 `Child` 主动发起，`Parent` 就收滑动回调并作出响应。

从上面的 Child 分析可知，滑动开始的调用 `startNestedScroll()`，Parent 收到 `onStartNestedScroll()` 回调，决定是否需要配合 Child 一起进行处理滑动，如果需要配合，还会回调 `onNestedScrollAccepted()`。

每次滑动前，Child 先询问 Parent 是否需要滑动，即 `dispatchNestedPreScroll()`，这就回调到 Parent 的 `onNestedPreScroll()`，Parent 可以在这个回调中“劫持”掉 Child 的滑动，也就是先于 Child 滑动。

Child 滑动以后，会调用 `onNestedScroll()`，回调到 Parent 的 `onNestedScroll()`，这里就是 Child 滑动后，剩下的给 Parent 处理，也就是 后于 Child 滑动。

最后，滑动结束，调用 `onStopNestedScroll()` 表示本次处理结束。


用个表格 言简意赅：

| child | parent |
| :-------| :--------|
| startNestedScroll | onStartNestedScroll、onNestedScrollAccepted|
| dispatchNestedPreScroll | onNestedPreScroll |
| dispatchNestedScroll | onNestedScroll |
| stopNestedScroll | onStopNestedScroll |


大概原理就这样，不再多说。

### 如何解决？
#### 作为Parent该如何处理？
在实现 NestedScrollingParent后我们要重写以下方法：

```
 @Override
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
        //通知协作
        return true;
    }

    @Override
    public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes) {
        mParentHelper.onNestedScrollAccepted(child, target, nestedScrollAxes);
        startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);//通知更上层的父布局协作
    }

    @Override
    public void onStopNestedScroll(View target) {
        mParentHelper.onStopNestedScroll(target);
        stopNestedScroll();
    }

    @Override
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
        //把子的消耗传上去，这里不对x方向处理。
        dispatchNestedScroll(0, dyConsumed, 0, dyUnconsumed, null);
    }

    @Override
    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
        dispatchNestedPreScroll(dx, dy, consumed, null);

    }

    @Override
    public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed) {
        return dispatchNestedFling(velocityX,velocityY,consumed);
    }

    @Override
    public boolean onNestedPreFling(View target, float velocityX, float velocityY) {
        return dispatchNestedPreFling(velocityX, velocityX);
//        return false;
    }
```

**作为parent时，我们的需求是将 支持Nested的 child 滑动事件上传。**

那么，在`onStartNestedScroll `中，直接返回 `true`，告诉`child` 可以协作;

在`onNestedScrollAccepted`中，调用`mParentHelper.onNestedScrollAccepted`后，再调用该`framelayout`作为`child`的方法：`startNestedScroll `，通知更高层的 `parent`;

在`onNestedScroll`中,直接把其child的消耗传上去;

在`onNestedPreScroll`也是如此，自己不作处理。
`Fling`相关也类似。

#### 作为child如何处理
在实现`NestedScrollingChild`接口，我们要重写如下方法，其实都是调用`mChildHelper`帮助类。

```
 @Override
    public void setNestedScrollingEnabled(boolean enabled) {
        mChildHelper.setNestedScrollingEnabled(enabled);
    }

    @Override
    public boolean isNestedScrollingEnabled() {
        return mChildHelper.isNestedScrollingEnabled();
    }

    @Override
    public boolean startNestedScroll(int axes) {
        return mChildHelper.startNestedScroll(axes);
    }

    @Override
    public void stopNestedScroll() {
        mChildHelper.stopNestedScroll();
    }

    @Override
    public boolean hasNestedScrollingParent() {
        return mChildHelper.hasNestedScrollingParent();
    }

    @Override
    public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow) {
        return mChildHelper.dispatchNestedScroll(dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed, offsetInWindow);
    }

    @Override
    public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
        return mChildHelper.dispatchNestedPreScroll(dx, dy, consumed, offsetInWindow);
    }

    @Override
    public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed) {
        return mChildHelper.dispatchNestedFling(velocityX, velocityY, consumed);
    }

    @Override
    public boolean dispatchNestedPreFling(float velocityX, float velocityY) {
        return mChildHelper.dispatchNestedPreFling(velocityX, velocityY);
    }
```
由于我们的`FrameLayout`并没有滚动的功能，在实现child接口时，就没那么复杂了。

```
  @Override
    public boolean onTouchEvent(MotionEvent event) {
        final int action = MotionEventCompat.getActionMasked(event);

        int y = (int) event.getY();
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                mLastY = y;
                startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);
                break;
            case MotionEvent.ACTION_MOVE:
                int dy = mLastY - y;
                int oldY = getScrollY();
                if (dispatchNestedPreScroll(0, dy, mScrollConsumed, mScrollOffset)) {
                    dy -= mScrollConsumed[1];
                }
                mLastY = y - mScrollOffset[1];
                if (dy < 0) { // 优化剪枝
                    int newScrollY = Math.max(0, oldY + dy);
                    dy -= newScrollY - oldY;
                    if (dispatchNestedScroll(0, newScrollY - dy, 0, dy, mScrollOffset)) {
                        mLastY -= mScrollOffset[1];
                    }
                }
//                stopNestedScroll();
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:

                stopNestedScroll();

                break;
        }
```

主要是在`onTouchEvent`中，在`MotionEvent.ACTION_MOVE`时，分别调用`dispatchNestedPreScroll`  `dispatchNestedScroll` 这两个函数， 在往上滑，并且`AppbarLayout`没有收起时，`dispatchNestedPreScroll`会返回true，否则会走到`dispatchNestedScroll`。

**完整代码如下：**

```
/**
 * Created by duanjobs on 17/9/25.
 */

public class SupportNestedFrameLayout extends FrameLayout implements NestedScrollingChild, NestedScrollingParent {

    private NestedScrollingParentHelper mParentHelper;
    private NestedScrollingChildHelper mChildHelper;
    private int mLastY;
    private final int[] mScrollOffset = new int[2];
    private final int[] mScrollConsumed = new int[2];
    private int mNestedOffsetY;

    public SupportNestedFrameLayout(Context context) {
        this(context, null);
    }

    public SupportNestedFrameLayout(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public SupportNestedFrameLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        mParentHelper = new NestedScrollingParentHelper(this);
        mChildHelper = new NestedScrollingChildHelper(this);
        setNestedScrollingEnabled(true);
    }

    @Override
    public void setNestedScrollingEnabled(boolean enabled) {
        mChildHelper.setNestedScrollingEnabled(enabled);
    }

    @Override
    public boolean isNestedScrollingEnabled() {
        return mChildHelper.isNestedScrollingEnabled();
    }

    @Override
    public boolean startNestedScroll(int axes) {
        return mChildHelper.startNestedScroll(axes);
    }

    @Override
    public void stopNestedScroll() {
        mChildHelper.stopNestedScroll();
    }

    @Override
    public boolean hasNestedScrollingParent() {
        return mChildHelper.hasNestedScrollingParent();
    }

    @Override
    public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow) {
        return mChildHelper.dispatchNestedScroll(dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed, offsetInWindow);
    }

    @Override
    public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
        return mChildHelper.dispatchNestedPreScroll(dx, dy, consumed, offsetInWindow);
    }

    @Override
    public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed) {
        return mChildHelper.dispatchNestedFling(velocityX, velocityY, consumed);
    }

    @Override
    public boolean dispatchNestedPreFling(float velocityX, float velocityY) {
        return mChildHelper.dispatchNestedPreFling(velocityX, velocityY);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        final int action = MotionEventCompat.getActionMasked(event);

        int y = (int) event.getY();
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                mLastY = y;
                startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);
                break;
            case MotionEvent.ACTION_MOVE:
                int dy = mLastY - y;
                int oldY = getScrollY();
                if (dispatchNestedPreScroll(0, dy, mScrollConsumed, mScrollOffset)) {
                    dy -= mScrollConsumed[1];
                }
                mLastY = y - mScrollOffset[1];
                if (dy < 0) {
                    int newScrollY = Math.max(0, oldY + dy);
                    dy -= newScrollY - oldY;
                    if (dispatchNestedScroll(0, newScrollY - dy, 0, dy, mScrollOffset)) {
                        mLastY -= mScrollOffset[1];
                    }
                }
//                stopNestedScroll();
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:

                stopNestedScroll();

                break;
        }
        return true;
    }

    //

    @Override
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
        //通知协作
        return true;
    }

    @Override
    public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes) {
        mParentHelper.onNestedScrollAccepted(child, target, nestedScrollAxes);
        startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);//fu buju xie做
    }

    @Override
    public void onStopNestedScroll(View target) {
        mParentHelper.onStopNestedScroll(target);
        stopNestedScroll();
    }

    @Override
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
        //把子的消耗传上去
        dispatchNestedScroll(0, dyConsumed, 0, dyUnconsumed, null);
    }

    @Override
    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
        dispatchNestedPreScroll(dx, dy, consumed, null);

    }

    @Override
    public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed) {
        return dispatchNestedFling(velocityX,velocityY,consumed);
    }

    @Override
    public boolean onNestedPreFling(View target, float velocityX, float velocityY) {
        return dispatchNestedPreFling(velocityX, velocityX);
//        return false;
    }

    @Override
    public int getNestedScrollAxes() {
        return 0;
    }


}

```
就这么多吧~





