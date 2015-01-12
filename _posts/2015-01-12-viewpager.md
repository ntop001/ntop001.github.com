---
layout: post
title: "Android ViewPager"
description: ""
category: "知识整理"
tags: [Android,Fragment,ViewPager]
---

自从原生Android使用虚拟按键之后，在Android的标准设计风格中就再也看不到底部Tab的实现了，这样的设计会造成两个Bar的问题（一个是系统
虚拟按键，一个是底部Tab）视觉效果很差，丑。在Android上的一个取代方案是使用ViewPager+Fragment来实现这种效果。Fragment相关的知识在
[这里](http://ntop001.github.io/%E7%9F%A5%E8%AF%86%E6%95%B4%E7%90%86/2015/01/12/android-fragment)已经做了介绍。本篇主要讲一下
ViewPager。

## 基本使用

ViewPager是一个继承自ViewGroup的UI控件，它提供了类似Launcher主屏一样横向分页，滑动切换页面的效果。它是SupportV4包中提供的一个控件。
使用非常简单。

在布局文件中布局 

```
<?xml version="1.0" encoding="utf-8"?>
<android.support.v4.view.ViewPager
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/pager"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

实现一个适配机器，每个页面都都是`DemoObjectFragment` 实例。ViewPager的Fragment适配器实现非常简单，仅仅需要实现三个方法。

```
//DemoCollectionPagerAdapter.java
public class DemoCollectionPagerAdapter extends FragmentStatePagerAdapter {
    public DemoCollectionPagerAdapter(FragmentManager fm) {
        super(fm);
    }

    //返回一个Fragment, Fragment 被ViewPager管理了，不需要操心Fragment的复用问题
    @Override
    public Fragment getItem(int i) {
        Fragment fragment = new DemoObjectFragment();
        Bundle args = new Bundle();
        // Our object is just an integer :-P
        args.putInt(DemoObjectFragment.ARG_OBJECT, i + 1);
        fragment.setArguments(args);
        return fragment;
    }

    //确定分页数量
    @Override
    public int getCount() {
        return 100;
    }
    //返回每个页面的标题
    @Override
    public CharSequence getPageTitle(int position) {
        return "OBJECT " + (position + 1);
    }
    
    // Instances of this class are fragments representing a single
    // object in our collection.
    public static class DemoObjectFragment extends Fragment {
      public static final String ARG_OBJECT = "object";
    
      @Override
      public View onCreateView(LayoutInflater inflater,
              ViewGroup container, Bundle savedInstanceState) {
          // The last two arguments ensure LayoutParams are inflated
          // properly.
          View rootView = inflater.inflate(
                  R.layout.fragment_collection_object, container, false);
          Bundle args = getArguments();
          ((TextView) rootView.findViewById(android.R.id.text1)).setText(
                  Integer.toString(args.getInt(ARG_OBJECT)));
          return rootView;
    }
}
```
在Activity中给ViewPager设置适配器

```
public class CollectionDemoActivity extends FragmentActivity {
    // When requested, this adapter returns a DemoObjectFragment,
    // representing an object in the collection.
    DemoCollectionPagerAdapter mDemoCollectionPagerAdapter;
    ViewPager mViewPager;

    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_collection_demo);

        // ViewPager and its adapters use support library
        // fragments, so use getSupportFragmentManager.
        mDemoCollectionPagerAdapter =
                new DemoCollectionPagerAdapter(
                        getSupportFragmentManager());
        mViewPager = (ViewPager) findViewById(R.id.pager);
        mViewPager.setAdapter(mDemoCollectionPagerAdapter);
    }
}
```

ViewPager 提供两种类型的FragmentAdapter用来加载Fragment

1. FragmentPagerAdapter 这种Adapter适合于固定数量且分页较少的情况
2. FragmentStatePagerAdapter 适用于分页数量不定的情况，它会立刻销毁看不到的Fragment来控制内存

## 给ViewPager添加Tab效果

单纯的ViewPager是没有Tab的，目前有三种常用方式给ViewPager添加Tab

1. 利用ActionBar的Tabs功能（ActionBar的导航功能真是繁复众多）
2. 利用V4包提供的 TitleStrip
3. 利用第三方提供的Tab组件结合ViewPager使用

#### ActionBar Tabs

要解锁ActionBar的Tabs功能需要先设置ActionBar的导航模式为 `NAVIGATION_MODE_TABS` ,然后创建`ActionBar.Tab`实例，并设置好
Tab触发的监听 `ActionBar.TabListener`。

```
@Override
public void onCreate(Bundle savedInstanceState) {
    final ActionBar actionBar = getActionBar();
    ...

    // Specify that tabs should be displayed in the action bar.
    actionBar.setNavigationMode(ActionBar.NAVIGATION_MODE_TABS);

    // Create a tab listener that is called when the user changes tabs.
    ActionBar.TabListener tabListener = new ActionBar.TabListener() {
        public void onTabSelected(ActionBar.Tab tab, FragmentTransaction ft) {
            // show the given tab
        }

        public void onTabUnselected(ActionBar.Tab tab, FragmentTransaction ft) {
            // hide the given tab
        }

        public void onTabReselected(ActionBar.Tab tab, FragmentTransaction ft) {
            // probably ignore this event
        }
    };

    // Add 3 tabs, specifying the tab's text and TabListener
    for (int i = 0; i < 3; i++) {
        actionBar.addTab(
                actionBar.newTab()
                        .setText("Tab " + (i + 1))
                        .setTabListener(tabListener));
    }
}
```

但是上面仅仅显示了ActionBar的Tab，我们需要ViewPager的配合才能在切换Tab的时候ViewPager页面也跟着动，反过来ViewPager滑动的时候
Tab也动。

```
    // 在Tab切换的时候改变ViewPager
    ActionBar.TabListener tabListener = new ActionBar.TabListener() {
        public void onTabSelected(ActionBar.Tab tab, FragmentTransaction ft) {
            // When the tab is selected, switch to the
            // corresponding page in the ViewPager.
            mViewPager.setCurrentItem(tab.getPosition());
        }
        ...
    };
    //在ViewPager切换的时候改变Tab
    mViewPager = (ViewPager) findViewById(R.id.pager);
    mViewPager.setOnPageChangeListener(
            new ViewPager.SimpleOnPageChangeListener() {
                @Override
                public void onPageSelected(int position) {
                    // When swiping between pages, select the
                    // corresponding tab.
                    getActionBar().setSelectedNavigationItem(position);
                }
            });
```

看上去是不是很疑惑，Tab页面给人的感觉是一个一体的东西，但是在我们的实现上却是两个组件配合完成的。

#### Title Trips

这种实现效果其实很不好看，但是实现起来异常简单，只需要在布局文件中添加一个UI元素即可。（缺少配图）

```
<android.support.v4.view.ViewPager
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/pager"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <android.support.v4.view.PagerTitleStrip
        android:id="@+id/pager_title_strip"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_gravity="top"
        android:background="#33b5e5"
        android:textColor="#fff"
        android:paddingTop="4dp"
        android:paddingBottom="4dp" />

</android.support.v4.view.ViewPager>
```

#### 使用第三方实现

使用ActionBar的实现容易将ActionBar和ViewPager绑定的太紧，网络上可以找到很多好看的Tab实现。我比较喜欢的是 [PagerSlidingTabStrip](https://github.com/astuetz/PagerSlidingTabStrip)
使用起来非常简单，它只是一个UI元素可以直接布局到XML文件中，然后在代码中在稍作设置绑定ViewPager即可。

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools" android:id="@+id/pager"
    xmlns:iviews="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    >
    <com.huoyu.huoyu.iviews.PagerSlidingTabStrip
        android:id="@+id/tabs"
        android:layout_width="match_parent"
        android:layout_height="48dip"
        iviews:pstsIndicatorHeight="2dp"
        iviews:pstsUnderlineHeight="0dp"
        iviews:pstsDividerColor="@android:color/transparent"
        />

    <android.support.v4.view.ViewPager
        android:id="@+id/viewpager"
        android:layout_width="match_parent" android:layout_height="match_parent"
        tools:context="com.huoyu.huoyu.MainActivity">
    </android.support.v4.view.ViewPager>

</LinearLayout>
```

在代码中绑定Tab和ViewPager

```
    ViewPager mViewPager = (ViewPager)root.findViewById(R.id.browser_viewpager);
    mViewPager.setAdapter(mSectionsPagerAdapter);

    final PagerSlidingTabStrip tabs = (PagerSlidingTabStrip) root.findViewById(R.id.tabs);
    tabs.setViewPager(mViewPager);
```

大功告成，拥有ActionBar Tabs的美观，也有TitleTrips的便捷。

## 使用ViewPager滚动图片

有很多App会在页面的顶端设置一组快速浏览的图片，左右滑动即可切换图片，实现这种效果明显使用ViewPager比较方便，但是这时候不能
在用FragmentAdapter了，这太重量级了，如果切换的就是ImageView效率会提升很多。那么需要继承 `PagerAdapter` 类，这个类提供4个必须
实现的方法。

1. `getCount()` 返回分页数量
2. `isViewFromObject` 判断一个页面是不是关联了一个key
3. `instantiateItem(ViewGroup, int)` 返回一个View,并需要把这个View添加到ViewGroup中
4. `destroyItem(ViewGroup, int, Object)` 从ViewGroup中移除这个View，并销毁View

这种 Adapter 是不是看上去很诡异和之前见到的都不一样，这个Adapter并没有实现View的复用而是描述了一次View替换的过程。

一个简单的实现是这样的（没有任何优化的机制，View会不断的创建和销毁）。

```
class ProductSnapshotAdapter extends PagerAdapter {
        private int[] imgs = {R.drawable.meinv, R.drawable.meinv2};

        ProductSnapshotAdapter(){
        }

        @Override
        public int getCount() {
            return imgs.length;
        }

        @Override
        public boolean isViewFromObject(View view, Object object) {
            return view==(View)object;
        }

        @Override
        public Object instantiateItem(ViewGroup container, int position) {
            ImageView iv = new ImageView(ProductActivity.this);
            iv.setImageResource(imgs[position]);
            iv.setScaleType(ImageView.ScaleType.CENTER_CROP);
            iv.setLayoutParams(new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
            container.addView(iv);
            return iv;
        }

        @Override
        public void destroyItem(ViewGroup container, int position, Object object) {
            container.removeView((View)object);
        }
    }
```

TODO:补充ViewPager内部的实现细节

内容来自Android文档：

1. [ViewPager](http://developer.android.com/reference/android/support/v4/view/ViewPager.html)
2. [PagerAdapter](http://developer.android.com/reference/android/support/v4/view/PagerAdapter.html)
3. [Creating Swipe Views with Tabs](http://developer.android.com/training/implementing-navigation/lateral.html)
