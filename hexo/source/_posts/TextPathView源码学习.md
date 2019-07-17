---
title: TextPathView源码学习
date: 2018-03-09 17:08:29
tags: 源码
categories: 源码周读
---

# TextPathView源码学习
## PathMeasure
PathMeasure是Path的坐标计算器，可以获取Path上每一个构成点的坐标。  
初始化：
```java
PathMeasure pathMeasure = new PathMeasure();
pathMeasure.setPath(path, true);
// 或者使用有参构造法
PathMeasure(Path path, boolean forceClosed)
```
forceClosed，简单的说，就是Path最终是否需要闭合，如果为True的话，则不管关联的Path是否是闭合的，都会被闭合。对绑定的Path没有影响，但是对PathMeasure的测量结果有影响，会包含最后一段闭合的路劲。

重要api：
- getSegment(start, stop, dst, move):将start到stop之间的path添加到dst中，可以重复添加，不会被覆盖
- nextContour():如果有多个contour的时候，跳入到下一个contour

## 核心代码
1.通过Paint.getTextPath获取文字path：
```java
   protected void initTextPath() {
        mDst.reset();
        mFontPath.reset();
        mTextPaint.getTextPath(mText, 0, mText.length(), 0,mTextPaint.getTextSize(), mFontPath);
        // pathMeasure绑定path，就可以进行测量
        mPathMeasure.setPath(mFontPath, false);
        mLengthSum = mPathMeasure.getLength();
        //获取所有路径的总长度
        while (mPathMeasure.nextContour()) {
            mLengthSum += mPathMeasure.getLength();
        }
    }
```

2.通过动画进度来控制绘制
```java
mAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    @Override
    public void onAnimationUpdate(ValueAnimator valueAnimator) {
        mAnimatorValue = (float) valueAnimator.getAnimatedValue();
        drawPath(mAnimatorValue);
    }
});

void startAnimation(){
  initTextPath()  // 获取到Text的path,放在mDst中
  mAnimation.start()
}

// 绘制出已计算出的
void View.onDraw(){
  canvas.drawPath(mDst, mDrawPaint);  // 这里的mDst由drawPath随时更新
}

// 根据动画进行比例计算出需要绘制的path
public void drawPath(float progress) {
        mAnimatorValue = progress;
        mStop = mLengthSum * progress;

        //重置路径,目的是从开头的字开始重新测量。
        mPathMeasure.setPath(mFontPath, false);
        mDst.reset();
        mPaintPath.reset();

        //根据进度获取路径
        while (mStop > mPathMeasure.getLength()) {
            mStop = mStop - mPathMeasure.getLength();
            mPathMeasure.getSegment(0, mPathMeasure.getLength(), mDst, true); // 这里计算出来的Path会和后面的0-stop的path相加
            if (!mPathMeasure.nextContour()) {
                break;
            }
        }
        mPathMeasure.getSegment(0, mStop, mDst, true);

        //绘画画笔效果
        if (showPainterActually) {
            mPathMeasure.getPosTan(mStop, mCurPos, null);
            drawPaintPath(mCurPos[0], mCurPos[1], mPaintPath);
        }

        //绘画路径
        postInvalidate();
}

```

从上面可以看到，只要我们修改一下，把while循环的起点改为每一个contour，分别绘制对应的比例就行。
```java
while (mPathMeasure.nextContour()) {
    mLength = mPathMeasure.getLength();
    mStop = mLength * mAnimatorValue;
    mPathMeasure.getSegment(0, mStop, mDst, true);

    //绘画画笔效果
    if (showPainterActually) {
        mPathMeasure.getPosTan(mStop, mCurPos, null);
        drawPaintPath(mCurPos[0],mCurPos[1],mPaintPath);
    }
}
```


## 启示
onDraw里面是不能进行耗时操作的，所以在动画进行的回调函数中去进行progress的判断，然后更新onDraw里面要用到的数值。
































。
