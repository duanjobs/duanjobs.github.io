---
title: 支持手势翻页的FlipPageView
date: 2018-03-10 11:55:14
categories: Android
keywords: 翻页, FlipPageView
description: 支持手势翻页的FlipPageView，可用于考试，读书等
tags: [Android, FlipPageView]
---
### 需求
在版本迭代中，产品需要上线考试，测验的模块，而且考试模块以后是主要的模块，考题一题题的翻页，可手势，也可按钮下一页。于是乎，必须得写一个支持此种操作的控件，而且还要考虑到以后的扩展性。

### 思考
1.必须要支持上下横滑，以及左右滑动翻页。

2.考虑到扩展性，不应当将具体页面（例如考题页面）耦合进该控件中。

3.尽可能的考虑到性能。

### 总体思路
那么，这个空间应包含 `mCurPage` 和 `mTogglePage` 两个子view，滑动切换时，更新相应view的内容，避免创建过多的view，尽量提高性能。

进一步地，为了支持上下滑操作，可以让上述两个view继承自`ScrollView`。

翻页操作时，将创建页面view的操作`creatView`作为接口暴露出去，这样就降低了耦合，可以对页面进行自己的定制。

### 关键代码
话不多说，看看关键代码

```

     /**
     * Page的container view，支持Scroll
     */
    public static class PageContainerView extends ScrollView {
        private FrameLayout mContainer;

        public PageContainerView(Context context) {
            super(context);
            setFillViewport(true);

            mContainer = new FrameLayout(getContext());
            mContainer.setClickable(true);
            addView(mContainer);

            setVerticalScrollBarEnabled(false);
        }

        public View getPageView() {
            return mContainer.getChildAt(0);
        }

        public void setPageView(View page) {
            scrollTo(0,0);
            mContainer.addView(page);
        }

        public void removePageView() {
            mContainer.removeAllViews();
        }

        public boolean checkScrollViewDraged() {
            boolean draged = false;
            try {
                Class scrollClass = ScrollView.class;
                Field draggingField = scrollClass.getDeclaredField("mIsBeingDragged");
                draggingField.setAccessible(true);
                draged = draggingField.getBoolean(this);
            }catch (Exception e) {
                e.printStackTrace();
            }
            return draged;
        }
    }

```
上面这段代码就是FlipPageView的子view，继承自ScrollView，逻辑很简单，就是将页面view 添加或者移除，其中还有个判断是否拖动的反射操作，这个之后在检测手势的时候会用到。

```
    public interface FlipPageListener {
        public View onCreatePage(View oldpage,int index);
        public void onPageChanged(View newpage,int index);
    }
```
上面是FlipPageView的回调，主要是创建和翻页的回调，创建时，会把oldpage回传，如果oldpage不为空，则可以直接拿来用，不需要创建新的view再添加进来。


在写一个功能时，不得已采用方案：`RecyclerView`嵌套`GridView`，于是乎，坑来了。


```
 @Override
    public boolean dispatchTouchEvent(MotionEvent event) {
        if(!allowGesture)
            return super.dispatchTouchEvent(event);
        boolean handled = false;
        int action = event.getAction();

        checkPageIndexValid();
        if(mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(event);

        switch(action) {
            case MotionEvent.ACTION_DOWN: {

                stopScrollPage();

                mScrollStartX = 0;
                mPageDragged = false;
                mFlipTo = PageFlipTo.NONE;
                mGesture = Gesture.NONE;
                mLastPointX = mCurPointX = mInitPointX = event.getX();
            }
            break;

            case MotionEvent.ACTION_MOVE: {

                mCurPointX = event.getX();

                if(mGesture == Gesture.NONE) {
                    if(mCurPage.checkScrollViewDraged()) {
                        mGesture = Gesture.VERTICAL;
                    }else if(checkPageDraged(event)) {
                        mGesture = Gesture.HORIZONTAL;
                    }
                }

                if(mGesture == Gesture.HORIZONTAL) {
                    checkPageDraged(event);
                    if(checkPageFlipTo(event)) {
                        dragPage();
                    }
                    handled = true;
                }

                mLastPointX = mCurPointX;
            }
            break;

            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP: {

                mVelocityTracker.computeCurrentVelocity(1000);
                float velocity = mVelocityTracker.getXVelocity();
                if(mVelocityTracker != null) {
                    mVelocityTracker.recycle();
                    mVelocityTracker = null;
                }

                if(mPageDragged) {
                    if(!checkFlingPage(velocity)) {
                        checkScrollPage();
                    }
                }
                handled = mGesture == Gesture.HORIZONTAL ? true : handled;
            }
        }

        if(!handled) {
            handled = super.dispatchTouchEvent(event);
        }
        return handled;
    }
```

上面这个函数是熟悉的时间分发函数，刚开始回判断是否支持手势翻页，不支持则不拦截。

在`ACTION_MOVE`里会判断手势的方向，如果是横滑操作，则进行滑动判断，如下：

```
    /**
     * 检查页面切换，上一页 or 下一页
     * @param event
     * @return true: 能滑动; false: 不能滑动
     */
    private boolean checkPageFlipTo(MotionEvent event) {
        boolean canflip = false;
        PageFlipTo original = mFlipTo;

        if(mCurPointX >= mInitPointX) {
            mFlipTo = PageFlipTo.PREVIOUS;
        }else if(mCurPointX < mInitPointX) {
            mFlipTo = PageFlipTo.NEXT;
        }
        if(mFlipTo == PageFlipTo.PREVIOUS) {
            canflip = mPageIndex - 1 < 0 ? false : true;
        }else if(mFlipTo == PageFlipTo.NEXT) {
            canflip = mPageIndex + 1 >= mPageCount ? false : true;
        }

        if(canflip && original != mFlipTo) {
            flipPageInit();
        }

        if(!canflip) {
            mPageDragged = false;
            mInitPointX = event.getX();
        }

        return canflip;
    }
```
主要是判断是上一页还是下一页相关，然后根据判断，进行翻页的初始化操作 `flipPageInit()`


```
 	 /**
     * 翻页初始化
     */
    private void flipPageInit() {
        View newPage;
        int pageindex;

        if(mFlipPageListener == null) {
            return;
        }

        if(mFlipTo == PageFlipTo.PREVIOUS) {
            pageindex = mPageIndex - 1 < 0 ? 0 : mPageIndex - 1;

            newPage = mFlipPageListener.onCreatePage(mRecyclePageView, pageindex);
            setPageView(mTogglePage,newPage);
            mTogglePage.setX(-mPageWidth);
            mCurPage.setX(0);

            mMovePage = mTogglePage;
        }else if(mFlipTo == PageFlipTo.NEXT) {
            pageindex = mPageIndex + 1 >= mPageCount ? mPageCount - 1 : mPageIndex + 1;

            newPage = mFlipPageListener.onCreatePage(mRecyclePageView, pageindex);
            setPageView(mTogglePage,newPage);
            mTogglePage.setX(0);
            mCurPage.setX(0);

            mMovePage = mCurPage;
        }
        mMovePage.bringToFront();
    }
```

上面代码就是进行翻页的初始化，主要是进行页面的创建和摆放位置，逻辑较简单。

其他代码则是辅助代码，整个的代码逻辑大体都在上面。

当然还有支持按钮翻页的则比较简单，就是利用 `Scroller` 进行页面的切换，具体代码不贴了。

具体代码在这 [点我](https://github.com/duanjobs/SomeThingMine)

效果

![image](http://oi5p36v0h.bkt.clouddn.com/ezgif.com-video-to-gif.gif)