## 自定义View

### 绘制的基本要素

* onDraw(Canvas) 绘制需要重写的方法，也可以重写其他的方法
* Canvas 实际绘制时调用的是Canvas的Api
* Paint 用来配置风格
* 坐标系

	![coordinator](/Users/lixiangyue/Personal/blog/Blog/Android/images/coordinator.jpg)

y轴朝下，x轴朝右，计算角度时，以x轴为起始轴，顺时针为正方向

* 尺寸单位   px像素，因为绘制时是在屏幕上画了。dp是用来适配的，具体可以看dp到px的转换。
* Api：

	| API | 备注 |
	| --- | --- | 
	|`Canvas.drawXXX()` | 画图形类的API，可以是line，circle，path等等，一般path可以在成员变量初始化，之后在onSizeChanged()方法中进行重新赋值|
	|`PathMeasure` | 可以用来测量path的各种数据，比如长度，角度等等（pathMeasure.length获取长度，pathMeasure.getPosTan()获取角度）|
	|`paint.pathEffect`|用来设置paint的效果，可以用来画虚线等|
	



