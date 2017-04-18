---
title: LBehavior
date: 2017-04-14 16:25:46
category: Android
tags: 
	- 自定义View
---



> Android Design包下的CoordinatorLayout是相当重要的一个控件，它让许多动画的实现变为可能，而且更加简便。按照官方解释CoordinatorLayout是用来协调子View交互动作的父view，Behavior可以看做CoordinatorLayout的子view实现交互的组件。
本篇博客主要用来实现仿知乎的Android客户端首页的滑动嵌套动画，前段时间利用空闲时间撸了一款干货集中营的客户端，做的时候采用自定义Behavior实现了整个嵌套滑动，并抽离了出来作为一个lib方便使用。

<!--more-->

## 先来一波效果图：
<img src="http://i.imgur.com/93zTA4s.gif" width = "270" height = "450" alt="效果图1" align=center /><img src="http://i.imgur.com/U02iHGv.gif" width = "270" height = "450" alt="效果图2" align=center />

## 效果实现思路：

1. 判断手势

2. 计算距离

3. 触发动画

## 文章目录：

1. CoordinatorLayout及Behavior简介
2. 自定义Behavior
3. 仿知乎效果的动画实现及个性化


## 1、CoordinatorLayout和Behavior简介

Android滑动嵌套的原理及Behavior分析已经有很多大神讲解过了，推荐Loader大神的[源码看CoordinatorLayout.Behavior原理](http://blog.csdn.net/qibin0506/article/details/50377592)。

这里简单介绍下,嵌套滑动时父View(需实现NestedScrollingParent接口)和子View(需实现NestedScrollingChild接口)之间的交互是由NestedScrolling两个接口控制,NestedScrollingParentHelper和NestedScrollingChildHelper两个辅助类分别处理了父布局和子View的大量逻辑。

滑动嵌套的简单流程为：控制子View(如RecyclerView)的onInterceptTouchEvent和onTouchEvent的事件分发 -> 调用NestedScrollingChildHelper不同的方法 -> 处理与NestedScrollingParent交互的逻辑 -> 父布局(如CoordinatorLayout)实现NestedScrollingParent处理具体的逻辑
 (-> 而Behavior的事件处理方法则主要由CoordinatorLayout的各种事件处理方法来调用,返回值控制了父布局的事件消费情况)。
 
具体方法的调用大家可以再研读Loader大神的博客。下边简单介绍下自定义Behavior实现的具体方法[Behavior官网](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html)。

### 方法
1.layoutDependsOn

确定提供的子视图是否具有另一个特定的兄弟视图作为布局依赖关系。即用来确定依赖关系，如果某个控件需要依赖控件，则重写改方法
如AppBarLayout

```java

	@Override
	public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
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
	此方法中consumed,指的是父布局要消费的滚动距离,consumed[0]为水平方向消耗的距离,consumed[1]为垂直方向消耗的距离,可控制此参数作出相应的调整。
	如垂直滑动时,若设置consumed[1]=dy,则代表父布局全部消耗了滑动的距离,类似AppBarLayout这种效果,当其由展开到折叠过渡时,通过consumed控制其中的嵌套滑动。

    /**
     * 触发滑动嵌套滚动之前调用的方法
     *
     * @param coordinatorLayout coordinatorLayout父布局
     * @param child             使用Behavior的子View
     * @param target            触发滑动嵌套的View(实现NestedScrollingChild接口)
     * @param dx                滑动的X轴距离
     * @param dy                滑动的Y轴距离
     * @param consumed          父布局消费的滑动距离，consumed[0]和consumed[1]代表X和Y方向父布局消费的距离，默认为0
     */
    @Override
    public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, View child, View target, int dx, int dy, int[] consumed) {
        super.onNestedPreScroll(coordinatorLayout, child, target, dx, dy, consumed);
    }
	
	
5.onNestedScroll
	此方法中dyConsumed代表TargetView消费的距离,如RecyclerView滑动的距离,可通过控制NestScrollingChild的滑动来指定一些动画,
	本篇博客实现的效果主要就是重写此方法,若根据onNestedPreScroll中dy来判断,则当RecyclerView条目很少时,也会触发逻辑代码,故选择了重写此方法。
	/**
     * 滑动嵌套滚动时触发的方法
     *
     * @param coordinatorLayout coordinatorLayout父布局
     * @param child             使用Behavior的子View
     * @param target            触发滑动嵌套的View
     * @param dxConsumed        TargetView消费的X轴距离
     * @param dyConsumed        TargetView消费的Y轴距离
     * @param dxUnconsumed      未被TargetView消费的X轴距离
     * @param dyUnconsumed      未被TargetView消费的Y轴距离(如RecyclerView已经到达顶部或底部，而用户继续滑动，此时dyUnconsumed的值不为0，可处理一些越界事件)
     */
    @Override
    public void onNestedScroll(CoordinatorLayout coordinatorLayout, View child, View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
        super.onNestedScroll(coordinatorLayout, child, target, dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed);
    }
	
## 2、自定义Behavior	
