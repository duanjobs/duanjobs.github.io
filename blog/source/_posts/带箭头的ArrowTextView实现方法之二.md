---
title: 带箭头的ArrowTextView实现方法之二
date: 2017-09-24 16:03:54
categories: Android
keywords: 箭头, ArrowTextView, 自定义View, Android
tags: [Android, 自定义View, ArrowTextView]
---
## 组合控件实现ArrowTextView

 在上篇写了如何继承TextView实现带箭头的TextView，这篇阐述如何通过继承RelativeLayout来实现相同需求。大体思路是通过一个箭头的资源和textView来实现。详细点说就是通过relativeLayout包裹一个TextView，然后和箭头的imageView进行组合，其中要考虑箭头的旋转、颜色以及两者的相对位置。
 
### 实现分析

#### step1 增加自定义属性
在上篇的ArrowTextView中，已经定义了以下几个方面：

 1. 箭头相关：方向、高、宽
 2. 整个View的背景色
 3. 箭头相对边的位置（箭头居中或者靠前等）
 4. 圆角矩形的角的Radius

现在在attr中增加如下属性:
```
<attr name="arrowTextSize" format="dimension"/>
<attr name="arrowTextColor" format="color"/>
<attr name="arrowText" format="string"/>
```
分别对应组合空间中TextView的字体大小、颜色和文本。

#### step2 继承RelativeLayout
首先，定义所需要的变量，分别对应自定义属性：
```
protected RelativeLayout rel;
protected ImageView arrowImage;
protected TextView tvContent;
protected TintedBitmapDrawable imgDrawable;
protected RoundRectDrawable roundRectDrawable;
```
相对于上篇来说主要是增加了上面五个，其中`imgDrawable`为箭头上色，`roundRectDrawable`作为包含textView的RelativeLayout的背景。

然后，重写构造函数，在构造函数里取出定义属性的值,在代码里可以看到，构造函数都会走到第三个函数里进行变量的初始化（此处只写第三个构造函数�di）。
```
public ArrowTextViewRelative(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mContext = context;
        TypedArray typedArray = context.obtainStyledAttributes(attrs, R.styleable.ArrowTextView);
        radius = typedArray.getDimension(R.styleable.ArrowTextView_arrowCornerRadius, 0);
        backroundColor = typedArray.getColor(R.styleable.ArrowTextView_arrowBackgroundColor, Color.GRAY);
        int textColor = typedArray.getColor(R.styleable.ArrowTextView_arrowTextColor, 0);
        float textSize = typedArray.getDimension(R.styleable.ArrowTextView_arrowTextSize, 0);
        String contentText = typedArray.getString(R.styleable.ArrowTextView_arrowText);
        int direction = typedArray.getInt(R.styleable.ArrowTextView_arrowDirection, 1);
        setArrowDirection(direction);

        float position = typedArray.getFraction(R.styleable.ArrowTextView_relativePosition, 1, 1, 0.3f);
        setRelativePosition(position);
        typedArray.recycle();
        layoutContent(radius, backroundColor, textColor, textSize, contentText);

    }
```
其中，和上篇内容差不多的不再赘述，layoutContent()为主要代码，主要是将组合的控件摆放到合适的位置，其中代码主要分为四部分：

 - **生成包裹textView的RelativeLaout**
```
rel = new RelativeLayout(mContext);
rel.setId(View.generateViewId());
        LayoutParams relParams = new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
        roundRectDrawable = new RoundRectDrawable(backroundColor, radius);
        rel.setBackground(roundRectDrawable)
```
在代码中可以看到，为rel添加了圆角矩形的背景，这个圆角矩形代码来源于v7包中。[链接在此:RoundRectDrawable.java][1]
 - **生成textView并添加到Rel中**
```
  //add textview
tvContent = new TextView(mContext);
tvContent.setId(View.generateViewId());
tvContent.setTextColor(textColor);
tvContent.setTextSize(TypedValue.COMPLEX_UNIT_PX, textSize);
tvContent.setText(contentText);
int vpadding = dip2px(18);
tvContent.setPaddingRelative(vpadding, vpadding, vpadding, vpadding);
rel.addView(tvContent);
```
 - **生成箭头的imageView，并做好旋转和上色**
```
//add arrowImage
arrowImage = new ImageView(mContext);
arrowImage.setId(View.generateViewId());
LayoutParams arrowParams = new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
```
由于添加的箭头是向右的，所以需要根据xml文件中定义的箭头指向旋转并摆放好与rel的相对位置。以top为例，此时箭头应当旋转270度，并且rel应该在箭头的下方，这也是`relParams.addRule(RelativeLayout.BELOW,arrowImage.getId())`这句代码所实现的功能。
```
int rotate =0;
switch (arrowDirection) {
    case TOP:
        rotate = 270;
        relParams.addRule(RelativeLayout.BELOW, arrowImage.getId());
        break;
    case BOTTOM:
        rotate = 90;
        arrowParams.addRule(RelativeLayout.BELOW, rel.getId());
        break;
    case LEFT:
        rotate = 180;
        relParams.addRule(RelativeLayout.END_OF, arrowImage.getId());
        break;
    case RIGHT:
        rotate = 0;
        arrowParams.addRule(RelativeLayout.END_OF, rel.getId());
        break;
        }
```
箭头的上色主要是参考了[Tinting drawables][2]这篇文章。
至此，准备工作完毕，开始onDraw方法的重写，在这里需要画出一个圆角矩形和一个箭头。具体在什么位置画还需要分析。
```
int arrowRes = R.mipmap.ic_arrow;
Bitmap source = BitmapFactory.decodeResource(this.getResources(), arrowRes);
Bitmap rotateBitmap = rotateBitmap(source, rotate);
//tint the bitmap with  backroundColor
imgDrawable = new TintedBitmapDrawable(mContext.getResources(), rotateBitmap, backroundColor);
arrowImage.setImageDrawable(imgDrawable);
```
 - **将控件添加到父RelativeLayout中**
```
this.addView(arrowImage, arrowParams);
this.addView(rel, relParams);
```
此时仅仅是做好了摆放，那么箭头的relativePosition怎么实现呢，我们都知道在`onCreate()`方法中调用`view.getHeight()`会返回0，所以并不能直接在摆放好之后就去获取rel的高度和宽度去调整箭头的相对位置，只能通过view的post方法去获取，所以用此类继承Runable接口，实现其run方法来实现箭头relativePosition,当然别忘了post(this);
```
@Override
    public void run() {
        int arrowOffset;
        int conRlw = rel.getWidth();
        int conRlh = rel.getHeight();
        LayoutParams params = (LayoutParams) arrowImage.getLayoutParams();
        switch (arrowDirection) {
            case TOP:
                arrowOffset = (int) ((conRlw * relativePosition - arrowImage.getWidth() / 2));
                params.setMargins(arrowOffset, arrowImage.getHeight(), 0, 0);
                break;
            case BOTTOM:
                arrowOffset = (int) ((conRlw * relativePosition - arrowImage.getWidth() / 2));
                params.setMargins(arrowOffset, 0, 0, arrowImage.getHeight());
                break;
            case LEFT:
                arrowOffset = (int) ((conRlh * relativePosition - arrowImage.getWidth() / 2));
                params.setMargins(arrowImage.getWidth(), arrowOffset, 0, 0);
                break;
            case RIGHT:
                arrowOffset = (int) ((conRlh * relativePosition - arrowImage.getWidth() / 2));
                params.setMargins(0, arrowOffset, arrowImage.getWidth(), 0);
                break;
        }
    }
```
上个图看看撒，后四个是本篇的，前两个是上一篇的。
![此处输入图片的描述][3]
代码地址：[gitHub][4]


  [1]: https://chromium.googlesource.com/android_tools/+/78ccfd5f7e4880597fd90c61453a3be0e7aee5f0/sdk/sources/android-21/android/support/v7/widget/RoundRectDrawable.java
  [2]: http://andraskindler.com/blog/2015/tinting_drawables/
  [3]: http://oi5p36v0h.bkt.clouddn.com/Screenshot_20170105-122745.png
  [4]: https://github.com/duanjobs/ArrowTextView
