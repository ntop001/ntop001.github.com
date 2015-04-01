1. Overdraw(重叠渲染？)

发生这种情况主要是因为重叠的图层都有背景或者其他颜色，被绘制在同一个像素上了，但是因为底下的图层虽然被绘制但是确实被遮挡看不到，造成了
资源浪费，在Android手机的Debug模式中，可以开启Overdraw测试。

* No color means there is no overdraw. The pixel was painted only once. In this example, you can see that the background is intact.
* Blue indicates an overdraw of 1x. The pixel was painted twice. Large blue areas are acceptable (if the entire window is blue, you can get rid of one layer.) 
* Green indicates an overdraw of 2x. The pixel was painted three times. Medium-sized green areas are acceptable but you should try to optimize them away.
* Light red indicates an overdraw of 3x. The pixel was painted four times. Small light red areas are acceptable.
* Dark red indicates an overdraw of 4x or more. The pixel was painted 5 times or more. This is wrong. Fix it.


[Android Performance Case Study](http://www.curious-creature.com/docs/android-performance-case-study-1.html)
[Video](https://www.youtube.com/watch?v=T52v50r-JfE&index=2&list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE)

2. VSYNC

刷新率指的是屏幕的刷新频率，这个是取决于硬件性能的一个固定值。帧率指GPU每秒渲染帧的数量。如果这两个值差不多的情况下在好不过了，但是基本会出现两个极端，刷新率大于帧率或者帧率大于刷新率，如果是前者屏幕会把还有渲染好的一帧显示到屏幕上，导致出现屏幕撕裂的现象，一般用双缓冲技术解决这个问题。在系统中实现两个缓冲区FrameBuffer和BackBuffer，屏幕刷新从FrameBuffer中读取画面，GPU绘制放在BackBuufer中，当绘制完毕后再一起写入到FrameBuffer，这两个Buffer的同步叫VSYNC。


3. Profile GPU Rendering (if you can measure it, you can optimize it)

Debug模式中可以打开这个模式，分为三部分，最上是通知栏的性能监测，最下是虚拟按键的性能监测，中间是App的性能监测。
三种颜色分别表示：

1. 蓝色 更新DisplayList（OpenGL中的术语）所用时间，一个按钮的绘制会先转化成OpenGL绘制指令（DisplayList），蓝色代表转换时间，GPU会缓存DisplayList，所以并不是每次绘制都会重新转化。
2. 红色 执行DisplayList所用时间，复杂的图形会导致执行时间很长
3. 黄色 GPU绘制这一帧的执行时间

绿线是16ms（60fps,1000/60=16.66）应该保持上面的色线的所代表的时间小于16ms，这样绘制出来的图像感官上会比较流畅。









