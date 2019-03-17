---
title: 自定义View(4)-绘制(Path)
top: false
cover: true
categories: Android
tags:
  - View
  - Canvas
  - Path
date: 2019-03-17 16:09:09
img:
coverImg:
summary: 
---

Canvas 中除了一系列绘制点、线、基础集合图形、图片、文字的方法, 还有一个非常有用的方法 drawPath. 利用 Path, 除了能实现类似前面所说的这些功能, 还可以构建一些更加复杂的图形.

## drawPath
```java
public void drawPath(@NonNull Path path, @NonNull Paint paint)
```

## Path
Path用于描述一段路径

### 构造方法
```java
// 创建一个空的 path
public Path()
// 复制 src 中的路径, 构建一个新的 path
public Path(Path src)
```

### Path的第一类方法: 用点和线构建路径
- 移动到某个点
```java
// 移动到某个点, 一般作为一段路径的起点
moveTo(float x, float y)
// 重新回到某个点, 一般用作下一段路径的起点 
rMoveTo(float dx, float dy)
```
- 画直线
```java
// 参数是绝对坐标
lineTo(float x, float y) 
// 参数是相对于当前点的 相对坐标               
rLineTo(float x, float y)               
```
- 画弧线
```java
// 矩形的理解: 圆弧是椭圆的一部分, 矩形用来描述椭圆的位置和大小
// startAngle: 起始角度, X轴正方向为0度
// sweepAngle: 圆弧的角度的大小
arcTo(RectF oval, float startAngle, float sweepAngle, boolean forceMoveTo) 
arcTo(float left, float top, float right, float bottom, float startAngle, float sweepAngle, boolean forceMoveTo) 
arcTo(RectF oval, float startAngle, float sweepAngle)
```
- 画二阶贝塞尔曲线
```java
// x1, y1 指控制点坐标
// x2, y2 指终点坐标
quadTo(float x1, float y1, float x2, float y2) 
rQuadTo(float dx1, float dy1, float dx2, float dy2)
```
- 画三阶贝塞尔曲线
```java
// x1, y1 指控制点1的坐标
// x2, y2 指控制点2的坐标
// x3, y3 指控终点的坐标
cubicTo(float x1, float y1, float x2, float y2, float x3, float y3) 
rCubicTo(float x1, float y1, float x2, float y2, float x3, float y3) 
```
- 封闭当前路径 
```java
// 等价于 lineTo(起点)
close()
```

### Path的第二类方法: 通过几何图形构建路径
- 添加矩形
```java
// dir 指路径的方向. 有两个取值   
	// Direction.CW    顺时针方向
	// Direction.CCW   逆时针方法
addRect(float left, float top, float right, float bottom, Direction dir) 
addRect(RectF rect, Direction dir) 
```
- 添加圆角矩形
```java
// rx、 ry 分别指圆角的 水平、垂直 半径
addRoundRect(RectF rect, float rx, float ry, Direction dir)
addRoundRect(float left, float top, float right, float bottom, float rx, float ry, Direction dir)
// radii 长度必须为 8, 对应了 4 个圆角的半径
addRoundRect(RectF rect, float[] radii, Direction dir) 
addRoundRect(float left, float top, float right, float bottom, float[] radii, Direction dir) 
```
- 添加圆
```java
// x, y 表示圆心的坐标
// radius 表示圆的半径
addCircle(float x, float y, float radius, Direction dir) 
```
- 添加椭圆
```java
// 矩形用于描述 椭圆的位置和大小
addOval(float left, float top, float right, float bottom, Direction dir) 
addOval(RectF oval, Direction dir) 
```
- 添加圆弧
```java
// 圆弧是椭圆的一部分
addArc (RectF oval, float startAngle, float sweepAngle)
```
- 添加另一个 Path
```java
addPath(Path path) 
```
				
### Path的第三类方法: 设置图形自交时的填充算法
```java
// FillType 的取值:
    // WINDING:			   非零环绕数, 默认值: 任意点取射线, 相交时顺时针加1, 逆时针减1, 非0则填充
    // EVEN_ODD:  		   奇偶填充, 任意点取射线, 计算和图形相交次数, 奇数填充, 偶数不填充  
    // INVERSE_EVEN_ODD:   取 EVEN_ODD 反色
    // INVERSE_WINDING:	   取 WINDING 反色
setFillType(FillType fillType)
```

### Path的第四类方法: 两个Path的叠加方式
```java
// op 用于描述叠加的方式, 是个枚举
	// Path.Op.DIFFERENCE:            结果取当前 path 中, 没有与参数path相交的部分
	// Path.Op.INTERSECT:             结果取当前path, 与参数path的交集
	// Path.Op.UNION:                 结果取当前path, 与参数path的并集
	// Path.Op.REVERSE_DIFFERENCE:    与DIFFERENCE刚好相反;
	// Path.Op.XOR:                   与INTERSECT刚好相反;
op(Path path, Op op);
```