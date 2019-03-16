---
title: 自定义View-测量
top: true
cover: true
categories: Android
tags:
  - Android
  - View
date: 2019-03-16 09:49:42
img:
coverImg:
summary:
---

# 自定义View-测量
作为一名Android开发者, 自定义View是一项必须要掌握的技能. 虽然Github上已经有各种各样丰富的轮子, 可以满足日常开发中绝大多数需求. 但只有掌握了相关的原理, 才能让我们在碰到一些比较特殊的需求时, 不会过于被动.

在Android中View从创建到显示到屏幕, 需要经过三个步骤:
* **测量: ** 确定大小
* **布局: ** 确定位置
* **绘制: ** 确定内容

测量作为三部曲中的第一步, 重要性不言而喻. 首先我们要知道一点, Android中的View都是矩形. 包括图片(ImageView)、文字(TextView)、...各种列表、容器,
不管它外在的表现形式是什么, 它们的本质, 都是屏幕上一个一个矩形. 而测量的作用, 就是确定矩形的宽高.

## onMeasure
我们知道, 在定义一个View时, 关于测量的逻辑, 就在下面这个方法中完成
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	super.onMeasure(widthMeasureSpec, heightMeasureSpec);
}
```
***onMeasure()***中的两个参数***widthMeasureSpec, heightMeasureSpec***是指什么? int类型, 难道就是指宽高吗? 事实上它们确实和宽高有关, 但情况没那么简单, 要了解这两个参数, 就必须来谈谈 **MeasureSpec**.

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

## FramLayout测量步骤简析
有了上面的初步认识, 下面可以通过简单的研究下系统源码, 来印证我们的想法. 以FramLayout为例
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

总结上面的代码,  可以知道 子View的 MeasureSpec 取值分以下情况:
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
通过上面的分析, 在自定义ViewGroup/View时, 在测量步骤中, 大致流程如下
1. 如果是自定义View,  只需要结合其MeasureSpec以及具体的业务需求, 计算自己的宽高, 然后通过 setMeasuredDimension() 保存即可;
2. 如果是自定义ViewGroup, 那么测量步骤相对复杂一点;
	* 第一步需要先对子View进行测量, 在这个过程中可能会用到一下API:
	```java
	ViewGroup.measureChildren()
	ViewGroup.measureChild()
	ViewGroup.measureChildWithMargins()
	ViewGroup.getChildMeasureSpec()
	// 根据传入的size, 和MeasureSpec, 返回一个合理的size
	View.resolveSize()
	```
	* 根据逻辑要求, 计算出自己的宽高, 然后通过  setMeasuredDimension()进行保存