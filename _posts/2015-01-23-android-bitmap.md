---
layout: post
title: "Android图片和内存"
description: ""
category: "知识整理"
tags: [Android,图片,内存,Bitmap,Memory]
---

每个App不可避免的都遇到过这种问题`java.lang.OutofMemoryError: bitmap size exceeds VM budget.`，基本上所有App的崩溃记录中都会有这条，所以App的内存控制变成了App后期主要面对的问题，而且对于图片处理的App这种问题从App的一开始就需要细心的解决。

发生内存溢出主要有这些原因

1. 移动设备的内存限制。Android给每个App分配16M的内存（这个内存上限是比较早期的限制，基本上每个出厂设备的真实分配的内存上限都会远大于这个值，所以经常在模拟器上溢出了，但是真机不会，而且不同机型表现也不一样）。
2. 图片会占用大量内存。使用GalaxyNexus拍一张照片大概的像素是2592x1936（大概5M），如果图片配置是ARGB_8888（Android2.3之后默认）那么会占用大概19M内存（2592 *1936 *4 bytes），系统分配的内存会立刻耗尽。
3. ListView，GridView 和 ViewPager这种类型的控件会频繁的加载不同的图片，内存消耗巨大。


## 图片加载

图片往往很大，但是设备的分辨率和UI控件可能很小，这些时候很明显不能把整个图片加载到内存中，而是应该加载正好适合的图片。如果有一个方法，只需要提供图片的路径和需要的尺寸就可以正确的加载需要的图片大小的方法，就再好不过了，但是现有的方法仅支持采样读取，也就是2的次方来缩放图片。如果自己提供这样的方法，至少需要两种信息1.图片的原始尺寸2.需要载入内存的尺寸，这样可以在图片载入内存的时候计算采样值对图片进行裁剪。

`BitmapFactory`类提供了一系列的工厂方法用来把图片导入内存，在导入图片的时候可以利用`BitmapFactory.Options`提前读取图片的尺寸，而不需要加载整张图片。

```
BitmapFactory.Options options = new BitmapFactory.Options();
//设置为true就可以只读取图片的尺寸了
options.inJustDecodeBounds = true;
BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
int imageHeight = options.outHeight;
int imageWidth = options.outWidth;
String imageType = options.outMimeType;
```

下面的方法，可以用来计算采样率，因为每次缩放都是2的次方，所以需要一个合适的采样率，来得到一个最接近所需要的长宽值的值。

```
public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // Raw height and width of image
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;

    if (height > reqHeight || width > reqWidth) {

        final int halfHeight = height / 2;
        final int halfWidth = width / 2;

        // Calculate the largest inSampleSize value that is a power of 2 and keeps both
        // height and width larger than the requested height and width.
        while ((halfHeight / inSampleSize) > reqHeight
                && (halfWidth / inSampleSize) > reqWidth) {
            inSampleSize *= 2;
        }
    }

    return inSampleSize;
}
```

然后就可以设置合适的参数读取图片了

```
//.使用
mImageView.setImageBitmap(
    decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));
//.优化图片读取的函数    
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
        int reqWidth, int reqHeight) {

    // First decode with inJustDecodeBounds=true to check dimensions
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);

    // Calculate inSampleSize
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

    // Decode bitmap with inSampleSize set
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}
```

## 异步加载图片

对于上面的图片加载过程是比较费时的，如果放在UI线程中，那么可能会在性能比较低的机型上造成卡顿，所以最好把它放在异步线程中。使用AsycTask完成这一任务非常简单。

```
//使用
public void loadBitmap(int resId, ImageView imageView) {
    BitmapWorkerTask task = new BitmapWorkerTask(imageView);
    task.execute(resId);
}
//异步加载图片
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    private final WeakReference<ImageView> imageViewReference;
    private int data = 0;

    public BitmapWorkerTask(ImageView imageView) {
        // Use a WeakReference to ensure the ImageView can be garbage collected
        imageViewReference = new WeakReference<ImageView>(imageView);
    }

    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        data = params[0];
        return decodeSampledBitmapFromResource(getResources(), data, 100, 100));
    }

    // Once complete, see if ImageView is still around and set bitmap.
    @Override
    protected void onPostExecute(Bitmap bitmap) {
        if (imageViewReference != null && bitmap != null) {
            final ImageView imageView = imageViewReference.get();
            if (imageView != null) {
                imageView.setImageBitmap(bitmap);
            }
        }
    }
}
```

这里面有个特殊的处理，为了避免AsyncTask持有ImageView的引用，导致ImageView无法回收（ImageView一般还会持有Activity的引用，导致整个Activity无法回收），在此使用了弱引用，弱引用不会阻碍ImageView的回收，所以在图片处理完之后，还需要判断弱引用中的ImageView是否已经被回收。

这种处理可以满足一般的ImageView图片填充，但是对于ListView或者GridView这种会产生ImageView复用的控件，可能会造成问题，如果一个ImageView已经被复用了，但是之前的AsyncTask却尝试给它设置图片，这显然是错误的。这种问题的处理Android文档上给出一种解决方法，让ImageView持有AsyncTask的引用，在每次给ImageView设置图片之前，先检查ImageView有没有已经被附着了一个AsyncTask，如果已经有了而且是错的，那么就取消这个任务，重新附着一个AsyncTask。

1. 创建一个持有AsyncTask图片加载任务的Drawable对象
2. 在给ImageView设置异步图片加载之前先检测，再设置

```
//一个持有AsyncTask对象的Drawable
static class AsyncDrawable extends BitmapDrawable {
    private final WeakReference<BitmapWorkerTask> bitmapWorkerTaskReference;

    public AsyncDrawable(Resources res, Bitmap bitmap,
            BitmapWorkerTask bitmapWorkerTask) {
        super(res, bitmap);
        bitmapWorkerTaskReference =
            new WeakReference<BitmapWorkerTask>(bitmapWorkerTask);
    }

    public BitmapWorkerTask getBitmapWorkerTask() {
        return bitmapWorkerTaskReference.get();
    }
}
```

这里同样使用若引用，避免ImageView持有了`BitmapWorkerTask`对象导致它不能回收。

```
//先检测，后设置
public void loadBitmap(int resId, ImageView imageView) {
    if (cancelPotentialWork(resId, imageView)) {
        final BitmapWorkerTask task = new BitmapWorkerTask(imageView);
        final AsyncDrawable asyncDrawable =
                new AsyncDrawable(getResources(), mPlaceHolderBitmap, task);
        imageView.setImageDrawable(asyncDrawable);
        task.execute(resId);
    }
}
// 具体的检测方法，先从ImageView中拿到被附着的AsyncTask对象，判断这个AsyncTask执行的任务是否是当前ImageView
// 需要的任务。如果不是就返回true是返回false
public static boolean cancelPotentialWork(int data, ImageView imageView) {
    final BitmapWorkerTask bitmapWorkerTask = getBitmapWorkerTask(imageView);

    if (bitmapWorkerTask != null) {
        final int bitmapData = bitmapWorkerTask.data;
        // If bitmapData is not yet set or it differs from the new data
        if (bitmapData == 0 || bitmapData != data) {
            // Cancel previous task
            bitmapWorkerTask.cancel(true);
        } else {
            // The same work is already in progress
            return false;
        }
    }
    // No task associated with the ImageView, or an existing task was cancelled
    return true;
}
//这是一个辅助方法用来从ImageView中拿到附着的AsyncTask
private static BitmapWorkerTask getBitmapWorkerTask(ImageView imageView) {
   if (imageView != null) {
       final Drawable drawable = imageView.getDrawable();
       if (drawable instanceof AsyncDrawable) {
           final AsyncDrawable asyncDrawable = (AsyncDrawable) drawable;
           return asyncDrawable.getBitmapWorkerTask();
       }
    }
    return null;
}
```

最后还需要更新一下BitmapWorkerTask类，因为我们会取消AsyncTask的任务，但是又非常有可能这个任务刚巧是当前需要的任务，这是List滚动太快，刚刚被取消然后又复用了。

```
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...

    @Override
    protected void onPostExecute(Bitmap bitmap) {
        if (isCancelled()) {
            bitmap = null;
        }

        if (imageViewReference != null && bitmap != null) {
            final ImageView imageView = imageViewReference.get();
            final BitmapWorkerTask bitmapWorkerTask =
                    getBitmapWorkerTask(imageView);
            if (this == bitmapWorkerTask && imageView != null) {
                imageView.setImageBitmap(bitmap);
            }
        }
    }
}
```

## 图片缓存

上述的实现已经非常完美了，截取适合大小的图片，不会占用更多的内存；图片异步加载，不会导致主线程的阻塞。但是如果一个列表来回滚动的话，无数个AsyncTask会被创建出来一次又一次的处理图片，对于CPU的消耗和图片显示速度会有很大的影响。这个时候需要缓存把每次已经处理好的图片缓存在内存或者SD卡上，这样下次使用的时候可以直接从缓存中拿到图片而不是每次都去重新读取。

API12提供了`LruCache`(http://developer.android.com/reference/android/util/LruCache.html)类（LRU = Least Recently Used），这种缓存算法每次当一个值被访问的时候都会把这个值移到队列的开始，如果新的值被插入到队列的开始了，就把队列尾部的值移除。为了兼容我们使用SupportV4包中提供的[兼容版本](http://developer.android.com/reference/android/support/v4/util/LruCache.html)。

> The LruCache class (also available in the Support Library for use back to API Level 4) is particularly well suited to the task of caching bitmaps, keeping recently referenced objects in a strong referenced LinkedHashMap and evicting the least recently used member before the cache exceeds its designated size.

默认情况下LruCache的大小是Cache中的值的数量，但是这里需要缓存的是图片，所以需要告诉LruCache新的大小计算方式

```
   int cacheSize = 4 * 1024 * 1024; // 4MiB
   LruCache bitmapCache = new LruCache(cacheSize) {
       protected int sizeOf(String key, Bitmap value) {
           return value.getByteCount();
       
   }}
```

下面是一段，使用LruCache的示例，使用1/8的内存作为图片的缓存。

```
private LruCache<String, Bitmap> mMemoryCache;

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    // Get max available VM memory, exceeding this amount will throw an
    // OutOfMemory exception. Stored in kilobytes as LruCache takes an
    // int in its constructor.
    final int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);

    // Use 1/8th of the available memory for this memory cache.
    final int cacheSize = maxMemory / 8;

    mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
        @Override
        protected int sizeOf(String key, Bitmap bitmap) {
            // The cache size will be measured in kilobytes rather than
            // number of items.
            return bitmap.getByteCount() / 1024;
        }
    };
    ...
}

public void addBitmapToMemoryCache(String key, Bitmap bitmap) {
    if (getBitmapFromMemCache(key) == null) {
        mMemoryCache.put(key, bitmap);
    }
}

public Bitmap getBitmapFromMemCache(String key) {
    return mMemoryCache.get(key);
}
```

使用起来很简单，在每次处理图片之前先到缓存中看看有没有？有的话就从缓存中直接读取，如果没有的话，就开启一个AsyncTask去加载，在加载完之后，要记得把图片放入缓存。

```
public void loadBitmap(int resId, ImageView imageView) {
    final String imageKey = String.valueOf(resId);

    final Bitmap bitmap = getBitmapFromMemCache(imageKey);
    if (bitmap != null) {
        mImageView.setImageBitmap(bitmap);
    } else {
        mImageView.setImageResource(R.drawable.image_placeholder);
        BitmapWorkerTask task = new BitmapWorkerTask(mImageView);
        task.execute(resId);
    }
}

class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...
    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        final Bitmap bitmap = decodeSampledBitmapFromResource(
                getResources(), params[0], 100, 100));
        addBitmapToMemoryCache(String.valueOf(params[0]), bitmap);
        return bitmap;
    }
    ...
}
```

但是仅仅缓存在内存中还是有缺点的，如果图片是服务器上下载的那么App关闭之后再打开还需要重新下载，如果List加载的图片太多，内存中的缓存很容易就满了。所以还需要把图片在SD卡上也缓存一份。文档上使用的一个实现[DiskLruCache](https://android.googlesource.com/platform/libcore/+/jb-mr2-release/luni/src/main/java/libcore/io/DiskLruCache.java)。

1. 初始化磁盘缓存（磁盘操作使用了异步线程）
2. 在AsyncTask中加载图片的时候先查询磁盘，是否有缓存？有就直接读取，没有则加载图片并同时放入内存和磁盘缓存

```
private DiskLruCache mDiskLruCache;
private final Object mDiskCacheLock = new Object();
private boolean mDiskCacheStarting = true;
private static final int DISK_CACHE_SIZE = 1024 * 1024 * 10; // 10MB
private static final String DISK_CACHE_SUBDIR = "thumbnails";

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    // Initialize memory cache
    ...
    // Initialize disk cache on background thread
    File cacheDir = getDiskCacheDir(this, DISK_CACHE_SUBDIR);
    new InitDiskCacheTask().execute(cacheDir);
    ...
}

// 初始化DiskCache
class InitDiskCacheTask extends AsyncTask<File, Void, Void> {
    @Override
    protected Void doInBackground(File... params) {
        synchronized (mDiskCacheLock) {
            File cacheDir = params[0];
            mDiskLruCache = DiskLruCache.open(cacheDir, DISK_CACHE_SIZE);
            mDiskCacheStarting = false; // Finished initialization
            mDiskCacheLock.notifyAll(); // Wake any waiting threads
        }
        return null;
    }
}
//加载图片的时候先查询磁盘缓存
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...
    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        final String imageKey = String.valueOf(params[0]);

        // Check disk cache in background thread
        Bitmap bitmap = getBitmapFromDiskCache(imageKey);

        if (bitmap == null) { // Not found in disk cache
            // Process as normal
            final Bitmap bitmap = decodeSampledBitmapFromResource(
                    getResources(), params[0], 100, 100));
        }

        // Add final bitmap to caches
        addBitmapToCache(imageKey, bitmap);

        return bitmap;
    }
    ...
}
//新的图片同时加入内存和磁盘缓存
public void addBitmapToCache(String key, Bitmap bitmap) {
    // Add to memory cache as before
    if (getBitmapFromMemCache(key) == null) {
        mMemoryCache.put(key, bitmap);
    }

    // Also add to disk cache
    synchronized (mDiskCacheLock) {
        if (mDiskLruCache != null && mDiskLruCache.get(key) == null) {
            mDiskLruCache.put(key, bitmap);
        }
    }
}
//辅助方法，从磁盘缓存中读取图片
public Bitmap getBitmapFromDiskCache(String key) {
    synchronized (mDiskCacheLock) {
        // Wait while disk cache is started from background thread
        while (mDiskCacheStarting) {
            try {
                mDiskCacheLock.wait();
            } catch (InterruptedException e) {}
        }
        if (mDiskLruCache != null) {
            return mDiskLruCache.get(key);
        }
    }
    return null;
}

// Creates a unique subdirectory of the designated app cache directory. Tries to use external
// but if not mounted, falls back on internal storage.
public static File getDiskCacheDir(Context context, String uniqueName) {
    // Check if media is mounted or storage is built-in, if so, try and use external cache dir
    // otherwise use internal cache dir
    final String cachePath =
            Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState()) ||
                    !isExternalStorageRemovable() ? getExternalCacheDir(context).getPath() :
                            context.getCacheDir().getPath();

    return new File(cachePath + File.separator + uniqueName);
}
```

需要注意的是，所以的磁盘操作都是放在异步线程中的，在查询内存缓存的时候，直接在UI线程中完成的，但是在查询磁盘缓存的时候，这个过程放在了AsyncTask里面。


PS: 其实对于上面的各种策略，网上已经有现成的而且实现的非常好的库可供使用，自己看看可能对理解这些缓存和图片处理方式更有帮助吧。

1. [Volley](http://developer.android.com/training/volley/index.html) 封装了网络请求队列和图片加载缓存功能
2. [Universal Image Loader](https://github.com/nostra13/Android-Universal-Image-Loader) 一个纯粹的图片加载和缓存框架（有一些额外的功能，比如图片圆角之类）
3. [Picasso](http://square.github.io/picasso/) 另一个非常强大的图片下载和缓存库

内容来源：

1. [Displaying Bitmaps Efficiently](http://developer.android.com/training/displaying-bitmaps/index.html)
2. [LruCache](http://developer.android.com/reference/android/util/LruCache.html)
2. [DiskLruCache](https://android.googlesource.com/platform/libcore/+/jb-mr2-release/luni/src/main/java/libcore/io/DiskLruCache.java)
