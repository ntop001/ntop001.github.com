---
layout: post
title: "Android Fragment总结"
description: ""
category: "知识整理"
tags: [Android,Fragment]
---

 在过去很长的一段时间里AndroidApp的界面组成都是通过Activity来完成的，如果把MVC套用在上面，那么XML布局文件负责的是View层，Activity就是
 处理Modlel和View的Controller，Android4.0之后的很多App都开始使用Fragment来构建了，Fragment是一个很奇葩的存在，放在XML文件中可以做
 View元素，在代码实现中确是一个实实在在的Controller。现在的大部分App功能，比如侧滑菜单，Tab页面都是使用Fragment来构建的。
 
 > You can think of a fragment as a modular section of an activity, which has its own lifecycle, receives its own input events, and which you can add or remove while the activity is running (sort of like a "sub activity" that you can reuse in different activities). 
 
 ## 构建Fragment
 
 Fragment是Android3.0才引入的东西，如果使用的话，需要用Android的兼容包提供的Fragment实现([Support Library](http://developer.android.com/tools/support-library/index.html))
 ,Android兼容包提供了强大的功能，包括以后可能会用到的ActionBar、ViewPager等等现在比较流行的Android设计元素都需要使用兼容包来实现。
 
 ```
    import android.os.Bundle;
    import android.support.v4.app.Fragment;
    import android.view.LayoutInflater;
    import android.view.ViewGroup;
    
    public class ArticleFragment extends Fragment {
        //这是一个很重要的方法，用来给Fragment填充布局
        @Override
        public View onCreateView(LayoutInflater inflater, ViewGroup container,
            Bundle savedInstanceState) {
            // Inflate the layout for this fragment
            return inflater.inflate(R.layout.article_view, container, false);
        }
    }
 ```
 这样就可以实现一个Fragment了，Fragment可以像UI元素一样直接嵌入到布局文件中
 
 ```
 //res/layout-large/news_articles.xml
 <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent">
    <fragment android:name="com.example.android.fragments.HeadlinesFragment"
              android:id="@+id/headlines_fragment"
              android:layout_weight="1"
              android:layout_width="0dp"
              android:layout_height="match_parent" />
    <fragment android:name="com.example.android.fragments.ArticleFragment"
              android:id="@+id/article_fragment"
              android:layout_weight="2"
              android:layout_width="0dp"
              android:layout_height="match_parent" />
</LinearLayout>
//
import android.os.Bundle;
import android.support.v4.app.FragmentActivity;

public class MainActivity extends FragmentActivity {
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.news_articles);
    }
}
 ```
 
在V4包中Activity需要继承自FragmentActivity，如果使用的是V7的包可以继承ActionBarActivity它是FragmentActivity的一个子类。

直接把Fragment写入XML文件比较简单，但是有个缺点就是Fragment不能动态的添加和移除，如果需要这种效果的话，可以使用代码来实现。
 FragmentManager类提供了一系列的增删改方法（add,remove,replace）来动态操作Fragment。动态改变Fragment的一个前提是在Activity的
 布局文件中必须提供一个ViewGroup容器来加载Fragment。同时需要在Activity的onCreate方法里面初始化一个Fragment，这样在接下来的
 操作中就可以无忧的调用remove、replace（=remove + add）方法了。
 
```
//res/layout/news_articles.xml: 提供Fragment的容易
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/fragment_container"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
//在onCreate方法中初始化这个Fragment
import android.os.Bundle;
import android.support.v4.app.FragmentActivity;

public class MainActivity extends FragmentActivity {
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.news_articles);

        // Check that the activity is using the layout version with
        // the fragment_container FrameLayout
        if (findViewById(R.id.fragment_container) != null) {

            // However, if we're being restored from a previous state,
            // then we don't need to do anything and should return or else
            // we could end up with overlapping fragments.
            if (savedInstanceState != null) {
                return;
            }

            // Create a new Fragment to be placed in the activity layout
            HeadlinesFragment firstFragment = new HeadlinesFragment();
            
            // In case this activity was started with special instructions from an
            // Intent, pass the Intent's extras to the fragment as arguments
            firstFragment.setArguments(getIntent().getExtras());
            
            // Add the fragment to the 'fragment_container' FrameLayout
            getSupportFragmentManager().beginTransaction()
                    .add(R.id.fragment_container, firstFragment).commit();
        }
    }
}
```

动态的替换之前的Fragment

```
  // Create fragment and give it an argument specifying the article it should show
  ArticleFragment newFragment = new ArticleFragment();
  Bundle args = new Bundle();
  args.putInt(ArticleFragment.ARG_POSITION, position);
  newFragment.setArguments(args);
  
  FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();
  
  // Replace whatever is in the fragment_container view with this fragment,
  // and add the transaction to the back stack so the user can navigate back
  transaction.replace(R.id.fragment_container, newFragment);
  transaction.addToBackStack(null);
  
  // Commit the transaction
  transaction.commit();
```

注意，每次Fragment变动的时候我们都会获取 `FragmentTransaction` 类，Fragment的每种操作都封装在这个类里面，调用commit()方法后，
操作才真正生效。`FragmentTransaction`类提供了一个方法 `addToBackStack` 可以将这次变动入栈，调用 `getSupportFragmentManager().popBackStack()`
或者按下back键都会恢复到之前的Fragment，这个有点像Activity的管理。


## 给ActionBar添加菜单

每个Fragment都可以给ActionBar添加自己的菜单，需要先在 onCreate方法中调用 `setHasOptionsMenu()` 告诉宿主Activity它需要改动
ActionBar，然后实现 ` onCreateOptionsMenu()` 方法，填充自己的菜单。
 
## Fragment 和 Activiy 之间的通信

Fragment 如果需要调用Activity中的方法，可以通过定义接口来完成，然后在Activity中实现这个方法。
 
```
// 在Fragment加载到Activity的时候获取接口
public class HeadlinesFragment extends ListFragment {
    OnHeadlineSelectedListener mCallback;

    // Container Activity must implement this interface
    public interface OnHeadlineSelectedListener {
        public void onArticleSelected(int position);
    }

    @Override
    public void onAttach(Activity activity) {
        super.onAttach(activity);
        
        // This makes sure that the container activity has implemented
        // the callback interface. If not, it throws an exception
        try {
            mCallback = (OnHeadlineSelectedListener) activity;
        } catch (ClassCastException e) {
            throw new ClassCastException(activity.toString()
                    + " must implement OnHeadlineSelectedListener");
        }
    }
    
    ...
}
//在Activity中实现这个接口
public static class MainActivity extends Activity
        implements HeadlinesFragment.OnHeadlineSelectedListener{
    ...
    
    public void onArticleSelected(int position) {
        // The user selected the headline of an article from the HeadlinesFragment
        // Do something here to display that article
    }
}
```
 
 如果Activity需要调用Fragment中的方法，可以直接通过FragmentActivity的 `getSupportFragmentManager().findFragmentById()` 查找到
 相关的Fragment然后调用它的Public方法。
 
 ## Fragment的生命周期
 
 ![Fragment 生命周期](http://developer.android.com/images/activity_fragment_lifecycle.png)
 
 这张图显示Fragment的生命周期，需要注意的 `onCreateView` 方法的调用是在`onCreate` 和 `onStart`之间的，`onDestroyView` 是在
 `onStop` 和 `onDestroy` 之间，如果一个Fragment被后面一个Fragment取代了，此时会发生两种结果
 
 1. 调用了 `transaction.addToBackStack` 方法，这个时候Fragment会调用 `onDestroyView` 方法，但是不会Destroy，也就是UI被回收了，但是Fragment实例还在
 2. 没有调用 `transaction.addToBackStack` 方法，那么Fragment会被直接摧毁，走完完整的生命周期。
 
 另外，按照Fragment的定义 onResume 和 onPause 方法是在Fragment 可见到不可以之间的时候调用，但是在ViewPager中的Fragment却不遵循
 这个规律。好像是support版本的一个[bug](http://stackoverflow.com/questions/18301979/onresume-not-called-on-viewpager-fragment-when-using-custom-loader)
 
 关于Fragment栈完全是由宿主的Activity管理的，如果Activity销毁了，那么她里面包含的所有Fragment都会销毁，一般情况下如果不调用
 `transaction.addToBackStack` 方法，这个栈就是空的。
 
 以上内容来自Android文档
 
 1. [Building a Dynamic UI with Fragments](http://developer.android.com/training/basics/fragments/index.html)
 2. [API Guides](http://developer.android.com/guide/components/fragments.html)
 
 
