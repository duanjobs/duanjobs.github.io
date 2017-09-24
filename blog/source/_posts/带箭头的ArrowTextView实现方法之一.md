---
title: 带箭头的ArrowTextView实现方法之一
date: 2017-09-24 15:23:38
categories: Android
keywords: 箭头, ArrowTextView, 自定义
tags: [Android, 自定义View, ArrowTextView]
---
## 自定义View的需求

在开发的时候，需要一个带箭头的TextView用来对某功能做简短介绍，当然可以让设计师切个图啦，但是自己搞一个岂不是美滋滋？
为此，想用两种方式实现这个需求：

 - 继承TextView，重写onDraw();
 - 继承RelativeLayout等类，形成组合控件。
那这一篇就先写写比较简单的第一种思路吧。

 
### 实现分析
**这里不做自定义View的讲解，只讲实现思路**
继承TextVie的好处就是能够利用TextView本身的文字功能，不需要过多的考虑如何处理文字；否则，如果继承View的话，光处理文字就比较复杂。
#### step1 增加自定义属性
自定义View需要有自定义的属性，在ArrowTextView中，大概有以下几个方面需要定义：

 1. 箭头相关：方向、高、宽
 2. 整个View的背景色
 3. 箭头相对边的位置（箭头居中或者靠前等）
 4. 圆角矩形的角的Radius
先定义这么多，后期需要的话再扩展。

那么，在attr中增加如下代码:
```
<resources>
    <declare-styleable name="ArrowTextView">
        <!-- Corner radius for ArrowTextView. -->
        <attr name="arrowCornerRadius" format="dimension"/>
        <!-- Background color for ArrowTextView. -->
        <attr name="arrowBackgroundColor" format="color"/>
        <!-- direction for arrow. -->
        <attr name="arrowDirection">
            <enum name="left" value="1"/>
            <enum name="top" value="2"/>
            <enum name="right" value="3"/>
            <enum name="bottom" value="4"/>
        </attr>
        <!-- position for arrow. -->
        <attr name = "relativePosition" format = "fraction" />
        <!-- arrow width height -->
        <attr name="arrowWidth" format="dimension" />
        <attr name="arrowHeight" format="dimension" />
        <attr name="arrowTextSize" format="dimension"/>
        <attr name="arrowTextColor" format="color"/>
        <attr name="arrowText" format="string"/>
    </declare-styleable>
</resources>
```
分别对应组合空间中TextView的字体大小、颜色和文本。

#### step2 继承RelativeLayout



#### step2 重写ArrowTextView
首先，定义所需要的变量，分别对应自定义属性：
```
protected float radius;
protected float arrowWidth;
protected int backroundColor;
protected float arrowHeight;
protected ArrowDirection arrowDirection;
protected float relativePosition;
```
然后，重写构造函数，在构造函数里取出定义属性的值,在代码里可以看到，构造函数都会走到第三个函数里进行变量的初始化。
```
public ArrowTextView(Context context) {
        this(context,null,0);
    }
    public ArrowTextView(Context context, AttributeSet attrs) {
        this(context, attrs,0);
    }
    public ArrowTextView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mContext = context;
        TypedArray typedArray = context.obtainStyledAttributes(attrs, R.styleable.ArrowTextView);
        radius = typedArray.getDimension(R.styleable.ArrowTextView_arrowCornerRadius,0);
        arrowWidth=typedArray.getDimension(R.styleable.ArrowTextView_arrowWidth, 0);
        arrowHeight=typedArray.getDimension(R.styleable.ArrowTextView_arrowHeight, 0);
        backroundColor = typedArray.getColor(R.styleable.ArrowTextView_arrowBackgroundColor, Color.GRAY);
        float position=typedArray.getFraction(R.styleable.ArrowTextView_relativePosition,1,1,0.3f);
        setRelativePosition(position);
        int direction = typedArray.getInt(R.styleable.ArrowTextView_arrowDirection,1);
        setArrowDirection(direction);
    }
```
其中，`setRelativePosition(position)`和`setArrowDirection(direction)`是将定义的属性值映射成所需的值，代码如下：
```
private void setRelativePosition(float position){
        if(position<=0.05f){
            relativePosition=0.5f;
        }else if(position>0.95f){
            relativePosition=0.95f;
        }else{
            relativePosition=position;
        }
    }

private void setArrowDirection(int direction) {
        switch (direction) {
            case 1: {
                arrowDirection = ArrowDirection.LEFT;
            }
            break;
            case 2: {
                arrowDirection = ArrowDirection.TOP;
            }
            break;
            case 3: {
                arrowDirection = ArrowDirection.RIGHT;
            }
            break;
            case 4: {
                arrowDirection = ArrowDirection.BOTTOM;
            }
            break;
            default:
                arrowDirection= ArrowDirection.LEFT;
        }
    }
```
在`setRelativePosition(float position)`方法中，对相对位置作处理，避免不合理的情况出现，默认相对位置最小为5%，最大为95%，其他情况，可以任意取值。
在`setArrowDirection(int direction)`方法中，将xml中定义的属性映射成arrowDirection（ENUM），默认方向为LEFT。
```
 public enum ArrowDirection {
        LEFT, TOP, RIGHT, BOTTOM
    }
```
至此，准备工作完毕，开始onDraw方法的重写，在这里需要画出一个圆角矩形和一个箭头。具体在什么位置画还需要分析。
**假设箭头为LEFT，那么，圆角矩形的左边起始X坐标不应为0，应为箭头高度：ArrowHeight的值。其他三个边不变，上侧边为0，右侧边为控件的宽度值，底侧边应为控件的高度值。**
**假设箭头为TOP，那么除了上侧边的起始位置为箭头高度：ArrowHeight的值，其他三个边不变。**
**假设箭头为Right，那么右侧边的位置为控件的宽度值 - arrowHeight，其他三个不变。**
**假设箭头为Bottom，那么底侧边的起始位置为控件的高度值-arrowHeight，其他三个不变。**
以左侧箭头为例：
`canvas.drawRoundRect(new RectF(arrowHeight, 0, width, height), radius, radius, paint);`
画完了圆角矩形，开始把箭头画上去，以左侧箭头为例，
```
float yMiddle = height * relativePosition;
//上面一行通过自定义属性relativePosition取三角形顶点的y坐标值
float yTop = yMiddle - (arrowWidth / 2);
float yBottom = yMiddle + (arrowWidth / 2);
//上面两行是取底边两个顶点的y坐标值。
path.moveTo(0, yMiddle);
path.lineTo(arrowHeight, yTop);
path.lineTo(arrowHeight, yBottom);
path.lineTo(0, yMiddle);
//上面四行是画出这个三角箭头，左侧箭头的起始坐标为（0, yMiddle），两个底边顶点的标为分别为(arrowHeight, yTop)、(arrowHeight, yBottom);
```
分析图如图所示：
![坐标分析图][1]

这样，ArrowTextView大体就完成了，但运行时发现，当有文字的时候会覆盖住箭头啊。看图：
![此处输入图片的描述][2]

原因在于哪呢？主要是由于TextView的大小是包含这个小箭头的，所以在没有设置padding的情况下，文字当然从箭头开始啊。那能不能把padding修改下呢，这个得追根溯源，看看TextView的代码，于是我们发现了`getCompoundPaddingLeft()`这个计算左侧padding的函数。
```
/**
     * Returns the left padding of the view, plus space for the left
     * Drawable if any.
     */
    public int getCompoundPaddingLeft() {
        final Drawables dr = mDrawables;
        if (dr == null || dr.mShowing[Drawables.LEFT] == null) {
            return mPaddingLeft;
        } else {
            return mPaddingLeft + dr.mDrawablePadding + dr.mDrawableSizeLeft;
        }
    }
```
找到根源啦，那就重写下计算padding的四个函数呗，思路就是根据箭头的方向，设置对应的padding为真实padding+arrowHeight的值，这样就可以把文字挤出小箭头的区域啦，还是以左侧为例。
```
@Override
    public int getCompoundPaddingLeft() {
        return arrowDirection == ArrowDirection.LEFT ? super.getCompoundPaddingLeft() +
                (int) arrowHeight : super.getCompoundPaddingLeft();
    }
```

这样，基本就完成了ArrowTextView，其中遇到的小坑就是，在OnDraw（）函数中，我们会调用父类的` super.onDraw(canvas);`这个顺序是有讲究的，如果先调用这个方法，则先绘制TextView，再绘制自己的逻辑；如果后调用，则先绘制自己的逻辑，在绘制原生TextView。在这里需要先绘制自己的图层，最后调用这个方法，否则会把TextView中的文字覆盖掉。效果图：
![此处输入图片的描述][3]
接下来就是用第二种方式实现，下一篇吧。
代码地址：https://github.com/duanjobs/ArrowTextView


  [1]: http://oi5p36v0h.bkt.clouddn.com/fenxibiaozuo.png
  [2]: http://oi5p36v0h.bkt.clouddn.com/layout-2016-12-14-114446.png
  [3]: http://oi5p36v0h.bkt.clouddn.com/QQ20161214-0@2x.png
