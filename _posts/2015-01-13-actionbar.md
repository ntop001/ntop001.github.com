---
layout: post
title: "Android ActionBar"
description: ""
category: "知识整理"
tags: [Android,ActionBar]
---

ActionBar 其实就是以前App的TitleBar的演变版本，Google一时心血来潮给它添加了好多功能，如图是一个典型的ActionBar

<img src="http://developer.android.com/design/media/action_bar_pattern_overview.png" width="600"/>

## 导航模式

<img src="http://developer.android.com/design/media/action_bar_basics.png" width="600"/>

1. App icon 左上角的点击App Icon导航，点击后回退或者返回上一级
2. View control 比如ActionBar上可以下拉一个SpinnerList或者ActionBar变身Tab模式
2. Action Button 通过ActionBar上的图标导航
3. Action overflow 通过最右的菜单（竖排三个点 ）导航
4. Contextual Action Bars 在Gmail上长按某个list的item就会出现这个bar，其实他是一个临时的ActionBar覆盖在了原来的ActionBar上面

关于ActionBar每个功能的详细解释可以看[这里](http://developer.android.com/design/patterns/actionbar.html)。事实上从Android 5.0开始ActionBar的众多功能已经被弱化或者移除，比如"View Control"功能已经废弃，App Icon的位置也被替换成一个Home菜单（5.0系统上的许多原生App都是三道横线）。

## 使用

ActionBar是Android3.0版本和Fragment一起引入的控件，如果使用为了兼容，需要使用SupportV7包提供的ActionBar。

1. 继承V7包提供的 ActionBarActivity, 
2. 设置主题为 Theme.AppCompat

```
//MainActivity.java
public class MainActivity extends ActionBarActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
//.xml
<activity android:theme="@style/Theme.AppCompat.Light" ... >
```

这时候ActionBar应该出现了，但是如何添加 "Action Button" 的按钮呢？

1. 在`res/menu/xxxx.xml` 下面添加菜单描述文件
2. 在`onCreateOptionsMenu` 方法中填充菜单
3. 在`onOptionsItemSelected` 处理菜单事件

```
<menu xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:app="http://schemas.android.com/apk/res-auto">
    <item android:id="@+id/action_search"
          android:icon="@drawable/ic_action_search"
          android:title="@string/action_search"/>
    <item android:id="@+id/action_compose"
          android:icon="@drawable/ic_action_compose"
          android:title="@string/action_compose"
          app:showAsAction="ifRoom"
          />
</menu>
```
注意第一个菜单项，会出现在点击ActionBar最右边的的三个点出现的菜单选项中，但是第二菜单项由于设置了 `app:showAsAction="ifRoom"` 它会以按钮图标的形式出现在 ActionBar上面。

```
//填充菜单
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    // Inflate the menu items for use in the action bar
    MenuInflater inflater = getMenuInflater();
    inflater.inflate(R.menu.main_activity_actions, menu);
    return super.onCreateOptionsMenu(menu);
}
//响应事件
@Override
public boolean onOptionsItemSelected(MenuItem item) {
    // Handle presses on the action bar items
    switch (item.getItemId()) {
        case R.id.action_search:
            openSearch();
            return true;
        case R.id.action_compose:
            composeMessage();
            return true;
        default:
            return super.onOptionsItemSelected(item);
    }
}
```

## 修改ActionBar图标

如果要修改ActionBar图标可以在代码中直接调用方法设置，也可以在 `<application>` 或者 `<activity>` XML 元素中，设置 `logo`或者`icon` 属性来实现。在 `<application>` 中设置会起到全局的作用。


## using split action bar

<img src="http://developer.android.com/images/ui/actionbar-splitaction@2x.png" width="600" />

其实就是这种效果，ActionBar上面的所有 "Action Button" 都被移到了页面的下方，文档的说法是考虑有些手机屏幕比较窄，移动到下面
好看一点。但是现在的主流手机屏幕都大的惊人，所以这种功能好像没啥必要。

1. 给`<activity>` 或者 `<application>` 元素添加 `uiOptions="splitActionBarWhenNarrow"` 属性。（API14才有，以前版本自动忽略）
2. 给`<activity>` 添加 meta-data 以兼容老的版本

```
<manifest ...>
    <activity uiOptions="splitActionBarWhenNarrow" ... >
        <meta-data android:name="android.support.UI_OPTIONS"
                   android:value="splitActionBarWhenNarrow" />
    </activity>
</manifest>
```

## ActionView 

<img src="http://developer.android.com/images/ui/actionbar-searchview@2x.png" width="600" />

ActionView就是这种效果，点击搜索按钮之后，在ActionBar上会出现一个EditText输入框，可以直接在里面输入搜索的关键词（Android文档的网页上提供的搜索按钮也是这种效果）。自定义ActionView需要 `actionLayout(xml)` 或者 `actionViewClass(java)` 属性来设置对应的View，比如系统自带的 [SearchView](http://developer.android.com/reference/android/support/v7/widget/SearchView.html) 是这么设置的。

可以在 `onCreateOptionsMenu` 函数触发的时候获取到自定义的ActionView并做出一些配置。

```
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.main_activity_actions, menu);
    MenuItem searchItem = menu.findItem(R.id.action_search);
    SearchView searchView = (SearchView) MenuItemCompat.getActionView(searchItem);
    // Configure the search info and add any event listeners
    ...
    return super.onCreateOptionsMenu(menu);
}
```

一般情况下，都会提前设置 `showAsAction="ifRoom|collapseActionView"` 把ActionView隐藏起来，节省ActionBar的空间。另外点击ActionButton之后会自动展开ActionView，这时候不要自行处理这个菜单点击事件，否则会导致系统无法展开ActionView。如果需要在ActionView展开的时候做出一些响应，应该实现 `OnActionExpandListener`回调。

```
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.options, menu);
    MenuItem menuItem = menu.findItem(R.id.actionItem);
    ...

    // When using the support library, the setOnActionExpandListener() method is
    // static and accepts the MenuItem object as an argument
    MenuItemCompat.setOnActionExpandListener(menuItem, new OnActionExpandListener() {
        @Override
        public boolean onMenuItemActionCollapse(MenuItem item) {
            // Do something when collapsed
            return true;  // Return true to collapse action view
        }

        @Override
        public boolean onMenuItemActionExpand(MenuItem item) {
            // Do something when expanded
            return true;  // Return true to expand action view
        }
    });
}
```


## ActionProvider

ActionProvider 可以看做是一个功能更强大的ActionView，比较起来有如下的区别

1. ActionButton图标 在ActionView的实现上，这个图标是不变的，但是ActionProvider上它会可以修改这个图标
2. 自定义的UI部分 ActionView的UI只能局限在ActionBar上，并且受到系统控制（展开还是收缩），但是ActionProvider的UI 可以完全自己控制

使用 ActionProvider （比如系统已经提供的ShareActionProvider）只需要在xml文件中设置就行了

```
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:yourapp="http://schemas.android.com/apk/res-auto" >
    <item android:id="@+id/action_share"
          android:title="@string/share"
          yourapp:showAsAction="ifRoom"
          yourapp:actionProviderClass="android.support.v7.widget.ShareActionProvider"
          />
    ...
</menu>
```

接下来需要做的事情是在点击每个分享项目的时候提供一个ShareIntent

```
private ShareActionProvider mShareActionProvider;

@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.main_activity_actions, menu);

    // Set up ShareActionProvider's default share intent
    MenuItem shareItem = menu.findItem(R.id.action_share);
    mShareActionProvider = (ShareActionProvider)
            MenuItemCompat.getActionProvider(shareItem);
    mShareActionProvider.setShareIntent(getDefaultIntent());

    return super.onCreateOptionsMenu(menu);
}

/** Defines a default (dummy) share intent to initialize the action provider.
  * However, as soon as the actual content to be used in the intent
  * is known or changes, you must update the share intent by again calling
  * mShareActionProvider.setShareIntent()
  */
private Intent getDefaultIntent() {
    Intent intent = new Intent(Intent.ACTION_SEND);
    intent.setType("image/*");
    return intent;
}
```

如果自定义自己的ActionProvider需要继承自ActionProvider类

```
    class MyProvider extends ActionProvider {
        public MyProvider(Context context){
            super(context);
        }

        @Override
        public View onCreateActionView() {
            return null;
        }
    }
```

## Navigation Mode

导航模式有三种 

