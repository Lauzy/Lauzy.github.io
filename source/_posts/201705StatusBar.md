---
title: 沉浸、透明及白底黑字状态栏技巧
date: 2017-03-30 09:25:46
tags: 
	- Android Tips
---


我们知道，Android5.0以上已经可以通过设置colorPrimaryDark来改变状态栏的颜色。但是，大多数Android开发者都会遇到让系统状态栏透明或者半透明的需求，如下图所示:

![透明](http://oop6dcmck.bkt.clouddn.com/20170421%E9%80%8F%E6%98%8E%E6%95%88%E6%9E%9C.png)

![半透明](http://oop6dcmck.bkt.clouddn.com/20170421%E5%8D%8A%E9%80%8F%E6%98%8E.png)

<!--more-->

读这篇文章之前建议研读郭神的[Android状态栏微技巧，带你真正理解沉浸式模式](http://blog.csdn.net/guolin_blog/article/details/51763825)。这里我们坚决不考虑4.4以下的系统，那么基本上可以分为几种情况，4.4、 5.0以上6.0以下、 6.0+。

本篇文章目录：
1.透明与半透明状态栏
2.结合Toolbar使用技巧
3.白底黑字状态栏
4.其他注意事项

## 透明与半透明状态栏

首先我们实现上边两图的效果，注意下版本的判断，直接上代码：

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
	View decorView = getWindow().getDecorView();
	int option = View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
		| View.SYSTEM_UI_FLAG_LAYOUT_STABLE;
	decorView.setSystemUiVisibility(option);
	getWindow().setStatusBarColor(Color.TRANSPARENT);
	//getWindow().setStatusBarColor(Color.parseColor("#40000000"));  //此种效果为类似QQ的半透明状态栏
} else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
	getWindow().setFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS, WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
}

```

版本号为21以上时，效果便如上边两图所示，4.4系统则如下图所示：

![4.4效果](http://oop6dcmck.bkt.clouddn.com/20170421%E9%80%8F%E6%98%8E4.4%E6%95%88%E6%9E%9C.png)


## 结合标题栏使用的技巧

上述方法基本上已经实现了透明状态栏的效果，但是我们添加一个Toolbar，设置各种属性后，各种元素都上移了，如下图所示：

![上移效果](http://oop6dcmck.bkt.clouddn.com/20170421%E4%B8%8D%E8%AE%BE%E7%BD%AEPadding%E6%95%88%E6%9E%9C.png)

部分开发者用android:fitsSystemWindows = "true"，然后给状态栏添加一个View来解决，经本人各种实验后，发现这种方法不太适合自己。
这里采用一个比较直接的方式，设置透明后相当于toolbar已经被拉长，所以为其设置一个padding值来解决问题，代码如下，ScreenUtils为获取屏幕属性的一个工具类：

```java 
mToolbar = (Toolbar) findViewById(R.id.toolbar_common);
if (mToolbar != null) {
	mToolbar.getLayoutParams().height += ScreenUtils.getStatusHeight(getApplicationContext());
	mToolbar.setPadding(0, ScreenUtils.getStatusHeight(getApplicationContext()), 0, 0);
	setSupportActionBar(mToolbar);
	ActionBar supportActionBar = getSupportActionBar();
	if (supportActionBar != null) {
		supportActionBar.setDisplayShowTitleEnabled(false);  //此处是为了不显示默认的标题
	}
}

```

设置之后，就达到我们想要的效果了。

## 白底黑字状态栏

有时候我们的toolbar背景为浅色甚至是白色，如果不加修饰的话，状态栏的文字由于默认白色，导致很难分辨。6.0以上系统直接提供了方法，但是考虑到其他版本，需要具体判断:

```java
//设置状态栏文字为暗色
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
	//6.0以上可以通过直接设置SYSTEM_UI_FLAG_LIGHT_STATUS_BAR属性即可。
	getWindow().getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
} else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
	getWindow().setStatusBarColor(Color.GRAY);  //21以上不支持6.0直接设置的方法，可用灰色代替，具体可自己设置
	//getWindow().setStatusBarColor(Color.parseColor("#40000000"));
} else  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN){
	getWindow().getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN);//4.4版本本身就含有暗色阴影，不作其他处理即可
}

```

效果图如下：
6.0效果：![6.0白色](http://oop6dcmck.bkt.clouddn.com/20170421%E7%99%BD%E8%89%B26.0%E6%95%88%E6%9E%9C.png)
5.0以上6.0以下： ![5.0以上6.0以下](http://oop6dcmck.bkt.clouddn.com/201704215.0%E4%BB%A5%E4%B8%8A6.0%E4%BB%A5%E4%B8%8B%E7%99%BD%E8%89%B2.png)
4.4效果： ![4.4效果](http://oop6dcmck.bkt.clouddn.com/20170421%E7%99%BD%E8%89%B24.4%E6%95%88%E6%9E%9C.png)

## 注意事项

上述方法基本上已经可以满足大部分需求了，效果都可以称为(半)透明状态栏。下边看看真正的沉浸式状态栏，状态栏文字和导航栏都被隐藏：

![沉浸](http://oop6dcmck.bkt.clouddn.com/20170421%E6%B2%89%E6%B5%B8.png)

代码如下：

```java 

@Override
public void onWindowFocusChanged(boolean hasFocus) {
	super.onWindowFocusChanged(hasFocus);
	if (hasFocus && Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
		View decorView = getWindow().getDecorView();
		decorView.setSystemUiVisibility(
		View.SYSTEM_UI_FLAG_LAYOUT_STABLE
		| View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
		| View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
		| View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
		| View.SYSTEM_UI_FLAG_FULLSCREEN
		| View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY);
	}
}

```

实际开发中，我们当然不可能每个Activity中都写一大堆重复的代码，所以建议讲透明状态栏的代码封装在BaseActivity中，然后Toolbar设置相同的id，或者使用<include>标签公用，自定义的一个布局或者其他View也是同理。
Demo中做了一些简单的封装，可以参考下，具体实现还是需要看具体的需求。

本文所有代码的地址,戳 [我的Github](https://github.com/Lauzy/LauzyCode) ，在StatusBar包中。
