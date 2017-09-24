---
title: 关于View测量宽度高度的一些思考
date: 2017-09-24 20:37:16
categories: Android
keywords: Android, Measure, 高度, 宽度
tags: [Android, ViewGroup, Measure]
---
在做一个平铺效果时，遇到了问题：LinearLayout包含GridView，当GridView的数据源变多时，LinearLayout平铺展开。但是对展开后的高度测量错误导致，展开过多，导致下方很大一片空白。

整个布局大致描述如下：
RecycleView的item布局是LinearLayout包含一个GridView，这里的GridView是重写了onMeasure的。

在分析问题之前，先说说关于Measure的事情，
一般得到某个view的高度或者宽度可以有以下几种

- View.MeasureSpec.makeMeasureSpec
- view.post
- ViewTreeObserver.addOnGlobalLayoutListener

等

一般在LinearLayout中下面的测量方式可以胜任大部分测量高度宽度的任务

```
int intw= View.MeasureSpec.makeMeasureSpec(0,View.MeasureSpec.UNSPECIFIED);
int inth=View.MeasureSpec.makeMeasureSpec(0,View.MeasureSpec.UNSPECIFIED);
v.measure(intw, inth);
int H = v.getMeasuredHeight();
int W = v.getMeasuredWidth();
```
这种在测量高度或者宽度时，当组件中包括其他子组件时，所获取的实际值是这些组件所占的最小宽度和最小高度。如果给父组件设置了matchParent，但是没有子组件，或者子组件为高度或者宽度为0 那么父组件相应的也为0，如果子组件为gone，哪怕孙组件设定了宽和高的值，父组件相应的宽高也为0。因为子组件gone，压根不测了。上面的测量方式和 `view.post` 方式获得的高度是不同的，后者可以得到实际的宽高。
再来说说这种测量方式的几种模式

1. UNSPECIFIED(未指定),父控件对子控件不加任何束缚，子元素可以得到任意想要的大小，这种MeasureSpec一般是由父控件自身的特性决定的。比如ScrollView，它的子View可以随意设置大小，无论多高，都能滚动显示，这个时候，size一般就没什么意义。
2. EXACTLY(完全)，父控件为子View指定确切大小，希望子View完全按照自己给定尺寸来处理，这时的MeasureSpec一般是父控件根据自身的MeasureSpec跟子View的布局参数来确定的。一般这种情况下size>0,有个确定值，或者**父组件有固定，子组件match_parent**
3. AT_MOST(至多)，父控件为子元素指定最大参考尺寸，希望子View的尺寸不要超过这个尺寸。这种模式也是父控件根据自身的MeasureSpec跟子View的布局参数来确定的，一般是子View的布局参数采用wrap_content的时候。

如何确定子View的测量模式呢？父类测量子类是调用子View的`measure(widthMeasureSpec，heightMeasureHeight)`，这个两个测量 `widthMeasureSpec` 规则是父类根据自己的宽度和子类的 LayoutParams 计算出来的。

具体如下图（网图）：
![image](http://oi5p36v0h.bkt.clouddn.com/%E6%B5%8B%E9%87%8F%E6%A8%A1%E5%BC%8F%E5%9B%BE)

那么，回到一开始的问题中去正是由于重写了GridView的onMeasure

```
   @Override
    public void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int expandSpec = MeasureSpec.makeMeasureSpec(Integer.MAX_VALUE >> 2,
                MeasureSpec.AT_MOST);
        super.onMeasure(widthMeasureSpec, expandSpec);
    }
```
一开始出问题的测量方式参数都是UNSPECIFIED，一般没问题，但是这里不行，将参数替换

```
int intw = View.MeasureSpec.makeMeasureSpec(linearLaoutname.getWidth(), View.MeasureSpec.AT_MOST);//（第一个参数也可以设定为屏幕宽度,这里是LinearLayout的宽度）
int inth = View.MeasureSpec.makeMeasureSpec(Integer.MAX_VALUE >> 2,
        MeasureSpec.AT_MOST);
```
问题得到解决。其中需要注意的点，好多人以为如果只测量高度，宽度的参数是不是不重要。当然不是，都得正确，不然测不准。
其中还有个小意外就是 gridview不调用notifyDataSetChanged 也能刷新，为啥呢？因为平铺展开也执行了 requestLayout() 和notifyDataSetChanged 一样的。
