---
layout: post
title:  Android自定义View之滑块验证码
date:   2020-07-19 22:01:03 +0800
image:  https://image.wsdydeni.top/orange.jpg
tags:   Android
---

啥也不说了 直接就是给上[项目地址](https://github.com/wsdydeni/CustomViewCollection) 有兴趣的朋友直接拖源码吧

![](https://image.wsdydeni.top/%E6%BB%91%E5%9D%97%E9%AA%8C%E8%AF%81%E7%A0%81%E6%95%88%E6%9E%9C%E5%9B%BE.gif)

滑块验证码相信各位看官老爷一定使用过 所以决定上手做一个

## 分析

**SlidePuzzle**分成了俩块 上方的`Puzzle`以及下方的`SlideBar`

首先把目光投向`SlideBar` emmmmm 这玩意好像没啥好讲的

再将目光看向`Puzzle` 发现还是没啥好讲的 滑块以及背景

## 绘制

### SlideBar拖动条

*SlideBar的Draw过程*

{% highlight kotlin %}
override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)
    mBitmapHeight = height / 5 * 3
    mBitmapWidth = width / 8
    mBitmap = BitmapUtils.getNewBitmap(mBitmap,mBitmapWidth,mBitmapHeight)
    backgroundRect.set(0f, height.toFloat() / 5 * 2, width.toFloat(), height.toFloat() / 5 * 3)
    //绘制拖动条背景
    canvas.drawRoundRect(backgroundRect,PixelUtils.dp2pxF(context,5f),PixelUtils.dp2pxF(context,5f),mPaint)
    if(mDistance >= width - mBitmapWidth ) mDistance = width - mBitmapWidth
    if(mDistance <= 0) mDistance = 0
    //绘制拖动图片
    canvas.drawBitmap(mBitmap, mDistance.toFloat(), ((height - mBitmapHeight) / 2).toFloat(),mPaint)
}
{% endhighlight %}

***是不是很朴实无华呢 是的***

按照`View`的宽高根据比例得到新的`Bitmap`

绘制一个具有小圆角的长方形`Rect` 作为背景

最后绘制一个用来拖动的图片

### 滑块Puzzle

*Puzzle的onDraw过程*

{% highlight kotlin %}
override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)
    if(isShowAnim) {
        canvas.save()
        //绘制完整背景Bitmap
        canvas.drawBitmap(mBitmap, 0f,0f,null)
        //绘制成功动画
        canvas.drawRect(rect,mSuccessPaint)
        canvas.restore()
    }else {
        canvas.save()
        //绘制背景Bitmap
        mBackgroundBitmap?.let { canvas.drawBitmap(it,0f,0f,null) }
        canvas.restore()
        canvas.save()
        //绘制滑块
        mPuzzleBitmap?.let { canvas.drawBitmap(it, -(randomX * mWidth) + mProgress * 0.9f * width, 0f, null) }
        canvas.restore()
    }
}
{% endhighlight %}

这里设置了一个参数`isShowAnim`用来控制是否显示验证成功动画

如果需要显示 则显示完整的`Bitmap`以及绘制闪光特效

如果不需要显示 则绘制带有缺口的背景`Bitmap` 以及 滑块

首先 需要构建一个随机位置形状的缺口 也就是`Path`路径

{% highlight kotlin %}
private fun createPath() {
    randomX = RandomUtils.randomFloat(0.5f,0.8f)
    randomY = RandomUtils.randomFloat(0.1f,0.3f)
    val gapArray = randomGap()
    val sideLength = if(0.1f * mWidth >= 0.1f * mHeight) { 0.1f * mWidth }else { 0.1f * mHeight }
    path.moveTo(randomX * mWidth, randomY * mHeight)
    path.lineTo(randomX * mWidth + 0.2f * sideLength, randomY * mHeight)
    if(1 in gapArray) {
        path.arcTo(RectF(randomX * mWidth + 0.2f * sideLength, randomY * mHeight - 0.2f * sideLength,
        randomX * mWidth + 0.6f * sideLength, randomY * mHeight + 0.2f * sideLength),180f,randomDirection())
    }
    path.lineTo(randomX * mWidth + sideLength, randomY * mHeight)
    path.lineTo(randomX * mWidth + sideLength, randomY * mHeight + 0.2f * sideLength)
    if(2 in gapArray) {
        path.arcTo(RectF(randomX * mWidth + 0.8f * sideLength, randomY * mHeight + 0.2f * sideLength,
        randomX * mWidth + 1.2f * sideLength, randomY * mHeight + 0.6f * sideLength),270f,randomDirection())
    }
    path.lineTo(randomX * mWidth + sideLength, randomY * mHeight + sideLength)
    path.lineTo(randomX * mWidth + 0.6f * sideLength, randomY * mHeight + sideLength)
    if(3 in gapArray){
        path.arcTo(RectF(randomX * mWidth + 0.2f * sideLength, randomY * mHeight + 0.8f * sideLength,
        randomX * mWidth + 0.6f * sideLength, randomY * mHeight + 1.2f * sideLength),0f,randomDirection())
    }
    path.lineTo(randomX * mWidth, randomY * mHeight + sideLength)
    path.lineTo(randomX * mWidth,randomY * mHeight + 0.8f * sideLength)
    if (4 in gapArray) {
        path.arcTo(RectF(randomX * mWidth - 0.2f * sideLength, randomY * mHeight + 0.4f * sideLength,
        randomX * mWidth + 0.2f * sideLength, randomY * mHeight + 0.8f * sideLength),90f,randomDirection())
    }
    path.close()
}
{% endhighlight %}

在一定区间内随机产生`randomX`以及`randomY` 相对于宽高的比例 确定缺口的左上角位置

这里呢 构建了一个正方形的Path `slidelength` 边长长度 `0.1`倍`width`或者`height`

`gapArray` 一个随机长度为2-4的数组 随机产生滑块圆弧的方向、位置以及数量

有了`Path`之后就可以开始绘制背景以及滑块了

*获取背景*

{% highlight kotlin %}
private fun getBackgroundBitmap(bitmap: Bitmap) : Bitmap {
    val newBitmap = Bitmap.createBitmap(bitmap.width,bitmap.height,Bitmap.Config.ARGB_8888)
    val canvas = Canvas(newBitmap)
    canvas.save()
    canvas.drawBitmap(bitmap,0f,0f,null)
    canvas.restore()
    canvas.save()
    canvas.drawPath(path,mShadowPaint)
    canvas.drawPath(path,mWhitePaint)
    canvas.restore()
    return newBitmap
}
{% endhighlight %}

创建一个`View`同等宽高的空白`Bitmap` 绘制传入的`bitmap`

随后绘制一个阴影区域以及阴影区域的白色描边

*获取滑块*

{% highlight kotlin %}
private fun getBlurBitmap(bitmap: Bitmap) : Bitmap {
    val targetBitmap = Bitmap.createBitmap(bitmap.width,bitmap.height,Bitmap.Config.ARGB_8888)
    val rs = RenderScript.create(context)
    val blurScript = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs))
    val allIn = Allocation.createFromBitmap(rs,bitmap)
    val allOut = Allocation.createFromBitmap(rs,targetBitmap)
    blurScript.setRadius(10f)
    blurScript.setInput(allIn)
    blurScript.forEach(allOut)
    allOut.copyTo(targetBitmap)
    rs.destroy()
    return targetBitmap
}

private fun getPuzzleBitmap(bitmap: Bitmap) : Bitmap {
    val newBitmap = Bitmap.createBitmap(bitmap.width,bitmap.height,Bitmap.Config.ARGB_8888)
    val canvas = Canvas(newBitmap)
    canvas.save()
    canvas.drawPath(path,mPuzzlePaint)
    mPuzzlePaint.xfermode = PorterDuffXfermode(PorterDuff.Mode.SRC_IN)
    canvas.drawBitmap(bitmap,0f,0f,mPuzzlePaint)
    mPuzzlePaint.xfermode = null
    canvas.restore()
    canvas.drawPath(path,mWhitePaint)
    return newBitmap
}
{% endhighlight %}

在`getBlurBitmap`中使用`RenderScript`对`bitmap`进行模糊处理

模糊处理过后,创建一个空白`Bitmap`

绘制一下路径 使用混合模式将目标区域的滑块截取下来 最后也加上白色描边

*如此 滑块和背景就大功告成啦*

## 拖动验证

### 滑动监听

直接重写`SlideBar`的`onTouchEvent` 拦截手势消费事件

{% highlight kotlin %}
override fun onTouchEvent(event: MotionEvent): Boolean {
    when(event.action) {
        MotionEvent.ACTION_DOWN -> {
            currentTemp = System.currentTimeMillis()
        }
        MotionEvent.ACTION_MOVE -> {
            mDistance = if(event.x.toInt() < 0) 0 else event.x.toInt()
            _onDrag?.invoke(mDistance.toFloat() / (width - mBitmapWidth),null,false)
            invalidate()
        }
        MotionEvent.ACTION_UP -> {
            userTime = (System.currentTimeMillis() - currentTemp) / 1000f
            _onDrag?.invoke(mDistance.toFloat() / (width - mBitmapWidth),userTime,true)
        }
    }
    return true
}

private var _onDrag : ((Float, Float?, Boolean) -> Unit)? = null
{% endhighlight %}

这里添加了监听`onDrag`用于同步滑动 参数分别为滑动进度 使用时间 是否需要验证

手指按下时 记录当前按下的时间

手指移动时 记录滑动距离以及当前滑动的比例 重绘`SlideBar`以及`Puzzle`

手指松开时 计算滑动时间 验证是否成功

### 验证回调

{% highlight kotlin %}
mSlideBar.setOnDragListener { progress, useTime, verify ->
    //同步滑块位置
    mPuzzle.setProgress(progress)
    //停止滑动时验证
    if(verify) { verify(abs(progress * 0.9f - mPuzzle.getCurRandomX()) < 0.018f,useTime) }
}

private fun verify(isSuccess : Boolean,useTime : Float?) {
    mTipText.text = if(isSuccess && useTime != null) {
        String.format("拼图成功: 耗时%.1f秒,打败了%d%%的用户!",useTime,(99 - ((if (useTime > 1f) useTime - 1f else 0f) / 0.1f)).toInt())
    }else {
        context.resources.getText(R.string.failure_text)
    }
    mTipText.visibility = View.VISIBLE
    ValueAnimator.ofFloat(0.8f,0.72f).apply {
        addUpdateListener { mTipText.translationY = it.animatedValue as Float }
        duration = 1500
        start()
        doOnEnd { mTipText.visibility = View.GONE }
    }
    if(isSuccess) {
        mPuzzle.showSuccessAnim()
        onVerify?.invoke(true)
    } else {
        mSlideBar.reset()
        onVerify?.invoke(false)
    }
}

private var onVerify : ((Boolean) -> Unit)? = null
{% endhighlight %}

如果拖动距离比例与滑块缺口比例在`0.018f`以内 判断验证成功 否则 验证失败

提供了`onVerify` 对验证结果的回调

------

#### 验证失败

弹出验证失败的文字以及重置滑块以及拖动条位置

使用`ValueAnimator`平滑的重置拖动距离 避免过于生硬

{% highlight kotlin %}
fun reset() {
    //重置拖动位置
    distanceAnimator = ValueAnimator.ofInt(mDistance,0).apply {
        duration = 1000
        addUpdateListener {
            mDistance = it.animatedValue as Int
            invalidate()
        }
        start()
    }
    //重置滑块位置
    progressAnimator = ValueAnimator.ofFloat(mDistance.toFloat() / (width - mBitmapWidth),0f).apply {
        duration = 1000
        addUpdateListener { _onDrag?.invoke(it.animatedValue as Float,null,false) }
        start()
    }
}
{% endhighlight %}

#### 验证成功

弹出验证成功以及使用时间的文字提示以及显示成功动画

成功动画就是一道闪光 通过`LinearGradient`来实现

{% highlight kotlin %}
override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
    super.onSizeChanged(w, h, oldw, oldh)
    rect.set(0,0,w,h)
    linearGradient = LinearGradient(mWidth.toFloat() / 2.toFloat(),0f,0f,mHeight.toFloat(),
        intArrayOf(Color.BLACK,Color.WHITE,Color.BLACK),null,Shader.TileMode.CLAMP)
    mSuccessPaint.shader = linearGradient
    mSuccessPaint.xfermode = PorterDuffXfermode(PorterDuff.Mode.LIGHTEN)
    mGradientMatrix.setTranslate(- 2f * mWidth.toFloat(),mHeight.toFloat())
    linearGradient.setLocalMatrix(mGradientMatrix)
}

private fun initAnimator() {
    valueAnimator = ValueAnimator.ofFloat(0f,1f).apply {
        duration = 1000
        addUpdateListener {
            val progress = it.animatedValue as Float
            mTranslateX = 4 * mWidth * progress - 2 * mWidth
            mTranslateY = mHeight * progress
            mGradientMatrix.setTranslate(mTranslateX,mTranslateY)
            linearGradient.setLocalMatrix(mGradientMatrix)
            invalidate()
        }
    }
}
{% endhighlight %}

绘制一个`View`等宽高的`Rect` 动画的区域

`LinearGradient` 前俩个参数 动画的倾斜角度

`intArrayOf(Color.BLACK,Color.WHITE,Color.BLACK)` 动画的颜色范围

`ValueAnimator`对`linearGradient`当前范围依次增加 重绘`Puzzle`实现闪光划过效果

------

***大功告成啦 哈哈哈哈哈哈哈哈哈哈哈哈***

***磕磕碰碰 总算是勉强完成了***

期间碰到了很多的问题 摸索着写过来 学习了很多

以后会在这个库中慢慢更新更多的自定义View

*最后一句话 送给自己以及各位看官朋友 与诸君共勉*

***路漫漫其修远兮 吾将上下而求索***