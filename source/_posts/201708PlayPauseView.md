---
title: Android自定义View：让播放、暂停按钮优雅的过渡
date: 2017-08-12 19:42:27
category: Android
toc: false
tags: Android 自定义View
---

最近想写个音乐播放器，偶然看到轻听这款播放器的播放和暂停按钮，在切换过程中的动画很是吸引我。本着造轮子（其实是 github 上边没找到）的想法，就花了点时间撸出来了这个效果。

效果就是下边这个样子：

<img src="http://oop6dcmck.bkt.clouddn.com/20170812PlayPauseView.gif" width = "200" height = "210" alt="效果图"/>

<!--more-->

下边说下实现方法，中间也踩了一些坑。

## 测量及初始化

首先要确实View的宽高，在这里由于是圆形按钮，所以设置宽高相等，onMeasure()方法中设置下即可：

```java

 		mWidth = MeasureSpec.getSize(widthMeasureSpec);
        mHeight = MeasureSpec.getSize(heightMeasureSpec);
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        switch (widthMode) {
            case MeasureSpec.EXACTLY:
                mWidth = mHeight = Math.min(mWidth, mHeight);
                setMeasuredDimension(mWidth, mHeight);
                break;
            case MeasureSpec.AT_MOST:
                float density = getResources().getDisplayMetrics().density;
                mWidth = mHeight = (int) (50 * density); //默认50dp
                setMeasuredDimension(mWidth, mHeight);
                break;
        }

```

然后画出底部的圆形

```java

 canvas.drawCircle(mWidth / 2, mHeight / 2, mRadius, mPaint);

```


## 计算Path

1、初始化完毕后，怎么实现两个竖条到一个三角形的过渡呢？这里首先想到的就是自定义 View 常用的 drawPath 方法，抛开动画不谈，整个 View 变化过程其实就是两个矩形变成两个直角三角形的过程。


<img src="http://oop6dcmck.bkt.clouddn.com/20170813PlayPauseViewBlog001.png" alt = "实现">

就是这个样子。知道大体的思路，怎么搞呢，当然是开车了。

<img src="http://oop6dcmck.bkt.clouddn.com/FACE001.gif">

就是 canvas.drawPath();

首先计算暂停时两个矩形的各个坐标位置：

```java

 		float distance = mGapWidth;  //暂停时左右两边矩形距离
        float barWidth = mRectWidth / 2 - distance / 2;     //一个矩形的宽度
        float leftLeftTop = barWidth;       //左边矩形左上角

        float rightLeftTop = barWidth + distance;       //右边矩形左上角
        float rightRightTop = 2 * barWidth + distance;  //右边矩形右上角
        float rightRightBottom = rightRightTop; //右边矩形右下角

```

bottom 的话直接加上矩形的高度即可。

```java

			mLeftPath.moveTo(0, 0);
            mLeftPath.lineTo(leftLeftTop, mRectHeight);
            mLeftPath.lineTo(barWidth, mRectHeight);
            mLeftPath.lineTo(barWidth, 0);
            mLeftPath.close();

            mRightPath.moveTo(rightLeftTop, 0);
            mRightPath.lineTo(rightLeftTop, mRectHeight);
            mRightPath.lineTo(rightRightBottom, mRectHeight);
            mRightPath.lineTo(rightRightTop, 0);
            mRightPath.close();

```

这样两个竖条就出来了。


2、在一开始写的时候就写了这么多计算的方法，但是这时候矩形的边角会超出 View 的范围，所以后来计算了一波位置：

<img src="http://oop6dcmck.bkt.clouddn.com/20170813PlayPauseViewBlog01.png"  alt = "计算过程1">

如上图所示，这样就需要再更改一些参数：

<img src="http://oop6dcmck.bkt.clouddn.com/20170813PlayPauseViewBlog02.png"  alt = "计算过程2">


首先定义出来这个矩形，计算下宽高：

```java

 		float space = (float) (mRadius / Math.sqrt(2)); 
        mRectLT = (int) (mRadius - space);
        int rectRB = (int) (mRadius + space);
        mRect.top = mRectLT;
        mRect.bottom = rectRB;
        mRect.left = mRectLT;
        mRect.right = rectRB;


```

然后只用在 确定 path 的路线时更改下坐标就可以了：

```java

 			mLeftPath.moveTo(mRectLT, mRectLT);
            mLeftPath.lineTo(leftLeftTop + mRectLT, mRectHeight + mRectLT);
            mLeftPath.lineTo(barWidth + mRectLT, mRectHeight + mRectLT);
            mLeftPath.lineTo(barWidth + mRectLT, mRectLT);
            mLeftPath.close();

            mRightPath.moveTo(rightLeftTop + mRectLT, mRectLT);
            mRightPath.lineTo(rightLeftTop + mRectLT, mRectHeight + mRectLT);
            mRightPath.lineTo(rightRightBottom + mRectLT, mRectHeight + mRectLT);
            mRightPath.lineTo(rightRightTop + mRectLT, mRectLT);
            mRightPath.close();

```

这时候画出来两个 Path，暂停按钮就完美的呈现了：

```java

		canvas.drawPath(mLeftPath, mPaint);
        canvas.drawPath(mRightPath, mPaint);

```

## 动画实现

画完暂停按钮后，怎么让他动画变成三角形呢？一开始我想根据一些宽高的属性来指定动画的变化值，然后更新过程中再画出来，但是计算过程中发现涉及动画的矩形宽度都是从原始的大小到0过渡的，那统一的使用一个参数确定会不会更好点呢？当然会了。

<img src="http://oop6dcmck.bkt.clouddn.com/FACE002.jpg">

这时候就可以设置动画属性了：

```java

		ValueAnimator valueAnimator = ValueAnimator.ofFloat(0 : 1);
        valueAnimator.setDuration(200);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                mProgress = (float) animation.getAnimatedValue();
                invalidate();
            }
        });

```

然后根据 progress 在更新View的过程中来更改矩形的宽高值：

```java

		float distance = mGapWidth * (1 - mProgress);  //暂停时左右两边矩形距离
        float barWidth = mRectWidth / 2 - distance / 2;     //一个矩形的宽度
        float leftLeftTop = barWidth * mProgress;       //左边矩形左上角

        float rightLeftTop = barWidth + distance;       //右边矩形左上角
        float rightRightTop = 2 * barWidth + distance;  //右边矩形右上角
        float rightRightBottom = rightRightTop - barWidth * mProgress; //右边矩形右下角

```

这样便可以实现两个矩形到三角形的过渡了，执行动画结束后便是这个样子：

<img src="http://oop6dcmck.bkt.clouddn.com/20170813PlayPauseViewBlog03.png"  alt = "计算过程3">

两个矩形变成三角形之后，只需要画布旋转一下，两个暂停按钮到播放按钮的动画已经可以执行了：

```java

canvas.rotate(rotation, mWidth / 2f, mHeight / 2f);

```

到这里基本上已经结束了，但是写完使用的时候总觉得位置有点不对劲，后来发现确实有问题：


<img src="http://oop6dcmck.bkt.clouddn.com/20170813PlayPauseViewBlog04.png"  alt = "计算过程3">

如图所示，旋转过后 A 和 C 本来是紧靠着圆周的，而 B 距离圆周还有一定的距离。所以需要将其位移 x 的距离，这时候需要 CE 的长度等于 BD 的长度。这时候就需要做个数学题了，利用勾股定理即可。电脑上打数学符号太麻烦，所以就手写了这个公式：

<img src="http://oop6dcmck.bkt.clouddn.com/20170813PlayPauseViewBlog05.png">


如上，计算出来 x 的值为 √2 x r/8， 即 radius x Math.sqrt(2) / 8f 。

然后在画布位移一下即可：

```java

canvas.translate((float) (mRectHeight * Math.sqrt(2) / 8f * mProgress), 0);

```

## 总结

上边几个步骤写完，整体效果已经实现了。后来又设置了一系列自定义的参数方便使用：

```java

	<declare-styleable name="PlayPauseView">
        <attr name="bg_color" format="color"/>
        <attr name="btn_color" format="color"/>
        <attr name="gap_width" format="float"/>
        <attr name="space_padding" format="float"/>
        <attr name="anim_duration" format="integer"/>
        <attr name="anim_direction">
            <enum name="positive" value="1"/>
            <enum name="negative" value="2"/>
        </attr>
    </declare-styleable>

```

所有代码都已经上传到 [我的Github](https://github.com/Lauzy) 上边了，[点击可查看](https://github.com/Lauzy/PlayPauseView)，希望提出问题相互讨论，随便给个 Star 再好不过了。
有问题交流可加QQ群 661614986 ，欢迎讨论。