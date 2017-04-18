---
title: LBehavior
date: 2017-04-14 16:25:46
category: Android
tags: 
	- 自定义View
---



> Android Design包下的CoordinatorLayout是相当重要的一个控件，它让许多动画的实现变为可能，而且更加简便。按照官方解释CoordinatorLayout是用来协调子View交互动作的父view，Behavior就是用来给CoordinatorLayout的子view们实现交互的。
本篇博客主要用来实现仿知乎的Android客户端首页的滑动嵌套动画，前段时间利用空闲时间撸了一款干货集中营的客户端，做的时候采用自定义Behavior实现了整个嵌套滑动，并抽离了出来作为一个lib方便使用。

## 先来一波效果图：
<img src="http://i.imgur.com/93zTA4s.gif" width = "270" height = "450" alt="效果图1" align=center />    <img src="http://i.imgur.com/U02iHGv.gif" width = "270" height = "450" alt="效果图2" align=center />

## 效果实现思路：

1. 判断手势

2. 计算距离

3. 触发动画

## 文章目录：

1. CoordinatorLayout及Behavior简介
2. 自定义Behavior
3. 仿知乎效果的动画实现及个性化


### 1、CoordinatorLayout和Behavior简介

Android滑动嵌套的原理及Behavior分析已经有很多大神讲解过了，这里推荐鸿神的[Android NestedScrolling机制完全解析](http://blog.csdn.net/lmj623565791/article/details/52204039)以及Loader大神的[源码看CoordinatorLayout.Behavior原理](http://blog.csdn.net/qibin0506/article/details/50377592)。本篇主要介绍下具体的实现方法：
[Behavior官网](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html)

### 方法
1.layoutDependsOn

确定提供的子视图是否具有另一个特定的兄弟视图作为布局依赖关系。即用来确定依赖关系，如果某个控件需要依赖控件，则重写改方法
如AppBarLayout

```java

	@Override
	public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
		// We depend on any AppBarLayouts
		return dependency instanceof AppBarLayout;
	}

```

2.onDependentViewChanged
依赖视图的大小、位置发生变化时调用此方法，重写此方法可以处理child的响应。如常用的AppBarLayout，当其发生变化时，childView会根据重写的方法作出响应。

```java

	@Override
	public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
		offsetChildAsNeeded(parent, child, dependency);
		return false;
	}
```

3.onStartNestedScroll
当CoordinatorLayout的子View开始嵌套滑动时（此处的滑动View必须实现NestedScrollingChild接口），触发此方法。添加Behavior的控件需要为CoordinatorLayout的直接子View，否则不会继续流程。

```java

	//判断是否垂直滑动
	@Override
    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, View child, View directTargetChild, View target, int nestedScrollAxes) {
        return (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;
    }
```
4.onNestedPreScroll
	触发滑动嵌套滚动之前调用的方法。可以在此处指定 consumed,指的是父布局要消费的滚动距离,consumed[0]为水平方向消耗的距离,consumed[1]为垂直方向消耗的距离,可控制此参数作出相应的调整。
	如垂直滑动时,若设置consumed[1]=dy,则代表子View全部消耗了滑动的距离,此时

    /**
     * 在嵌套滑动的子View未滑动之前告诉过来的准备滑动的情况
     * @param target 具体嵌套滑动的那个子类
     * @param dx 水平方向嵌套滑动的子View想要变化的距离
     * @param dy 垂直方向嵌套滑动的子View想要变化的距离
     * @param consumed 这个参数要我们在实现这个函数的时候指定，回头告诉子View当前父View消耗的距离 
     *                    consumed[0] 水平消耗的距离，consumed[1] 垂直消耗的距离 好让子view做出相应的调整
     */
    public void onNestedPreScroll
	
5.onNestedScroll
	/**
     * 嵌套滑动的子View在滑动之后报告过来的滑动情况
     *
     * @param target 具体嵌套滑动的那个子类
     * @param dxConsumed 水平方向嵌套滑动的子View滑动的距离(消耗的距离)
     * @param dyConsumed 垂直方向嵌套滑动的子View滑动的距离(消耗的距离)
     * @param dxUnconsumed 水平方向嵌套滑动的子View未滑动的距离(未消耗的距离)
     * @param dyUnconsumed 垂直方向嵌套滑动的子View未滑动的距离(未消耗的距离)
     */
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
                               int dxUnconsumed, int dyUnconsumed);
|onStartNestedScroll|
|onNestedPreScroll|
|onNestedScroll|