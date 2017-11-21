---
title: HorizontalScrollView实现自定义翻页（一页一页）
date: 2017-11-10 15:45:14
categories: Android
keywords: HorizontalScrollView, 翻页
description: HorizontalScrollView实现自定义翻页（一页一页）非ScrollTo方法
tags: [Android, HorizontalScrollView]
---
### 前文

做这个的初衷是为了满足产品的修改要求：在展示介绍图片时候，需要有翻页的效果不能停在半路。。哈哈哈，产品要求说改就改，那原来用的`HorizontalScrollView`咋办？

### 分析
`HorizontalScrollView`还能不能用？能用啊，判断每个`child`的位置，然后`scrollto`是可以满足要求的啊，可是交互不愿意了：你这切换的太快了啊！好好好，我改行么。

难不成放弃`HorizontalScrollView`？**不，我不会的。**
`scrollto`的时间又不能改，人家写死`250`了。
于是我想到了属性动画`ObjectAnimator`。
> 总结下就是：不想放弃HorizontalScrollView，反射又不想用，只能用属性动画才能维持得了代码这样子。


### 代码实现分析

大体思路：首先，继承`HorizontalScrollView`。然后重写`dispatchTouchEvent`，再然后重写`fling`。

具体思路：`dispatchTouchEvent`里判断手势滑动了多少，如果超过阈值，那就换页。反之，则恢复原位。

`fling` 时判断方向，默认为直接翻页操作。

是的，就这么简洁。 不过写的时候要注意很多。

直接上代码！

```
public class HScrollView extends HorizontalScrollView {
  
    float x, y;
    float curX = 0;
    float downX;
    private ViewGroup mChild;
    private int maxScrollx;
    private HashMap<Integer, Integer> map = new HashMap<>();
    private int nowChild = 0;
    private int childCout = 0;

    private int slidGap = 0;
    private Boolean isScroll = false;
    private static final int duration =350;

    public HScrollView(Context context) {
        this(context, null);
    }

    public HScrollView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        mChild = (ViewGroup) getChildAt(0);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        maxScrollx = mChild.getMeasuredWidth() - getMeasuredWidth();
        caculateChild(mChild);
        Log.e("tag", "firstwidth=" + mChild.getMeasuredWidth() + "; getMeasuredWidth()=" + getMeasuredWidth());
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN: {
                downX = curX = x = ev.getX();
                y = ev.getY();
                isScroll = false;
            }
            break;

            case MotionEvent.ACTION_MOVE: {
            //这里的操作为为了兼容在viewpager里使用该控件，滑到两边会把事件抛上去。
                curX = ev.getX();
                if (curX - x > 0 && getScrollX() == 0) {
                    return false;
                }
                if (curX - x < 0 && getScrollX() == maxScrollx) {
                    return false;
                }
                x = ev.getX();
                y = ev.getY();
            }
            break;
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP: {
               if (downX - ev.getX() > getSlidGap()) {
                    if (nowChild <= childCout - 2) {
                        ObjectAnimator.ofInt(this, "scrollX", map.get(++nowChild)).setDuration(duration).start();
                        isScroll = true;
                    }
                } else if (downX - ev.getX() < -getSlidGap()) {
                    if (nowChild >= 1) {
                        ObjectAnimator.ofInt(this, "scrollX", map.get(--nowChild)).setDuration(duration).start();
                        isScroll = true;
                    }
                } else {
                    ObjectAnimator.ofInt(this, "scrollX", map.get(nowChild)).setDuration(duration).start();
                }
            }
            break;
        }
        Log.d("tag", "lastX=" + x + "; curX=" + curX + ";Event=" + ev.getAction() + ";scrollX=" + getScrollX() + ";downX=" + downX);
        return super.dispatchTouchEvent(ev);
    }

    private void caculateChild(ViewGroup viewGroup) {
        childCout = viewGroup.getChildCount();
        float center = 0;
        map.put(0, 0);
        for (int i = 1; i < childCout; i++) {
            View v = viewGroup.getChildAt(i - 1);
            if (i == 1) {
                map.put(i, map.get(i - 1) + v.getMeasuredWidth());
            } else {
                map.put(i, map.get(i - 1) + v.getMeasuredWidth());
            }
        }
    }

    private int getSlidGap() {
        if (slidGap != 0) {
            return slidGap;
        } else {
            slidGap = getScreenWidth(getContext()) / 3;
            return slidGap;
        }
    }

    @Override
    public void fling(int velocityX) {
        if (!isScroll) {
            if (velocityX < 0 && nowChild > 0) {
                ObjectAnimator.ofInt(this, "scrollX", map.get(--nowChild)).setDuration(duration).start();
            } else if (velocityX > 0 && nowChild < childCout - 1) {
                ObjectAnimator.ofInt(this, "scrollX", map.get(++nowChild)).setDuration(duration).start();
            }
        }
    }
```

代码中 `slidGap = getScreenWidth(getContext()) / 3;`是滑动阈值，超过时候会翻页，这里写死了屏幕宽度的三分之一。

`caculateChild(ViewGroup viewGroup)`方法是计算每个`child`的最左边，记录下来，之后翻页的时候直接滚动。

比较核心的代码就是下面这些哈：

```
case MotionEvent.ACTION_UP: {
                Log.d("log", "up x=" + ev.getX());
                if (downX - ev.getX() > getSlidGap()) {
                    if (nowChild <= childCout - 2) {
                        ObjectAnimator.ofInt(this, "scrollX", map.get(++nowChild)).setDuration(duration).start();
                        isScroll = true;
                    }
                } else if (downX - ev.getX() < -getSlidGap()) {
                    if (nowChild >= 1) {
                        ObjectAnimator.ofInt(this, "scrollX", map.get(--nowChild)).setDuration(duration).start();
                        isScroll = true;
                    }
                } else {
                    ObjectAnimator.ofInt(this, "scrollX", map.get(nowChild)).setDuration(duration).start();
                }
            }
            break;
```

主要就是在UP事件发生时，判断滑动距离，看是否是切到下一页，还是上一页，还是滑回原样。

```
ObjectAnimator.ofInt(this, "scrollX", map.get(++nowChild)).setDuration(duration).start();
``` 
`ObjectAnimator`动画一句话搞定

基本实现需求，不多说了~看看效果吧(gif录得不好。。)：
![](http://oi5p36v0h.bkt.clouddn.com/ezgif.com-gif-maker.gif)













