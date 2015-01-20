---
layout: post
title: "Android 自定义View（Layout）"
description: ""
category: "知识整理"
tags: [Android,View]
---


在Android开发中有时候需要自定义一些View或者Layout来满足特殊的需求，自定义View需要集成[View类](http://developer.android.com/reference/android/view/View.html)
并重载一些必要的方法，同时为了能够在XML文件中像使用系统原生控件一样简单，也需要自定义一些属性。

## 继承一个View类

自定义View可以从继承View类开始，有两个构造方法最好都实现了，第一个构造方法比较方便在代码中实例化类的时候使用，第二个构造方法多了
一个 `AttributeSet` 参数，这个参数是用来在XML中配置类的属性的时候使用的，如果再XML布局中使用必须实现这个方法。

```
class PieChart extends View {
    public PieChart(Context context) {
        super(context);
    }
    
    public PieChart(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
}
```

为了能够在XML中配置自定义控件的属性，需要先在`attrs.xml`文件中把需要的配置属性声明出来，再在代码中解析这些属性，然后就可以在
XML布局中使用了。比如对于PieChart这个控件，希望可以在XML中配置显示的文字(showText)和文字位置(labelPosition)，那么需要在
`attrs.xml`文件中添加一个`<declare-styleable>`元素，并做类似这样的配置：

```
<resources>
   <declare-styleable name="PieChart">
       <attr name="showText" format="boolean" />
       <attr name="labelPosition" format="enum">
           <enum name="left" value="0"/>
           <enum name="right" value="1"/>
       </attr>
   </declare-styleable>
</resources>
```

`<declare-styleable>` 元素的名字约定是和类的名字一样的都是 `PieChart`（并不是必须的，这样约定方便编辑器做代码补全），然后就可以在
XML布局中使用这些属性了。在XML文件中有命名空间的概念告诉XML文件某个属性的来源和说明，在我们写下`android:layout_width="wrap_content"`
的时候可能并没有注意到在XML文件的开始有一句话 `xmlns:android="http://schemas.android.com/apk/res/android"` 这是一个命名空间的声明，引号里面的是相关属性的命名空间，`android`
是这个命名空间的别名，关于命名空间的概念可以查看[这里](http://www.w3school.com.cn/xml/xml_namespaces.asp)。在自定义的XML文件中所引用的属性也需要加上这样一个声明才可以使用,Android中约定成这样`http://schemas.android.com/apk/res/[your package name]`。

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
   xmlns:custom="http://schemas.android.com/apk/res/com.example.customviews">
  <com.example.customviews.charting.PieChart
     custom:showText="true"
     custom:labelPosition="left" />
</LinearLayout>
```

但是这样在XML文件中配置完了，并不会对PieChart这个View产生任何影响，还需要在代码中解析这些属性，在通过XNL配置的View中，配置的属性
在代码中是通过 ` AttributeSet` 来传递的，Android在解析这个XML布局的时候会把这些配置信息解析出来，并通过` AttributeSet`传入到View的
构造函数中。

```
public PieChart(Context context, AttributeSet attrs) {
   super(context, attrs);
   TypedArray a = context.getTheme().obtainStyledAttributes(
        attrs,
        R.styleable.PieChart,
        0, 0);

   try {
       mShowText = a.getBoolean(R.styleable.PieChart_showText, false);
       mTextPos = a.getInteger(R.styleable.PieChart_labelPosition, 0);
   } finally {
       a.recycle();
   }
}
```
`AttributeSet` 中存储的属性信息是经过编码的，需要通过`TypedArray` 来把这些信息解析出来。为了在代码中也能更改一些必要信息，还需要
提供一些函数设置这些属性

```
public void setShowText(boolean showText) {
   mShowText = showText;
   // View内容发生变化的时候调用此方法法，重绘View
   invalidate();
   // View的大小发生变化的时候调用此方法，重新布局
   requestLayout();
}
```

## 绘制View

View被绘制的时候都会调用`onDraw(Canvas c)`方法，这个方法只有一个参数[Canvas](http://developer.android.com/reference/android/graphics/Canvas.html) ，Canvas就是一张画布啦，通过这个类提供的诸多方法
就可以图形画到屏幕上了。有了画布还需要画笔[Paint](http://developer.android.com/reference/android/graphics/Paint.html)，对于这两个类Android文档上有一段详细的解释

1. What to draw, handled by Canvas 决定画什么，比如是线还是正方形还是图片等等
2. How to draw, handled by Paint. 决定怎么画，比如是什么颜色的，线条粗细啊等等


一般来说画布只有一个，但是画笔却有很多，绘制View的时候，需要提前定义好这个画笔，避免在绘制的时候再生成, 在PieChart示例中会定义
这些画笔。

```
private void init() {
   mTextPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
   mTextPaint.setColor(mTextColor);
   if (mTextHeight == 0) {
       mTextHeight = mTextPaint.getTextSize();
   } else {
       mTextPaint.setTextSize(mTextHeight);
   }

   mPiePaint = new Paint(Paint.ANTI_ALIAS_FLAG);
   mPiePaint.setStyle(Paint.Style.FILL);
   mPiePaint.setTextSize(mTextHeight);

   mShadowPaint = new Paint(0);
   mShadowPaint.setColor(0xff101010);
   mShadowPaint.setMaskFilter(new BlurMaskFilter(8, BlurMaskFilter.Blur.NORMAL));

   ...
```

在继续View的绘制之前有必要看一下View在被绘制的时候的生命周期

| 分类 | 方法 | 描述 |
| ---- | -----|------|
| Creation | Constructors | 构造方法 |
| Creation | onFinishInflate() | 一个View从XML中初始化完之后调用 |
| Layout | onMeasure(int, int) | 用来决定这个View和它的子View的大小 |
| Layout | onLayout(boolean, int, int, int, int) | 决定子View的位置和大小 |
| Layout | onSizeChanged(int, int, int, int) | View 的大小变化的时候调用 |
| Drawing | onDraw(Canvas) | View被绘制的时候调用 |

一个View会先调用构造方法创建自己，如果是从XML文件中构造的还会调用 `onFinishInflate`方法，之后会调用 `onMeasure`
方法来决定自己的大小，如果是包含子View的Layout类型的View还需要在 `onLayout` 中计算子View的大小和位置。最后在View大小确定的情况
下会调用 `onSizeChanged` 方法，最后在绘制的时候调用 `onDraw` 方法。

`onMeasure` 是非常重要的方法，它会接收两个参数，这两个参数来自父View，是父View期望的大小，这个方法没有返回值，但是必须调用
`setMeasuredDimension()` 方法，来确定View的最终大小。它的两个参数是经过编码的int型，里面包含了模式和大小的信息，可以用`View.MeasureSpec`
来解析。

```
int widthMode = MeasureSpec.getMode(widthMeasureSpec);
int widthSize = MeasureSpec.getSize(widthMeasureSpec);
```

模式有如下三种 

1. AT_MOST 子View最大只能是父View指定的大小,设置"wrap_content"时候会导致这种效果
2. EXACTLY 父View已经决定了子View的大小，设置"match_parent"的时候等同于 EXACTLY + 父View大小
3. UNSPECIFIED 父View没有限制，子View可以任意大小（少见）

一般情况可以这么处理

```
      int desiredWidth = xxxxx; //计算期望的宽度
	    int desiredHeight = xxxxx; //计算期望的高度

	    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
	    int widthSize = MeasureSpec.getSize(widthMeasureSpec);
	    int heightMode = MeasureSpec.getMode(heightMeasureSpec);
	    int heightSize = MeasureSpec.getSize(heightMeasureSpec);

	    int width;
	    int height;

	    //Measure Width
	    if (widthMode == MeasureSpec.EXACTLY) {
	        //Must be this size
	        width = widthSize;
	    } else if (widthMode == MeasureSpec.AT_MOST) {
	        //Can't be bigger than...
	        width = Math.min(desiredWidth, widthSize);
	    } else {
	        //Be whatever you want
	        width = desiredWidth;
	    }

	    //Measure Height
	    if (heightMode == MeasureSpec.EXACTLY) {
	        //Must be this size
	        height = heightSize;
	    } else if (heightMode == MeasureSpec.AT_MOST) {
	        //Can't be bigger than...
	        height = Math.min(desiredHeight, heightSize);
	    } else {
	        //Be whatever you want
	        height = desiredHeight;
	    }

	    //MUST CALL THIS
	    setMeasuredDimension(width, height);
```

或者可以调用 [resolveSizeAndState()](http://developer.android.com/reference/android/view/View.html#resolveSizeAndState(int, int, int)) 这个方法
内部实现和上面是一样的。对于PieChart类来说，可以使用下面这些代码来决定自身大小。

```
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
   // Try for a width based on our minimum
   int minw = getPaddingLeft() + getPaddingRight() + getSuggestedMinimumWidth();
   int w = resolveSizeAndState(minw, widthMeasureSpec, 1);

   // Whatever the width ends up being, ask for a height that would let the pie
   // get as big as it can
   int minh = MeasureSpec.getSize(w) - (int)mTextWidth + getPaddingBottom() + getPaddingTop();
   int h = resolveSizeAndState(MeasureSpec.getSize(w) - (int)mTextWidth, heightMeasureSpec, 0);

   setMeasuredDimension(w, h);
}
```

真正的绘制需要在onDraw方法中进行，Canvas类提供了各种各样的方法来绘制图形 

```
protected void onDraw(Canvas canvas) {
   super.onDraw(canvas);

   // Draw the shadow
   canvas.drawOval(
           mShadowBounds,
           mShadowPaint
   );

   // Draw the label text
   canvas.drawText(mData.get(mCurrentItem).mLabel, mTextX, mTextY, mTextPaint);

   // Draw the pie slices
   for (int i = 0; i < mData.size(); ++i) {
       Item it = mData.get(i);
       mPiePaint.setShader(it.mShader);
       canvas.drawArc(mBounds,
               360 - it.mEndAngle,
               it.mEndAngle - it.mStartAngle,
               true, mPiePaint);
   }

   // Draw the pointer
   canvas.drawLine(mTextX, mPointerY, mPointerX, mPointerY, mTextPaint);
   canvas.drawCircle(mPointerX, mPointerY, mPointerSize, mTextPaint);
}
```

## 继承ViewGroup

有时候需要实现自定义的布局，大部分情况下需要继承ViewGroup，对于这种情况，需要格外的处理子View的事件，在`onMeasure`方法中手动调用子View的`measure`方法，来触发子View的`onMeasure`方法

```
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    	super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    	  int mHeight = View.MeasureSpec.getSize(heightMeasureSpec);
        int mWidth = View.MeasureSpec.getSize(widthMeasureSpec);
        setMeasuredDimension(mWidth, mHeight);
        
        int count = getChildCount();
        
        for (int i = 0; i < count; i++) {
            getChildAt(i).measure(MeasureSpec.makeMeasureSpec(mChildSize, MeasureSpec.EXACTLY),
                    MeasureSpec.makeMeasureSpec(mChildSize, MeasureSpec.EXACTLY));
        }
    }
```

同时需要重载 `onLayout` 来确定子View的布局,下面是一个横向的布局

```
@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
    	  int count = getChildCount();
        
        int childWidth = xxxx;
        int childHeight = xxxx;
        
        for (int i = 0; i < count; i++) {
            getChildAt(i).layout(childWidth*i, 0, childWidth*(i+1),childHeight);
        }
    	}
    }
```

## 其他

有的时候只需继承一个现有的View比如 Button就可以实现某些功能，有的时候需要组合其他的多种View，比如AutoCompleteTextView是一个EditText
和ListView的组合。


内容来源：

1. [Custom Components](http://developer.android.com/guide/topics/ui/custom-components.html)
2. [Creating Custom Views](http://developer.android.com/training/custom-views/index.html)






