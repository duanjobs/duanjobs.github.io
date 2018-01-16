---
title: ViewPager作为ListView的Item时的疑难杂症
date: 2018-01-16 14:06:16
categories: Android
keywords: ViewPager, ListView, Item
description: ViewPager作为ListView的Item时,划出屏幕再划回来会有什么刺激的表现
tags: [Android, ViewPager, ListView, 复用]
---
### 前文

长复杂页面在项目中采用`listview`实现的形式，利用`mergeAdapter`来对界面做控制，偏偏有个`itemView` 是`viewpager`的形式，于是该`itemView`里有个`ViewPager`作为子`view`来做横滑操作。做出后，看上去很美。但是，突然发现当该`item`划出屏幕，再划回来的时候，第一次切换`viewPager`总会秒切，而且不丝滑，但之后的切换操作并无什么异常，都很丝滑。强迫症受不了，必须搞。而且只在7.x的版本上有问题，在4.X，6.X上都很perfect。

### 分析

* **首先是冥想阶段**

1. 既然和划出屏幕有关系，那么可以认为：`listView`的`Item`回收机制造成的影响。
2. 可能是`ViewPager`在划出划回的状态有问题。

### 排查阶段
* 先看看`ViewPager` 的`OnPageChangeListener` 在正常与异常情况下切换时回调的状态。发现在`onPageScrollStateChanged`的回调中，异常状态少了 `IDLE`（0） 和`SETTLING` （2），只有`DRAGGING`（1）。

* 查看源码 `IDLE`（0） 只在`smoothScrollTo`方法中设置过此状态，依次上溯，发现在`setCurrentItemInternal`方法中有代码块：

```
if (mFirstLayout) {
            
   mCurItem = item;
   if (dispatchSelected) {
       dispatchOnPageSelected(item);
   }
   requestLayout();
   } else {
       populate(item);
       scrollToItem(item, smoothScroll, velocity, dispatchSelected);
   }
```

* 很大概率是此代码块的`if else` 语句造成的，换言之就是 `mFirstLayout` 变量造成的。打个断点试试看！
* 哇，果然是 当划出屏幕后再回来 走的是 `mFirstLayout＝true`，而正常情况下则是走的`else`方法。

* 接下来看看 `mFirstLayout`在哪被赋值过。

```
public void setAdapter(PagerAdapter adapter) {
    ...
    mFirstLayout = true;
}

@Override
protected void onAttachedToWindow() {
    super.onAttachedToWindow();
    mFirstLayout = true;
}

@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    ...
    mFirstLayout = false;
}
```
**mFirstLayout在上述方法中赋值过，那么问题就显而易见了。**

**原因：当该`itemView`被划出屏幕时，会被`detach`掉，再划回来则会走`onAttachedToWindow`方法，此时`mFirstLayout`被设置为`true`，所以在划回来，第一次切换时会造成前文提到的情况。**


**解决方案也随之出现(回字有几种写法？)：**

1. 设置`viewpager`的`OnAttachStateChangeListener`侦听，在`onViewAttachedToWindow`回调中，调用`ViewPager.requestLayout()`;方法将`mFirstLayout` 置为`false`。（对应`onLayout`）
2. 利用反射，在`onViewAttachedToWindow`回调中，将`mFirstLayout`置为`false`。这个方法有人会有疑问就是 那岂不是在第一次`onViewAttachedToWindow`的时候`mFirstLayout`也为`false`？答案是`yes`，但是别忘了，`setAdapter`里会置为`true`。往往我们第一次`onViewAttachedToWindow`之后都会给`ViewPager` `setAdapter` 啊。所以整体还是对的。
3. 投机取巧法。当`itemView`划出时调用两次`setCurrentItem（currentItem）`方法，这样第二次就走`else`的代码了。但是此方法需要写的代码有点多，而且罗嗦，耦合高，我压根就没试。

**方法一：**

```
viewPager.addOnAttachStateChangeListener(new OnAttachStateChangeListener() {
            @Override
            public void onViewAttachedToWindow(View v) {
                viewPager.requestLayout();
            }

            @Override
            public void onViewDetachedFromWindow(View v) {

            }
        });
```

**方法二：**


```
viewPager.addOnAttachStateChangeListener(new OnAttachStateChangeListener() {
            @Override
            public void onViewAttachedToWindow(View v) {
               resetFirstLayout();
            }
            @Override
            public void onViewDetachedFromWindow(View v) {

            }
        });    
           

```
`resetFirstLayout`的主要语句为：

```
            Field mFirstLayout = ViewPager.class.getDeclaredField("mFirstLayout"); 
       
```

`mFirstLayout.setAccessible(true);`
`mFirstLayout.set(viewPager, false);
`
(这两句加到代码块生成不了html，蛋疼，拿出来意会就好)

好了，就这么多吧。。









