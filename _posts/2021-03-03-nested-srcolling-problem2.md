---
layout: post
title:   RecyclerView嵌套反向滑动问题
date:   2021-03-03 05:24:53 +0800
image:  https://image.wsdydeni.top/wl001002631-J8TT8zZR2PM-unsplash.jpg
tags:   Android
---

> - 原文地址：[Fixing RecyclerView nested scrolling in opposite direction
>   and making ViewPager2 usable](https://bladecoder.medium.com/fixing-recyclerview-nested-scrolling-in-opposite-direction-f587be5c1a04)
> - 原文作者：[Christophe Beyls](https://bladecoder.medium.com/)
> - 憨憨翻译：[无伤大雅的你呀](https://www.wsdydeni.top/)

------

众所周知，RecyclerView 可以通过设置 LayoutManager 的方式来选择滑动的方向，RecyclerView 还实现了 NestedScrollingChild3 接口来支持嵌套滚动，NestedScroll 机制可以让内层滚动子项拦截滚动事件，并与外层嵌套父项启动嵌套滚动。CoordinatorLayout 就是比较典型的例子，内部的 Behavior 可以监听实现了 NestedScrollingChild 接口的子 View 的滑动状态(用于实现嵌套滚动)，当然，它的功能不仅仅只有这个，还可以监听位置和尺寸的变化(折叠布局和动画效果)，这里就不做过多讨论了。

前不久，最新推出了 ViewPager2，不过它和 ViewPager 不同的是，内部是基于 RecyclerView 实现的，这样就会导致出现一些问题，问题的起因是 RecyclerView 并未实现 NestedScrollingParent3 接口，因此不支持内部嵌套可滚动(相同方向)的子项。不过官方提供了一个解决方案，用来支持 ViewPager2 中实现嵌套滚动。

{% highlight kotlin %}
class NestedScrollableHost : FrameLayout {
    constructor(context: Context) : super(context)
    constructor(context: Context, attrs: AttributeSet?) : super(context, attrs)

    private var touchSlop = 0
    private var initialX = 0f
    private var initialY = 0f
    private val parentViewPager: ViewPager2?
        get() {
            var v: View? = parent as? View
            while (v != null && v !is ViewPager2) {
                v = v.parent as? View
            }
            return v as? ViewPager2
        }

    private val child: View? get() = if (childCount > 0) getChildAt(0) else null

    init {
        touchSlop = ViewConfiguration.get(context).scaledTouchSlop
    }

    private fun canChildScroll(orientation: Int, delta: Float): Boolean {
        val direction = -delta.sign.toInt()
        return when (orientation) {
            0 -> child?.canScrollHorizontally(direction) ?: false
            1 -> child?.canScrollVertically(direction) ?: false
            else -> throw IllegalArgumentException()
        }
    }

    override fun onInterceptTouchEvent(e: MotionEvent): Boolean {
        handleInterceptTouchEvent(e)
        return super.onInterceptTouchEvent(e)
    }

    private fun handleInterceptTouchEvent(e: MotionEvent) {
        val orientation = parentViewPager?.orientation ?: return

        // Early return if child can't scroll in same direction as parent
        if (!canChildScroll(orientation, -1f) && !canChildScroll(orientation, 1f)) {
            return
        }

        if (e.action == MotionEvent.ACTION_DOWN) {
            initialX = e.x
            initialY = e.y
            parent.requestDisallowInterceptTouchEvent(true)
        } else if (e.action == MotionEvent.ACTION_MOVE) {
            val dx = e.x - initialX
            val dy = e.y - initialY
            val isVpHorizontal = orientation == ORIENTATION_HORIZONTAL

            // assuming ViewPager2 touch-slop is 2x touch-slop of child
            val scaledDx = dx.absoluteValue * if (isVpHorizontal) .5f else 1f
            val scaledDy = dy.absoluteValue * if (isVpHorizontal) 1f else .5f

            if (scaledDx > touchSlop || scaledDy > touchSlop) {
                if (isVpHorizontal == (scaledDy > scaledDx)) {
                    // Gesture is perpendicular, allow all parents to intercept
                    parent.requestDisallowInterceptTouchEvent(false)
                } else {
                    // Gesture is parallel, query child if movement in that direction is possible
                    if (canChildScroll(orientation, if (isVpHorizontal) dx else dy)) {
                        // Child can scroll, disallow all parents to intercept
                        parent.requestDisallowInterceptTouchEvent(true)
                    } else {
                        // Child cannot scroll, allow all parents to intercept
                        parent.requestDisallowInterceptTouchEvent(false)
                    }
                }
            }
        }
    }
}
{% endhighlight %}

上面的思路总结为，如果内部有可以水平方向滑动的子视图，那么在执行 onInterceptTouchEvent() 方法之前调用 requestDisallowInterceptTouchEvent() 来让外层不要拦截这个事件。这是翻译文章，不多扯事件机制了。

## 问题

正常的逻辑思维，嵌套反方向的滚动视图，应该是开箱即用的。垂直的滚动视图不应该拦截水平方向手势，反之亦然。不过呢，ViewPager 源码里，工程师们做了滑动冲突的处理；而 ViewPager2 只考虑了水平方向的滑动。

上面已经提了 ViewPager2 是基于 RecyclerView 实现的，那么在垂直方向的 RecyclerView 中嵌套了几个水平方向的 RecyclerView,结果可想而知。

![](https://image.wsdydeni.top/%E5%B5%8C%E5%A5%97%E6%BB%91%E5%8A%A8%E6%94%B9%E9%80%A0%E5%89%8D1.gif)

预期是水平滑动，结果是外层 RecyclerView 拦截处理了这次事件，导致的垂直滚动。为什么会产生这种情况，因为很多时候的触摸事件，并不是完全水平的手势，更多的是对角线形式的，即使是意图非常明显的水平手势(水平滑动距离大于垂直滑动距离)。用户呀，毕竟是人，不可能每次划出完美的笔直的横线。

下面是另外一种情况，横向 ViewPager2 内部嵌套垂直滚动视图 ：

![](https://image.wsdydeni.top/%E5%B5%8C%E5%A5%97%E6%BB%91%E5%8A%A8%E6%94%B9%E9%80%A0%E5%89%8D2.gif)

不用想，外部的 ViewPager2 拦截了内部 RecylcerView 的垂直滑动事件，导致 ViewPager2 横向滑动。

## 原因

> 真男人是不会后退的，你让我不用 ViewPager2,重新用 ViewPager？抱歉！这是不可能的！

问题出在哪里呢？遇事不决，量子力学！搞错了，直蹦源码。

{% highlight java %}
// RecyclerView -> onInterceptTouchEvent()

boolean canScrollHorizontally = mLayout.canScrollHorizontally();
boolean canScrollVertically = mLayout.canScrollVertically();
switch (action) {
    ...
    case MotionEvent.ACTION_MOVE: {
        ...
        final int x = (int) (e.getX(index) + 0.5f);
        final int y = (int) (e.getY(index) + 0.5f);
        if (mScrollState != SCROLL_STATE_DRAGGING) {
            final int dx = x - mInitialTouchX;
            final int dy = y - mInitialTouchY;
            boolean startScroll = false;
            if (canScrollHorizontally && Math.abs(dx) > mTouchSlop) {
                mLastTouchX = x;
                startScroll = true;
            }
            if (canScrollVertically && Math.abs(dy) > mTouchSlop) {
                mLastTouchY = y;
                startScroll = true;
            }
            if (startScroll) {
                setScrollState(SCROLL_STATE_DRAGGING);
            }
        }
    } break;
    ...
}
return mScrollState == SCROLL_STATE_DRAGGING;
{% endhighlight %}

上面提到过 RecyclerView 是有一个确定的方向的，而且有滑动手势大多数情况为对角线方向，也就是斜的(俩个方向都有)，假设现在有俩个反方向的 RecyclerView 嵌套，滑动时，只要有一小部分滑动距离(外层 RecyclerView 相同方向的)大于可滑动的最小距离，那么外层 RecyclerView 就会拦截处理这次滑动事件。

问题就在于，RecyclerView 单向滚动不会计算这个手势是否更接近不同方向的手势，再决定是否拦截它，这样就会导致嵌套滑动冲突。

原文作者提出了修改方案，对单个方向的滚动进行优化。

{% highlight java %}
if (canScrollHorizontally && Math.abs(dx) > mTouchSlop && (canScrollVertically || Math.abs(dx) > Math.abs(dy))) {
    mLastTouchX = x;
    startScroll = true;
}
if (canScrollVertically && Math.abs(dy) > mTouchSlop && (canScrollHorizontally || Math.abs(dy) > Math.abs(dx))) {
    mLastTouchY = y;
    startScroll = true;
}
{% endhighlight %}

## 解决方案

哦，我牛逼的原文作者希望上述方案，Google 工程师能够在新版源码中实现。不过我去查看了最新的版本，还是老样子。下面来讨论一下其他的可行性方案。

显而易见的可以想到直接重写 RecyclerView的 onInterceptTouchEvent() 方法不就好了，很抱歉，里面有大量的私有字段，所以必须要调用 super.onInterceptTouchEvent() 方法，并在这个语句后，添加一些其他操作，对逻辑进行调整。ViewPager2 也是 final 类，是不可以通过继承的方式修改使用的，事实证明，这个方案是不切实际的。

可以通过实现 OnItemTouchListener 和 OnScrollListener 俩个接口的方式，来实现想要的效果：

{% highlight kotlin %}
fun RecyclerView.enforceSingleScrollDirection() {
    val enforcer = SingleScrollDirectionEnforcer()
    addOnItemTouchListener(enforcer)
    addOnScrollListener(enforcer)
}

private class SingleScrollDirectionEnforcer : RecyclerView.OnScrollListener(), OnItemTouchListener {

    private var scrollState = RecyclerView.SCROLL_STATE_IDLE
    private var scrollPointerId = -1
    private var initialTouchX = 0
    private var initialTouchY = 0
    private var dx = 0
    private var dy = 0

    override fun onInterceptTouchEvent(rv: RecyclerView, e: MotionEvent): Boolean {
        when (e.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                scrollPointerId = e.getPointerId(0)
                initialTouchX = (e.x + 0.5f).toInt()
                initialTouchY = (e.y + 0.5f).toInt()
            }
            MotionEvent.ACTION_POINTER_DOWN -> {
                val actionIndex = e.actionIndex
                scrollPointerId = e.getPointerId(actionIndex)
                initialTouchX = (e.getX(actionIndex) + 0.5f).toInt()
                initialTouchY = (e.getY(actionIndex) + 0.5f).toInt()
            }
            MotionEvent.ACTION_MOVE -> {
                val index = e.findPointerIndex(scrollPointerId)
                if (index >= 0 && scrollState != RecyclerView.SCROLL_STATE_DRAGGING) {
                    val x = (e.getX(index) + 0.5f).toInt()
                    val y = (e.getY(index) + 0.5f).toInt()
                    dx = x - initialTouchX
                    dy = y - initialTouchY
                }
            }
        }
        return false
    }

    override fun onTouchEvent(rv: RecyclerView, e: MotionEvent) {}

    override fun onRequestDisallowInterceptTouchEvent(disallowIntercept: Boolean) {}

    override fun onScrollStateChanged(recyclerView: RecyclerView, newState: Int) {
        val oldState = scrollState
        scrollState = newState
        if (oldState == RecyclerView.SCROLL_STATE_IDLE && newState == RecyclerView.SCROLL_STATE_DRAGGING) {
            recyclerView.layoutManager?.let { layoutManager ->
                val canScrollHorizontally = layoutManager.canScrollHorizontally()
                val canScrollVertically = layoutManager.canScrollVertically()
                if (canScrollHorizontally != canScrollVertically) {
                    if ((canScrollHorizontally && abs(dy) > abs(dx))
                            || (canScrollVertically && abs(dx) > abs(dy))) {
                        recyclerView.stopScroll()
                    }
                }
            }
        }
    }
}
{% endhighlight %}

重写 OnItemTouchListener 的 onInterceptTouchEvent 方法，逻辑不变，返回 false 即可，目的是不要拦截事件；重写 OnScrollListener 的 onScrollStateChanged 方法，在静止状态向拖动状态转变时，如果与 RecyclerView 反向的滑动距离大于同方向的距离，停止滑动。

对于 ViewPager2 来说，先获取内部的 RecyclerView，再采用上述方案即可。

{% highlight kotlin %}
val ViewPager2.recyclerView: RecyclerView
    get() {
        return this[0] as RecyclerView
    }
{% endhighlight %}

{% highlight kotlin %}
val pager: ViewPager2 = findViewById(R.id.pager)
pager.recyclerView.enforceSingleScrollDirection()
{% endhighlight %}

到这里，问题就解决完毕了。

![](https://image.wsdydeni.top/%E5%B5%8C%E5%A5%97%E6%BB%91%E5%8A%A8%E6%94%B9%E9%80%A0%E5%90%8E1.gif)

![](https://image.wsdydeni.top/%E5%B5%8C%E5%A5%97%E6%BB%91%E5%8A%A8%E6%94%B9%E9%80%A0%E5%90%8E2.gif)

可以看到的是，现在的滑动行为和预期是一致的。

## 总结

嵌套滑动问题是不可避免的，这里提供了一个临时可解决方案。

感谢各位的阅读，欢迎交流学习。