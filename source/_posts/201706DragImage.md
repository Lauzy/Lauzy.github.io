---
title: 拖拽图片
date: 2017-05-05 15:34:59
category: Android
toc: false
tags: Android 交互
---

闲来无事，在[干货集中营](https://github.com/Lauzy/GankPro)里又撸了个效果。

先来一波效果图：

<img src="http://oop6dcmck.bkt.clouddn.com/20170502Gank2.gif" width = "270" height = "450" alt="效果图"/>

<!--more-->

实现方法当然是手势交互，那自然想到OnTouchListener()，流程还是很简单的，继承OnTouchListener，然后重写onTouch方法，由于效果相对简单，这里直接上代码：

```java

	public boolean onTouch(View v, MotionEvent event) {
	
		//获取View的LayoutParams，注意此种方法仅适合FrameLayout，这里仅是一种简单的实现思路。
        FrameLayout.LayoutParams layoutParams = (FrameLayout.LayoutParams) v.getLayoutParams();
        switch (event.getActionMasked()) {
            case MotionEvent.ACTION_DOWN:
			//获取手指按下时X和Y轴的坐标
                mOriginalY = (int) event.getRawY();
                mOriginalX = (int) event.getRawX();
                break;
            case MotionEvent.ACTION_MOVE:
			//获取移动的距离
                mMotionY = (int) (event.getRawY() - mOriginalY);
                mMotionX = (int) (event.getRawX() - mOriginalX);
			//计算缩放比例
                float ratio = Math.abs(mMotionY) * 1.0f / v.getHeight();
                float ratioY = Math.abs(mMotionY) <= v.getHeight() ? (ratio <= 0.5f ? ratio : 0.5f) : 0.5f;
            //根据Y轴变化缩放比例
                v.setScaleX(1 - ratioY);
                v.setScaleY(1 - ratioY);
			//设置layoutParams变化
                layoutParams.topMargin = mMotionY / 2;
                layoutParams.leftMargin = mMotionX / 2;
                layoutParams.bottomMargin = -mMotionY / 2;
                layoutParams.rightMargin = -mMotionX / 2;
                v.setLayoutParams(layoutParams);
			//调用requestLayout 重置布局
                v.requestLayout();
			//设置透明度最低值
                if (Math.abs(mMotionY) > 500) {
                    mContentLayout.getBackground().setAlpha(100);
                    mCurAlpha = 100;
                } else {
				//根据移动距离计算透明度
                    float ratioAlpha = (Math.abs(mMotionY) / 500.0f) * (255 - 100);
                    mContentLayout.getBackground().setAlpha(255 - (int) ratioAlpha);
                    mCurAlpha = 255 - (int) ratioAlpha;
                }
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
			//手势抬起或者取消时，若移动距离大于400，实现自定义的接口方法
                if (Math.abs(mMotionY) > 400) {
                    mImageEventListener.onActionBack();
                } else {
                //无动画返回原状
                    /*v.setScaleY(1);
                    v.setScaleX(1);
                    layoutParams.topMargin = 0;
                    layoutParams.bottomMargin = 0;
                    layoutParams.leftMargin = 0;
                    layoutParams.rightMargin = 0;
                    mContentLayout.getBackground().setAlpha(255);*/
				//通过一系列动画将View复原
                    setScaleAnim(v);
                    setMarginAnim(v, layoutParams, Direction.LEFT);
                    setMarginAnim(v, layoutParams, Direction.RIGHT);
                    setMarginAnim(v, layoutParams, Direction.TOP);
                    setMarginAnim(v, layoutParams, Direction.BOTTOM);
                    v.requestLayout();
                }
                ValueAnimator animator = ValueAnimator.ofInt(mCurAlpha, 255);
                animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                    @Override
                    public void onAnimationUpdate(ValueAnimator animation) {
                        mContentLayout.getBackground().setAlpha((Integer) animation.getAnimatedValue());
                    }
                });
                animator.setDuration(300).start();
                break;
        }
	//消费事件
        return true;
    }

```


然后在使用的地方直接调用即可：

```java
		imageView.setOnTouchListener(new DragImageOnTouchListener(frameLayout, new DragImageOnTouchListener.ImageEventListener() {
            @Override
            public void onActionBack() {
                onBackPressed();
				//可处理具体的逻辑
            }
        }));
```

注意：此方法是根据LayoutParams重置布局实现，所以使用FrameLayout可达到图片所示效果。若使用其他布局，如RelativeLayout，则需要先初始化RelativeLayout.LayoutParams，使得视图不便控制。
具体代码可参考  [我的Github](https://github.com/Lauzy/LauzyCode) 的[DragImage包下](https://github.com/Lauzy/LauzyCode/tree/master/app/src/main/java/com/lauzy/freedom/lauzycode/DragImage),
在我的[干货集中营](https://github.com/Lauzy/GankPro) 中也有具体的使用。