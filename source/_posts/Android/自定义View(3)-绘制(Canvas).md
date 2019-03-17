---
title: 自定义View(3)-绘制(Canvas)
top: false
cover: true
categories: Android
tags:
  - View
  - Canvas
date: 2019-03-17 09:28:28
img:
coverImg:
summary: 
---

经过 [onMeasure()](http://zjloong.github.io/2019/03/16/android/zi-ding-yi-view-1-ce-liang/) 和 [onLayout()](https://zjloong.github.io/2019/03/16/android/zi-ding-yi-view-2-bu-ju/) 之后, View里显示到屏幕上, 还差最后一步 -- 绘制.

## 绘制的顺序

- View 的绘制
```java
// View 的绘制, 在 draw() 方法中完成
public void draw(Canvas canvas) {
	...
	// 绘制背景（不能重写, 只能通过 android:background  或 setBackgroundXxx方法 设置）
	drawBackground(canvas); 
	// 绘制主体 (可以重写, 系统针对该方法有优化, 可以的情况下, 尽量选择重写此方法)
	onDraw(canvas); 
	// 绘制子 View(可以重写)
	dispatchDraw(canvas); 
	// 绘制前景比如(滑动边缘渐变和滑动条)
	// API 23才开始支持, 可以重写 
	// 可通过 android:scrollbarXXX 和 setXXXScrollbarXXX 设置 滑动条
	// 可通过 android:foreground 和 setForeground 设置 前景
	onDrawForeground(canvas); 
	...
}
```
可以看出, 就像画画一样, 先画背景, 再画主题, 最后画前景. Android也将绘制流程按照类似的逻辑拆分到不同的方法中去完成. 我们在自定义控件时, 根据View和ViewGroup的不同, 需要实现的步骤也不同.

- 自定义View的时候, 我们只需要实现一个方法就行:
```java
protected void onDraw(Canvas canvas) {
	// 为什么是空实现, 而不是抽象方法?
	// 因为 onDraw 方法用于绘制自己的主体内容, 除了定义View的时候必须实现, 定义ViewGroup 可以不实现该方法
}
```

- 自定义ViewGroup时, 
	- 根据需要选择是否要实现 onDraw() 方法;
	- 一般情况下, 也无需我们实现 dispatchDraw() 方法, 因为 ViewGroup已经实现了;
	- 除非有需要, 大部分情况, 无需实现 onDrawForeground() 方法.
	- 出于效率的考虑，系统的某些 ViewGroup 默认会绕过 draw() 方法，换而直接执行 dispatchDraw()，以此来简化绘制流程. 如果自定义 ViewGroup, 且需要重写 dispatchDraw() 以外的绘制方法时, 可能需要调用 view.setWillNotDraw(false) 来切换到完整的绘制流程

## Canvas 绘制
经过分析, 可以知道, 当我们自定义View的时候, 绘制相关的逻辑, 就在 onDraw()方法中完成. 该方法带有一个参数 **Canvas**, 类似于现实中画画需要画布与画笔一样, canvas就是Android中的画布. 在Canvas中, 提供了丰富的API, 协助我们完成各种绘制

### 绘制颜色
```java
drawColor(@ColorInt int color)                                          
drawRGB(int r, int g, int b)
drawARGB(int a, int r, int g, int b)
```

### 绘制点
```java
// Paint 就是画笔, 后面单独再说
drawPoint(float x, float y, Paint paint)                                
drawPoints(float[] pts, Paint paint)   
// pts指点的坐标, 每两个数表示一个点;  
// offset 指跳过前面几个数; 
// count指绘制几个数                            
drawPoints(float[] pts, int offset, int count, Paint paint)     
```

### 绘制直线
```java
drawLine(float startX, float startY, float stopX, float stopY, Paint paint) 
drawLines(float[] pts, int offset, int count, Paint paint) 
drawLines(float[] pts, Paint paint)
```

### 绘制基础几何图形
- 圆形
```java
drawCircle(float centerX, float centerY, float radius, Paint paint)     
```
- 矩形
```java
drawRect(float left, float top, float right, float bottom, Paint paint) 
drawRect(RectF rect, Paint paint)
drawRect(Rect rect, Paint paint)
```
- 圆角矩形
```java
drawRoundRect(float left, float top, float right, float bottom, float rx, float ry, Paint paint)
drawRoundRect(RectF rect, float rx, float ry, Paint paint)
```
- 椭圆
```java
drawOval(float left, float top, float right, float bottom, Paint paint) 
drawOval(RectF rect, Paint paint)
```
- 弧形或扇形 
```java
// 前4个参数描述所在的椭圆; startAngle指起始角度; sweepAngle指划过的角度; useCenter表示是否连接到圆心
drawArc(float left, float top, float right, float bottom, float startAngle, float sweepAngle, boolean useCenter, Paint paint) 
```

### 绘制图片
```java
drawBitmap(Bitmap bitmap, float left, float top, Paint paint) 
// 截取bitmap的一个区域绘制到view的指定区域内
drawBitmap(Bitmap bitmap, Rect src, RectF dst, Paint paint)         
drawBitmap(Bitmap bitmap, Rect src, Rect dst, Paint paint)
drawBitmap(Bitmap bitmap, Matrix matrix, Paint paint)
```

### 绘制路径 (内容较多, 以后单独再说)
```java
drawPath(Path path, Paint paint)
```

### 绘制文字 (内容较多, 以后单独再说)
```java
drawText(String text, float x, float y, Paint paint)  
```

## Picture (记录绘制操作)
对于绘制过程中的一些重复性的操作, 可以通过 Picture 将其 '录制' 下来, 需要的时候拿来就能用, 对于重复的操作可以更加省时省力.
- canvas 相关API
```java
public void drawPicture (Picture picture)
public void drawPicture (Picture picture, Rect dst)
public void drawPicture (Picture picture, RectF dst)
```
- Picture 的API
```java
public int getWidth ()					                // 获取宽度
public int getHeight ()					                // 获取高度
public Canvas beginRecording (int width, int height)	// 开始录制 (返回一个Canvas，此Canvas上的操作都会记录下来)
public void endRecording ()				                // 结束录制
public void draw (Canvas canvas)			            // 将Picture中内容绘制到Canvas中
```
- 示例
```java
Picture mPicture = new Picture();
Canvas canvas = mPicture.beginRecording(500, 500);        // 开始录制 
Paint paint = new Paint();                                // 创建一个画笔
paint.setColor(Color.BLUE);
paint.setStyle(Paint.Style.FILL);
canvas.translate(250,250);                                // 在Canvas中具体操作
canvas.drawCircle(0,0,100,paint);
mPicture.endRecording();                                  // 结束录制
```
- Picture 还可以包装为 PictureDrawable 使用
```java
PictureDrawable drawable = new PictureDrawable(mPicture);
drawable.setBounds(0,0,250,mPicture.getHeight());         // 设置绘制区域 -- 注意此处所绘制的实际内容不会缩放
```

## Canvas 裁剪 & 几何变换
除了提供绘制功能外, Canvas还提供了一些其它的方法, 可以让我们再绘制过程中, 实现一些相对复杂的功能

### 裁剪 
- 相关API
```java
// 裁剪矩形范围
boolean clipRect(RectF rect)
boolean clipRect(Rect rect) 
boolean clipRect(float left, float top, float right, float bottom)
boolean clipRect(int left, int top, int right, int bottom)
// 按路径裁剪
boolean clipPath(Path path)
boolean clipOutRect(float left, float top, float right, float bottom)
...
boolean clipOutPath(@NonNull Path path)
```
- 示例: (记得要在裁剪前后加上 Canvas.save() 和 Canvas.restore() 来及时恢复绘制范围)
```java
canvas.save();
canvas.clipRect(left, top, right, bottom);                      // 裁剪
canvas.drawBitmap(bitmap, x, y, paint);
canvas.restore();
```

### 几何变换
- 使用 Canvas 来做常见的二维变换 (canvas的几何变化可以叠加使用, 但是作用顺序是反的, 后写的会先生效)
```java
// 平移
translate(float dx, float dy) 
// 旋转
rotate(float degrees, float px, float py)
// 缩放
scale(float sx, float sy, float px, float py) 
// 扭曲(斜切)
skew(float sx, float sy)
```
- 使用 Matrix 来做常见变换
```java
Matrix matrix = new Matrix();                                // 第一步: 初始化 Matrix 对象
matrix.reset();
matrix.postTranslate() / postRotate() / scale() / skew()     // 第二步: matrix 应用几何变换
... 
canvas.save();
canvas.concat(matrix);
canvas.drawBitmap(bitmap, x, y, paint);                      // 第三步: 将几何变换应用到 canvas
canvas.restore();
// canvas.concat(matrix) 用 Canvas 当前的变换矩阵和 Matrix 相乘，即基于 Canvas 当前的变换，叠加上 Matrix 中的变换
// canvas.setMatrix(matrix) 替换canvas当前的变换矩阵, 该方法可能有问题, 尽量不用
```
- 使用 Matrix 来做自定义变换 (多点映射)
```java
// src 和 dst 是源点集合目标点集, 长度必须是偶数
// srcIndex 和 dstIndex 是 从 src的第几个点开始采集, 从 dst的第几个点开始映射
// pointCount 是采集的点的个数
setPolyToPoly(float[] src, int srcIndex, float[] dst, int dstIndex, int pointCount)
```

### 使用 Camera 来实现三维变换
Camera 就是一个假想中的相机, 其默认位置View的左上角, 屏幕的上方. 可以将Camera理解为Unity中的光源, 我们在屏幕上最终看到的View, 就是Camera从它所在的视角'拍摄'出来的图片.
- Camera的坐标系
	|   原点  | x轴方向 | y轴方向 | z轴方向 |
	| ------ | ------ | ------ |
	| 在view的左上角 (0, 0, 0) | 向右为正 | 向上为正(和view的坐标系相反) | 向里为正 |
- Camera所在的坐标: 
	- 默认坐标: (0, 0, -8) 即在view的左上角向屏幕方向 8英寸处(一英尺 = 72px) 
	- 设置Camera的坐标:
	```java
	// 实际使用中一般 x, y都传 0, 修改z的值(远小近大, 注意单位是 英寸), 因为平面的移动一般考canvas实现
	camera.setLocation(x, y, z) 
	```
- Camera旋转
```java
camera.rotateX(deg)      // deg为正时: 图片下部靠近屏幕
camera.rotateY(deg)      // deg为正时: 图片左部靠近屏幕
camera.rotateZ(deg)      // deg为正时: 图片逆时针旋转
camera.rotate(x, y, z)
```
- Camera平移
```java
// 基本用不到, 一般靠 canvas 实现平面的移动
camera.translate(float x, float y, float z)   
```
- 示例
```java
camera.save();                        // Camera 和 Canvas 一样也需要保存和恢复状态才能正常绘制
camera.rotateX(30); 			      // 旋转 Camera 的三维空间
canvas.translate(centerX, centerY);   // 旋转之后把投影移动回来
camera.applyToCanvas(canvas);         // 把旋转投影到 Canvas
canvas.translate(-centerX, -centerY); // 旋转之前把绘制内容移动到轴心（原点）
camera.restore(); 					  // 恢复 Camera 的状态
canvas.drawBitmap(bitmap, point1.x, point1.y, paint);
canvas.restore();
```

## Canvas 图层
在绘图软件中, 图层是个很常见的概念, 常常会将不同的元素, 分别画在不同的图层中. Canvas 也有图层的概念, 它提供了一系列的方法, 可以将当前画布的状态保存到栈中, 在需要的时候, 又可以随时回复. 有了图层的概念, 可以让我们在绘制的时候有了 '反悔' 的机会. 同时利用图层也可以实现一些更复杂的功能.
- 保存画布
```java
// 将画布的所有信息保存到栈中, 返回值表示栈中的 index
public int save () 
// 和 save() 类似. 不过会新建一个不在屏幕内的bitmap(离屏缓冲), 之后的绘制都在该bitmap上进行. 只有在调用 canvas.restoreXX() 后, 才会将 bitmap 绘制到屏幕上  
// 参数可指定新图层的矩形范围
public int saveLayer (RectF bounds, Paint paint)
public int saveLayer (float left, float top, float right, float bottom, Paint paint)
// 个 saveLayer() 类似, 且新图层有透明度, 透明度为 alpha
// alpha 取值范围: 0-255
public int saveLayerAlpha (RectF bounds, int alpha)
public int saveLayerAlpha (float left, float top, float right, float bottom, int alpha)
```
- 恢复
```java
// 状态回滚，就是从栈顶取出一个状态然后根据内容进行恢复
public void restore()
// 弹出指定位置及其以上所有的状态，并按照指定位置的状态进行恢复
// saveCount 一般传入 saveXX() 方法的返回值
public void restoreToCount(int saveCount)
// 获取栈中内容的数量(即保存次数). 该函数的最小返回值为1, 即使弹出了所有的状态，返回值依旧为1，代表默认状态
public int getSaveCount()
```
