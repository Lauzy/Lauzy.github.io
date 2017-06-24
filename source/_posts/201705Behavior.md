---
title: LBehavior
date: 2017-04-14 16:25:46
category: Android
tags: 
	- 自定义View
---



> Android Design包下的CoordinatorLayout是相当重要的一个控件，它让许多动画的实现变为可能，而且更加简便。按照官方解释CoordinatorLayout是用来协调子View交互动作的父view，Behavior可以看做CoordinatorLayout的子view实现交互的组件。
本篇博客主要用来实现仿知乎的Android客户端首页的滑动嵌套动画，前段时间利用空闲时间撸了一款[干货集中营的客户端](https://github.com/Lauzy/GankPro)，做的时候采用自定义Behavior实现了整个嵌套滑动，并抽离了出来作为一个lib方便使用。

<!--more-->

## 先来一波效果图：
<img src="http://oop6dcmck.bkt.clouddn.com/20170420B01.gif" width = "270" height = "450" alt="效果图1"/><img src="http://oop6dcmck.bkt.clouddn.com/20170420B02.gif" width = "270" height = "450" alt="效果图2"/>

## 效果实现思路：

1. 判断手势

2. 计算距离

3. 触发动画

## 文章目录：

1. CoordinatorLayout及Behavior简介
2. 自定义Behavior
3. 仿知乎效果的动画实现及个性化


## CoordinatorLayout和Behavior简介

Android滑动嵌套的原理及Behavior分析已经有很多大神讲解过了，推荐Loader大神的[源码看CoordinatorLayout.Behavior原理](http://blog.csdn.net/qibin0506/article/details/50377592)。

这里简单介绍下，嵌套滑动时父View(需实现NestedScrollingParent接口)和子View(需实现NestedScrollingChild接口)之间的交互是由NestedScrolling两个接口控制，NestedScrollingParentHelper和NestedScrollingChildHelper两个辅助类分别处理了父布局和子View的大量逻辑。

滑动嵌套的简单流程为：控制子View(如RecyclerView)的onInterceptTouchEvent和onTouchEvent的事件分发 -> 调用NestedScrollingChildHelper不同的方法 -> 处理与NestedScrollingParent交互的逻辑 -> 父布局(如CoordinatorLayout)实现NestedScrollingParent处理具体的逻辑
 (-> 而Behavior的事件处理方法则主要由CoordinatorLayout的各种事件处理方法来调用，返回值控制了父布局的事件消费情况)。
 
具体方法的调用大家可以再研读Loader大神的博客。下边简单介绍下自定义Behavior实现的具体方法[Behavior官网](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html)。

### 方法

#### 1.layoutDependsOn

确定提供的子视图是否具有另一个特定的兄弟视图作为布局依赖关系。即用来确定依赖关系，如果某个控件需要依赖控件，则重写该方法
如AppBarLayout

```java

	@Override
	public boolean layoutDependsOn(CoordinatorLayout parent， View child， View dependency) {
		return dependency instanceof AppBarLayout;
	}

```

#### 2.onDependentViewChanged

依赖视图的大小、位置发生变化时调用此方法，重写此方法可以处理child的响应。如常用的AppBarLayout，当其发生变化时，childView会根据重写的方法作出响应。

```java

	@Override
	public boolean onDependentViewChanged(CoordinatorLayout parent， View child， View dependency) {
		offsetChildAsNeeded(parent， child， dependency);
		return false;
	}
```

#### 3.onStartNestedScroll

当CoordinatorLayout的子View开始嵌套滑动时（此处的滑动View必须实现NestedScrollingChild接口），触发此方法。添加Behavior的控件需要为CoordinatorLayout的直接子View，否则不会继续流程。

```java

	//判断是否垂直滑动
	@Override
    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout， View child， View directTargetChild， View target， int nestedScrollAxes) {
        return (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;
    }
```

#### 4.onNestedPreScroll

此方法中consumed，指的是父布局要消费的滚动距离，consumed[0]为水平方向消耗的距离，consumed[1]为垂直方向消耗的距离，可控制此参数作出相应的调整。
如垂直滑动时，若设置consumed[1]=dy，则代表父布局全部消耗了滑动的距离，类似AppBarLayout这种效果，当其由展开到折叠过渡时，通过consumed控制其中的嵌套滑动。

```java

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
    public void onNestedPreScroll(CoordinatorLayout coordinatorLayout， View child， View target， 
		int dx， int dy， int[] consumed) {
        super.onNestedPreScroll(coordinatorLayout， child， target， dx， dy， consumed);
    }
```
	

#### 5.onNestedScroll

此方法中dyConsumed代表TargetView消费的距离，如RecyclerView滑动的距离，可通过控制NestScrollingChild的滑动来指定一些动画，
本篇博客实现的效果主要就是重写此方法，若根据onNestedPreScroll中dy来判断，则当RecyclerView条目很少时，也会触发逻辑代码，故选择了重写此方法。
	
```java
	
	/**
     * 滑动嵌套滚动时触发的方法
     *
     * @param coordinatorLayout coordinatorLayout父布局
     * @param child             使用Behavior的子View
     * @param target            触发滑动嵌套的View
     * @param dxConsumed        TargetView消费的X轴距离
     * @param dyConsumed        TargetView消费的Y轴距离
     * @param dxUnconsumed      未被TargetView消费的X轴距离
     * @param dyUnconsumed      未被TargetView消费的Y轴距离(如RecyclerView已经到达顶部或底部，
	 *				而用户继续滑动，此时dyUnconsumed的值不为0，可处理一些越界事件)
     */
    @Override
    public void onNestedScroll(CoordinatorLayout coordinatorLayout， View child， View target， 
		int dxConsumed， int dyConsumed， int dxUnconsumed， int dyUnconsumed) {
        super.onNestedScroll(coordinatorLayout， child， target， 
			dxConsumed， dyConsumed， dxUnconsumed， dyUnconsumed);
    }

```
	
## 自定义Behavior

### 自定义Behavior主要有两种实现方式：

第一种为layoutDependsOn和onDependentViewChanged，child需要依赖于dependency，当dependency View发生变化时，onDependentViewChanged会被调用，child可作出响应的响应。
第二种为onStartNestedScroll 等嵌套滑动的流程，首先在onStartNestedScroll方法中判断是否垂直滑动等，然后在onNestedPreScroll、onNestedScroll等方法中实现效果。
由于第一种方式会导致child必须依赖于某个特定的View，这样就导致灵活性不太强，所以本文采用第二种实现方式。

### 具体实现

在嵌套滑动开始之前，可以判断是否垂直滑动，做一些初始化工作，比如获取childView的初始坐标。

```java

	//判断垂直滑动
    @Override
    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, View child, View directTargetChild, View target, int nestedScrollAxes) {
        if (isInit) {// 设置标记，防止new Anim导致的parent和child坐标变化
            mCommonAnim = new LTitleBehaviorAnim(child);
            isInit = false;
        }
        return (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;
    }
	
```

触发嵌套滑动之前，可以在此处判断一些滑动手势，以及父布局的消费情况。由于若根据此方法中dy来判断，则当RecyclerView条目很少时，也会触发逻辑代码，故本文只是在此方法中给动画做一些自定义操作。

```java

	@Override
    public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, View child, View target, int dx, int dy, int[] consumed) {
        if (mCommonAnim != null) {
            mCommonAnim.setDuration(mDuration);
            mCommonAnim.setInterpolator(mInterpolator);
        }
        super.onNestedPreScroll(coordinatorLayout, child, target, dx, dy, consumed);
    }

```

滑动嵌套滚动时触发的方法，以Title(Toolbar)为例，若向上滑动，则隐藏Toolbar，反之显示。

```java

	@Override
    public void onNestedScroll(CoordinatorLayout coordinatorLayout, View child, View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
        super.onNestedScroll(coordinatorLayout, child, target, dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed);
		if (dyConsumed < 0) {
            if (isHide) {
                mCommonAnim.show();
                isHide = false;
            }
        } else if (dyConsumed > 0) {
            if (!isHide) {
                mCommonAnim.hide();
                isHide = true;
            }
        }
    }


```


## 仿知乎效果的动画实现及个性化

大家都知道知乎客户端的各种动画非常优雅，网上仿写其动画的博客也是层出不穷，之前利用空闲时间撸了一款[干货集中营客户端](https://github.com/Lauzy/GankPro)，突然想到了采用知乎的首页效果，然后就拿起键盘，复制粘贴搞了起来。
开个玩笑，其实大致实现效果还是比较容易的，这里主要分享下实现的思路以及需要注意的细节。

首先大致流程就如上边几个方法介绍，动画效果的实现也非常简单，这里以显示和隐藏BottomView为例，直接上代码。

```java

	public LBottomBehaviorAnim(View bottomView) {
        mBottomView = bottomView;
        mOriginalY = mBottomView.getY();//因为Y值随动画会发生变化，嵌套滑动开始之前先记录初始的坐标。
    }

	@Override
    public void show() {//显示
        ValueAnimator animator = ValueAnimator.ofFloat(mBottomView.getY(), mOriginalY);
        animator.setDuration(getDuration());
        animator.setInterpolator(getInterpolator());
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator) {
                mBottomView.setY((Float) valueAnimator.getAnimatedValue());
            }
        });
        animator.start();
    }

    @Override
    public void hide() {//隐藏
        ValueAnimator animator = ValueAnimator.ofFloat(mBottomView.getY(), mOriginalY + mBottomView.getHeight());
        animator.setDuration(getDuration());
        animator.setInterpolator(getInterpolator());
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator) {
                mBottomView.setY((Float) valueAnimator.getAnimatedValue());
            }
        });
        animator.start();
    }

```

整个大致流程这样其实已经结束了，但是还达不到我们预期的效果。再次打开知乎客户端，以很缓慢的速度滑一滑，这时候你会发现竟然没有触发动画，OK，先记录下这个问题；再以很缓慢的速度向下滑，突然又触发动画了。整体来看，知乎的动画有种分层嵌套的效果。

先来解决第一个问题，只用加一行代码，即dyConsumed距离大于一定值的时候才允许滑动。

```java

	if(Math.abs(dyConsumed) > minScrollY){
		...//onNestedScroll里边的逻辑代码
	}

```

对于第二个问题，我一开始想，滑动一定的距离，难道要根据判断RecyclerView滑动的距离来判断是否触发动画？其实思路是正确的，但是我们不可能再去实现addOnScrollListener的一系列方法。这时候再想一想嵌套滑动，dyConsumed不就是recyclerView消费的距离吗，想到这里，那就很好实现了，只用将dyConsumed相加，相加的和大于一定值，就触发动画，代码也是很简单，结合第一个问题，知乎的效果就实现了。

```java

	mTotalScrollY += dyConsumed;//累加消费的距离
    if (Math.abs(dyConsumed) > minScrollY || Math.abs(mTotalScrollY) > scrollYDistance) {
		...//onNestedScroll里边的逻辑代码
        mTotalScrollY = 0;//动画执行完毕后重置
    }

```

接下来我们可以自定义设置一些属性值。首先要获取这个Behavior对象。

```java

	public static CommonBehavior from(View view) {
        ViewGroup.LayoutParams params = view.getLayoutParams();
        if (!(params instanceof CoordinatorLayout.LayoutParams)) {
            throw new IllegalArgumentException("The view is not a child of CoordinatorLayout");
        }
        CoordinatorLayout.Behavior behavior = ((CoordinatorLayout.LayoutParams) params).getBehavior();
        if (!(behavior instanceof CommonBehavior)) {
            throw new IllegalArgumentException("The view's behavior isn't an instance of CommonBehavior. Try to check the [app:layout_behavior]");
        }
        return (CommonBehavior) behavior;
    }

```

然后可以设置对象的属性：

```java

	public CommonBehavior setDuration(int duration) {
        mDuration = duration;
        return this;
    }

    public CommonBehavior setInterpolator(Interpolator interpolator) {
        mInterpolator = interpolator;
        return this;
    }

    public CommonBehavior setMinScrollY(int minScrollY) {
        this.minScrollY = minScrollY;
        return this;
    }

    public CommonBehavior setScrollYDistance(int scrollYDistance) {
        this.scrollYDistance = scrollYDistance;
        return this;
    }
	
```


至此，整个流程已经实现了，其他TitleView及悬浮按钮的动画也是类似的规则，我又给Behavior和动画设置了Common类剔除掉一些重复代码，这里就不贴出来了。具体可以参考[我的Github](https://github.com/Lauzy/LBehavior)

动画已经实现，但是写代码的时候坑貌似永远是填不完的。

当我使用写出来的动画时，就发现了一个问题，由于是CoordinatorLayout作为根布局，所以RecyclerView顶部的item被toolbar遮挡了，
我们再看看知乎，轻轻滑动一小段距离，发现他的顶部Toolbar遮挡的地方其实是空白，可以发现知乎其实也是有这个问题的，不过人家处理的很好，所以用户基本上不会发现。
不过这个问题还是可以解决的，比如判断item为第一个时，可以加一个View填充，个人采用的自定义ItemDecoration，判断下若为第一个item，outRect.set(0, titleHeight, 0, 0)，设置titleHeight的大小即可。BottomView也是同理，解决方法也是有不少的。

还有一个问题是写demo的时候发现的，我用LinearLayout作为BottomView，发现浮动按钮竟然是在LinearLayout上层执行各种动画，看起来不太和谐，后来发现FloatingActionButton的elevation若大于BottomView的elevation，则FloatingActionButton动画覆盖在BottomView上层，反之则在下层。之前却一直没有注意。

此外，当知乎的RecyclerView滑动到底部的时候，BottomView是会自动显示的，个人觉得可以根据dyUnconsumed的值或者onStopNestedScroll来判断RecyclerView是否滑动到底部来处理，全部加载完毕后再处理最后一个item的ItemDecoration，本文并没有具体实现，只是提供思路。

我们再来整理下解决问题的思路：首先想好做什么，然后研究原理，选择方案，再初步实现，继续优化细节，最后应用到项目。我想我们写程序的时候都应该这样，知其然知其所以然，做到举一反三。


整个流程就是这样，实现后再封装，然后呢，抽离出来提交到Github。

```java

	allprojects {
	    repositories {
		    ...
		    maven { url 'https://jitpack.io' }
	    }
	}

    dependencies {
	    compile 'com.github.Lauzy:LBehavior:1.0.1'
	}

```

具体使用也很简单

参数     							|	说明
-----------------------------------|-----------------------
@string/title_view_behavior   		|   顶部标题栏
@string/bottom_view_behavior   	|   底部导航栏
@string/fab_scale_behavior   		|   浮动按钮（缩放）
@string/fab_vertical_behavior   	|    浮动按钮（上下滑动）



自定义(均设有默认值，可不使用)：


| 方法           	 		|    参数           	| 说明  					|
| ------------------------- |------------------ | --------------------- |
| setMinScrollY				| int y 			| 设置触发动画的最小滑动距离，如 setMinScrollY(10)为滑动10像素才可触发动画，默认为5.|
| setScrollYDistance		| int y      	    | 设置触发动画的滑动距离，防止用户缓慢滑动时单次滑动距离一直小于setMinScrollY的最小滑动距离导致无法触发动画.如设置此值为100，则用户即便缓慢滑动，当滑动距离达到100时也可触发动画.默认为40.|
| setDuration				| int duration     	| 设置动画持续时间.默认为400ms.|
| setInterpolator			| Interpolator interpolator | 设置动画插补器，修饰动画效果.默认模式为LinearOutSlowInInterpolator. [Interpolator官方文档](https://developer.android.google.cn/reference/android/view/animation/Interpolator.html)|

```java

	CommonBehavior.from(mFloatingActionButton)
		.setMinScrollY(20)
		.setScrollYDistance(100)
		.setDuration(1000)
		.setInterpolator(new LinearOutSlowInInterpolator());
		
```

最后附上项目的地址，戳  [我的Github](https://github.com/Lauzy/LBehavior) ，顺便可以看看撸的[干货集中营客户端](https://github.com/Lauzy/GankPro)。