---
layout: post
title: "Android listview 和 GridView 总结"
description: 
category: "知识整理"
tags: [Android,ListView,GridView]
---

ListView算是Android开发中使用的最多的纯UI控件，下面会记录一些开发过程中用到的功能。

## 一般使用

一般使用ListView只需要实现一个Adapter填充数据就行了。
```
class MyAdapter extends BaseAdapter {

		@Override
		public int getCount() {
			// TODO Auto-generated method stub
			return 0;
		}

		@Override
		public Object getItem(int arg0) {
			// TODO Auto-generated method stub
			return null;
		}

		@Override
		public long getItemId(int position) {
			// TODO Auto-generated method stub
			return 0;
		}

		@Override
		public View getView(int position, View convertView, ViewGroup parent) {
			// TODO Auto-generated method stub
			return null;
		}
		
	}
```
没有需要特别说明的地方，但是ListView的主题永远围绕在性能优化上面，Android文档给了几条优化方案

1. 使用View重用机制
2. 使用ViewHolder避免每次查询子View（AndroidL中新增的RecyclerView强制实现了ViewHolder的模式）
3. 在ListView快速滚动的时候，使用临时图片替换真实图片

我们在做加载很多图片的ListView的时候还会使用[UIL](https://github.com/nostra13/Android-Universal-Image-Loader)这样的图片加载库，
它内部实现了图片的异步读取，图片裁切和图片缓存，使整个图片的加载非常迅速。

## 小技巧

比如要做一个像微信那样的对话窗口，会用到两种子View，一种是“头像+对话” 另一种是“对话+头像” 对于这种需求，ListView需要同时缓存
这两种类型的View，我们需要实现如下的接口

```
    @Override
		public int getItemViewType(int position) {
			// TODO Auto-generated method stub
			return super.getItemViewType(position);
		}

		@Override
		public int getViewTypeCount() {
			// TODO Auto-generated method stub
			return super.getViewTypeCount();
		}
```

这两个接口告诉ListView需要缓存几种不同类型的子View，这样ListView就可以适当的重复利用子View了。

还有一些需求是Disable掉某些行，实现这种需求也很简单,实现这两个接口，就可以使某些行无法点击了

```
    @Override
    public boolean areAllItemsEnabled() {
        return false;
    }

    @Override
    public boolean isEnabled(int position) {
        return false;
    }
```

另外还有一种需求是把ListView放在ScrollView里面，这种需求一般网上的建议都是“最好不要这么做” 但是如果这么设计产品比较好（Google的
原生App很多都是这样的）那为什么不这么做呢？有两种实现方式

1. 把ListView做成固定大小的（全部显示，有多少子View都显示出来）这样做对于比较小的ListView比较适合，当List比较小一屏完全可以装下的情况下ListView的子View复用功能根本用不上，所以完全没有问题。
2. 利用ListView的HeaderView 和 FooterView， 整个页面使用ListView来展示，可以把ListView想象成一个拥有List功能的ScrollView，把需要放在ScrollView中的布局都放在ListView的HeaderView或者FooterView里面

对于第一种方式其实就是使ListView支持`wrap_content`布局,有一段常用的代码片段来增加这个功能

```
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int heightSpec;

        if (getLayoutParams().height == LayoutParams.WRAP_CONTENT) {
            // The great Android "hackatlon", the love, the magic.
            // The two leftmost bits in the height measure spec have
            // a special meaning, hence we can't use them to describe height.
            heightSpec = MeasureSpec.makeMeasureSpec(Integer.MAX_VALUE >> 2, MeasureSpec.AT_MOST);
        }
        else {
            // Any other height should be respected as is.
            heightSpec = heightMeasureSpec;
        }

        super.onMeasure(widthMeasureSpec, heightSpec);
    }
```

重载 `onMeasure` 方法，使之支持WrapContent属性。

对于第二种方式，是目前我在项目中广泛使用的一种，但是对于比较复杂的布局还是无能为力。

给ListView中子控件添加按钮，这个设计很常见，比如微博列表里面中每个Item下面都有“赞”、“评论”和“分享”三个按钮，点击按钮和点击整个ListView的子View触发的是不同的事件。在子View中添加Button的处理响应事件，我看到好多人处理的都很怪异，比如在`getView`函数中new一个OnClick接口。有一种实现可以这样做：

```
	//布局文件(listitem.xml)
	<?xml version="1.0" encoding="UTF-8"?>
	<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    android:layout_width="fill_parent"
	    android:layout_height="fill_parent" >
	
	    <ImageView
	        android:id="@+id/add_selector_image"
	        android:layout_width="fill_parent"
	        android:layout_height="100dp"
	        android:scaleType="centerCrop" />
	
	    <ImageButton
	        android:id="@+id/add_selector_check"
	        android:clickable="false"
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:layout_alignParentRight="true"
	        android:layout_alignParentTop="true"
	        android:layout_marginRight="3dp"
	        android:layout_marginTop="3dp"
	        android:background="@null"/>
	</RelativeLayout>
	//代码
	@Override
	public View getView(int position, View convertView, ViewGroup parent) {
		View view = convertView;
		ViewHolder vholder = null;
		if(view == null){
			view = LayoutInflater.from(context).inflate(R.layout.listitem, null);
			ViewHolder holder = new ViewHolder();
			view.setTag(holder);
			vholder = holder;
			
			holder.ImageIamge = (ImageView)view.findViewById(R.id.add_selector_image);
			holder.ImageCheck = (ImageButton)view.findViewById(R.id.add_selector_check);
			CheckListener listener = new CheckListener();
			holder.ImageCheck.setOnClickListener(listener);
			holder.Listener = listener;
		}else{
			vholder = (ViewHolder)view.getTag();
		}
		
		vholder.Listener.Position = position;
		
		return view;
	}
	
	class ViewHolder {
		public ImageView ImageIamge;
		public ImageButton ImageCheck;
		public CheckListener Listener;
	}
	
	class CheckListener implements OnClickListener {
		public int Position;
		
		@Override
		public void onClick(View v) {
			
		}
	}
```

这样给界面上存在的每个Item设置一个 `OnClickListener`，在每次View复用的时候从新设置被点击的View的位置，原理和View复用一样。


## GridView相关

处理GridView的技巧和ListView都差不多，只是GridView是没有HeaderView和FooterView的，但是优化的技巧是一样的，做GridView中经常遇到的一个问题是需要设定子View的大小，比如设定子View显示成正方形，但是宽度却要自适应的情况。设定GridView中每个Grid的宽度自适应只要设置如下两个标签就可以了

```
android:numColumns="4"
android:stretchMode="columnWidth"
```

如上设置Grid显示4列，宽度自动拉伸到合适。但是这时候会发现高度变得很奇怪。一个简单的方法是自定义一个正方形的View，作为子View，比如

```
public class SquareImageView extends ImageView {
    public SquareImageView(Context context) {
        super(context);
    }

    public SquareImageView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public SquareImageView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        int width = getMeasuredWidth();
        setMeasuredDimension(width, width);
    }
}
```

修改ImageView的 `onMeasure` 方法，是他的高度和宽度是同样的值。。。如果你的子View是一个布局，比如`RelativeLayout`只要用相似的方法修改宽高即可。


