1. Overdraw(重叠渲染？)

发生这种情况主要是因为重叠的图层都有背景或者其他颜色，被绘制在同一个像素上了，但是因为底下的图层虽然被绘制但是确实被遮挡看不到，造成了
资源浪费，在Android手机的Debug模式中，可以开启Overdraw测试。

* No color means there is no overdraw. The pixel was painted only once. In this example, you can see that the background is intact.
* Blue indicates an overdraw of 1x. The pixel was painted twice. Large blue areas are acceptable (if the entire window is blue, you can get rid of one layer.) Green indicates an overdraw of 2x. The pixel was painted three times. Medium-sized green areas are acceptable but you should try to optimize them away.
* Light red indicates an overdraw of 3x. The pixel was painted four times. Small light red areas are acceptable.
* Dark red indicates an overdraw of 4x or more. The pixel was painted 5 times or more. This is wrong. Fix it.
