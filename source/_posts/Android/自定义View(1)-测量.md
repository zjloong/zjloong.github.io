---
title: 自定义View(1)-测量
top: true
cover: true
categories: Android
tags:
  - View
date: 2019-03-16 09:49:42
img:
coverImg:
summary:
---

## 为什么要了解自定义控件
自定义控件是一项Android开发中必须要掌握的技能. 虽然Github上有各种现成的轮子, 可以满足日常开发中的大部分需求. 但实际开发中, 各种情况都可能发生, 只有掌握了相关原理, 才能更好的应对各种场景. 

一个完整的自定义控件, 主要包含下面三个步骤:
- onMeasure: 测量子View和自己的宽高
- onLayout: 将子View布局到指定的位置
- onDraw: 绘制内容

onMeasure作为需要我们处理的第一个步骤, 重要性不言而喻. 首先我们要知道一点, Android中的View都是矩形. 包括图片(ImageView)、文字(TextView)、...各种列表、容器,
不管它外在的表现形式是什么, 它们的本质, 都是屏幕上一个一个矩形. 而测量的作用, 就是确定矩形的宽高.

## onMeasure
onMeasure 是 View 里面的一个方法:
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	super.onMeasure(widthMeasureSpec, heightMeasureSpec);
}
```
该方法有两个 int 类型的参数***widthMeasureSpec, heightMeasureSpec***. 实际上它们确实和宽高有关, 但并不仅仅指宽高, 要了解这两个参数, 就必须先了解 **MeasureSpec**.

## MeasureSpec
MeasureSpec是View的静态内部类, 其中主要有三个属性和三个方法我们需要了解
```java
// 测量模式-无限制. 子View想要多大就多大, 一般用于ScrollView等滑动控件, 大部门情况下, 我们可以不用考虑
public static final int UNSPECIFIED = 0 << MODE_SHIFT;
// 测量模式-精确值. 在布局时设置宽(高)为具体的值, 或match-parent时, 测量模式就是 EXACTLY
public static final int EXACTLY     = 1 << MODE_SHIFT;
// 测量模式-限制最大值. 子View宽(高)根据自己的内容决定, 但不超过父容器允许的限制, 布局属性为wrap-content, 测量模式一般就是 AT_MOST
public static final int AT_MOST     = 2 << MODE_SHIFT;
...
// 将size和mode组成一个int值, 
public static int makeMeasureSpec(int size, int mode){
	// sUseBrokenMakeMeasureSpec 只在API17以下作用, 可以忽略
	if (sUseBrokenMakeMeasureSpec) {
		return size + mode;
    } else {
	    // int值中的高2位表示mode(测量模式), 低30位表示大小(size)
		return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
// 从上面的结果逆向取出 mode
public static int getMode(int measureSpec) {
	return (measureSpec & MODE_MASK);
}
// 取出size
public static int getSize(int measureSpec) {
	return (measureSpec & ~MODE_MASK);
}
```

## 简析FramLayout测量步骤
以FramLayout为例, 简单的看下它的测量逻辑, 看看MeasureSpec的具体用法(只保留关键代码):
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	int count = getChildCount();
	...
	for (int i = 0; i < count; i++) {
		...
		// 测量子View
		measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
		...
	}
	// 测量自己
	setMeasuredDimension(...);
	...
}
... 
// ViewGroup的方法
protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
	...
	final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
	// 计算子View的 MeasureSpec
	final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec, ...);
	...
	// View.measure
	child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
...
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
	...
	onMeasure(widthMeasureSpec, heightMeasureSpec);
	...
}
```
可以看出, 整个测量过程大概是这样的: 
1. 父容器触发onMeasure方法;
2. 父容器结合自己的 MeasureSpec 和子View的LayoutParams计算出子View的MeasureSpec;
3. 父容器触发子View的measure方法, 子Viewd的measure方法中又触发器onMeasrue...;
4. 父容器计算自己的宽高, 然后通过 setMeasuredDimension 进行保存 ;

## getChildMeasureSpec 分析
而计算 MeasureSpec 的关键步骤, 就在 ViewGroup.getChildMeasureSpec 这个方法中:
```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
	// 父容器的测量模式
	int specMode = MeasureSpec.getMode(spec);
	// 父容器的大小
	int specSize = MeasureSpec.getSize(spec);
	// 父容器减去padding后的大小
	int size = Math.max(0, specSize - padding);
	// 该变量用于接收子View的大小
	int resultSize = 0;
	// 该变量用于接收子View的测量模式
	int resultMode = 0;
	switch (specMode) {
		case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
	            // 如果子View的 layoutParams.width/height >= 0
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
	            // layoutParams.width/height == MATCH_PARENT
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
	            // layoutParams.width/height == WRAP_CONTENT
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

总结一下子控件中 widthMeasureSpec 和 heightMeasureSpec 的计算规则:
* 父View 的测量模式为 MeasureSpec.EXACTLY (固定值):
	* 子View 的 layoutParams.width/height >= 0, 即指定了具体的值:
	 ```java
		MeasureSpec.makeMeasureSpec(layoutParams.width/height, MeasureSpec.EXACTLY)
	 ```
	* 子View 的 layoutParams.width/height 为 MATCH_PARENT:
	 ```java
		MeasureSpec.makeMeasureSpec(父View 的可用尺寸, MeasureSpec.EXACTLY)
	 ```
	* 子View 的 layoutParams.width/height 为 WRAP_CONTENT:
	 ```java
		MeasureSpec.makeMeasureSpec(父View 的可用尺寸, MeasureSpec.AT_MOST)
	```
* 父View 的测量模式为 MeasureSpec.AT_MOST (限制上限):
	* 子View 的 layoutParams.width/height >= 0, 即指定了具体的值:
	 ```java
		MeasureSpec.makeMeasureSpec(layoutParams.width/height, MeasureSpec.EXACTLY)
	 ```
	* 子View 的 layoutParams.width/height 为 MATCH_PARENT:
	 ```java
		MeasureSpec.makeMeasureSpec(父View 的可用尺寸, MeasureSpec.AT_MOST)
	 ```
	* 子View 的 layoutParams.width/height 为 WRAP_CONTENT:
	 ```java
		MeasureSpec.makeMeasureSpec(父View 的可用尺寸, MeasureSpec.AT_MOST)
	```
* 父View 的测量模式为 MeasureSpec.UNSPECIFIED (无限制):
	* 子View 的 layoutParams.width/height >= 0, 即指定了具体的值:
	 ```java
		MeasureSpec.makeMeasureSpec(layoutParams.width/height, MeasureSpec.EXACTLY)
	 ```
	* 子View 的 layoutParams.width/height 为 MATCH_PARENT:
	 ```java
		MeasureSpec.makeMeasureSpec( if (API<23){ 0 } else { 父View 的可用尺寸 }, MeasureSpec.UNSPECIFIED)
	 ```
	* 子View 的 layoutParams.width/height 为 WRAP_CONTENT:
	 ```java
		MeasureSpec.makeMeasureSpec( if (API<23){ 0 } else { 父View 的可用尺寸 }, MeasureSpec.UNSPECIFIED)
	```

## 总结
通过上面的分析, 在自定义控件时, 在测量步骤中, 大致流程如下
1. 如果是自定义View,  只需要结合其MeasureSpec以及具体的业务需求, 计算自己的宽高, 然后通过 setMeasuredDimension() 保存即可;
2. 如果是自定义ViewGroup, 那么测量步骤相对复杂一点;
	- 第一步: 需要先测量子View宽高:
    - 第二步: 根据逻辑要求, 计算出自己的宽高, 然后通过  setMeasuredDimension()进行保存
3. 系统为我们提供的关于测量的方法
```java
ViewGroup.measureChildren()
ViewGroup.measureChild()
ViewGroup.measureChildWithMargins()
ViewGroup.getChildMeasureSpec()
// 根据传入的size, 和MeasureSpec, 返回一个合理的size
View.resolveSize()
```
	