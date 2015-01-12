---
layout: post
title: "Android图库读取"
description: ""
category: "知识整理"
tags: [Android,App]
---

花了一天的时间才搞定一个项目中图片选的功能，因为之前都是用iPhone的，豁然发现Android的图库(Gallery)中包含各种各样的图片煞是费解。在iPhone上
所有用相机拍摄的图片都会默认进入一个叫“照片”的应用，其他应用生成的图片默认是在自己的私有目录下，需要专门选择保存到“照片”应用，才会
在“照片”中找到。但是Android不是这样的，在Android中打开“图库”应用后会发现，SD Card 卡上的所有图片（包括照片、截图、广告图片还有各种App
自己生成的杂图）都会出现在“图库”中。各有利弊吧，但是我还是喜欢iPhone的管理方式比较干净，不需要看到很多不想看到的图片（比如一些广告
图片，非常碍眼）。

一般情况下，读取照片在Android上面发送一个Intent就可以了

```
  Intent intent=new Intent(Intent.ACTION_GET_CONTENT);//ACTION_OPEN_DOCUMENT
  intent.addCategory(Intent.CATEGORY_OPENABLE);
  intent.setType("image/jpeg");
  //不同版本Android返回结果略有差别，需要区分处理
  if(android.os.Build.VERSION.SDK_INT>=android.os.Build.VERSION_CODES.KITKAT){
      startActivityForResult(intent, 0);
  }else{
      startActivityForResult(intent, 1);
  }
```

但是这样每次只能选择一张图片，而且待选的图片中包含各种杂图。关于如何选择多张图片在StackOverflow上有一段[讨论](http://stackoverflow.com/questions/19585815/select-multiple-images-from-android-gallery)
大意是可以设置 `intent.putExtra(Intent.EXTRA_ALLOW_MULTIPLE,true);` 就可以选择多张图片了，但是图库本身不支持多选，所以如果需要
实现这种效果还是自己实现吧。


自己实现图片选择的功能，可以自定义UI风格，添加新的功能，好处多多。在Android中MediaStore类负责管理手机上的图片、音频和视频资源的。
，图片资源的读取需要 `MediaStore.Images` 类。

```
  Uri mImageUri = MediaStore.Images.Media.EXTERNAL_CONTENT_URI;
  String[] projection = null;
  String selection = MediaStore.Images.Media.MIME_TYPE + "=? or "+ MediaStore.Images.Media.MIME_TYPE + "=?";
  String[] selectionArgs = new String[] { "image/jpeg", "image/png" };
  String sortOrder = MediaStore.Images.Media.DATE_MODIFIED;

  ContentResolver mContentResolver = context.getContentResolver();
  Cursor mCursor = mContentResolver.query(mImageUri, projection,selection, selectionArgs,sortOrder);
```

这样就可以读取手机上的图片资源了，但是如果循环打印这些资源就会发现读取到的图片和图库里面的图片一个德性，各种杂图都掺和在里面了。
如果查看`MediaStore.Images.ImageColumns`类会看到图库存储的列 

|类型| 字段| 说明 |
|----|----|------|
| String	| BUCKET_DISPLAY_NAME	| The bucket display name of the image.|
|String	| BUCKET_ID	| The bucket id of the image.|
|String |	DATE_TAKEN	| The date & time that the image was taken in units of milliseconds since jan 1, 1970.|
|String	| DESCRIPTION	| The description of the image Type: TEXT|
|String	| IS_PRIVATE	| Whether the video should be published as public or private Type: INTEGER|
|String	| LATITUDE	| The latitude where the image was captured.|
|String	| LONGITUDE	| The longitude where the image was captured.|
|String	| MINI_THUMB_MAGIC |	The mini thumb id.|
|String	| ORIENTATION |	The orientation for the image expressed as degrees.|
|String	| PICASA_ID |	The picasa id of the imageType: TEXT|

有一个字段 `BUCKET_DISPLAY_NAME` 这个字段就是图库在显示文件夹的时候显示的文字，照片的文件夹叫 "Camera", 所以只要添加过滤条件
就可以筛选出所有照片了。

```
  String selection = MediaStore.Images.Media.BUCKET_DISPLAY_NAME + " = ?";
  String[] selectionArgs = new String[] { "Camera" };
```

PS:一般处理照片的App比如美图秀秀和Camera360都会把图片保存到照片下面，所以只获取照片分类中的图片应该可以满足绝大部分的需求。

1. API8 开始提供了一个在SD Card上存放公共图片的方法: `Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES)`
2. API8 开始提供了一个在SD Card上存放私有文件的地方：`Context.getExternalFilesDir(null)` App卸载的时候会删除这个目录,并且MediaScanner不会扫描这个目录，所以私有的图片最好放在这里，以免污染图库
3. `Environment.getExternalStorageDirectory()` 直接获取了外部SDCard的路径（不推荐）

关于外部存储路径更多可以查看这里的[说明](http://developer.android.com/reference/android/content/Context.html#getExternalFilesDir(java.lang.String))
