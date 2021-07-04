---
layout: post
title:  RecyclerView正确使用姿势
date:   2021-03-02 05:27:36 +0800
image:  https://image.wsdydeni.top/nathaniel-yeo-1paTYd1MrHI-unsplash.jpg
tags:   Android
---


{% highlight markdown %}
> 原文地址： [Nested recycler in Android done right!](https://medium.com/nerd-for-tech/nested-recycler-in-android-done-right-b101744e2a9a)
>
> 原文作者：[Jakub Minarik](https://jakub-minarik.medium.com/)
>
> 憨憨翻译：[无伤大雅的你呀](https://www.wsdydeni.top/about/)
{% endhighlight %}

这篇文章主要讨论解决了俩个问题，外层 RecyclerView 垂直滚动时嵌套的横向 RecyclerView 滑动位置的丢失以及水平方向滑动与垂直滑动的冲突解决。

So,Here we go !!!

### 构建一个示例程序

中间省略一大堆翻译，就是构建了俩个适配器，直奔实现效果(直接搬原作者图了)

![](https://image.wsdydeni.top/ConcatAdapter.png)

在垂直滑动的 RecyclerView 嵌套了几个横向滑动的 RecyclerView,通过最新推出的 ConcatAdapter 加以实现。需要了解 ConcatAdapter 更多细节的可以看一下这篇文章 [给 Adapter 做 “加法” —— 实战 MergeAdapter](https://juejin.cn/post/6844904117479931912) ,该文推出时为测试版，正式版已经更名为 ConcatAdapter ，不影响理解和使用。

{% highlight kotlin %}
class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding
    private lateinit var concatAdapter: ConcatAdapter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        val view = binding.root
        setContentView(view)
        initViews()
    }

    private fun initViews() {
        //create a populated list of sections
        val sections = DataSource.createSections(numberOfSections = 50, itemsPerSection = 25)

        //create an instance of ConcatAdapter
        concatAdapter = ConcatAdapter()

        //create AnimalSectionAdapter for the sections and add to ConcatAdapter
        val sectionAdapter = AnimalSectionAdapter(sections)
        concatAdapter.addAdapter(sectionAdapter)

        //setup the recycler
        val linearLayoutManager = LinearLayoutManager(this, RecyclerView.VERTICAL, false)
        binding.recyclerView.run {
            layoutManager = linearLayoutManager
            adapter = concatAdapter
        }
    }
}
{% endhighlight %}

可以简单看一下构建的过程，没什么好说的。

### 出现的问题

#### 滑动位置的丢失

![](https://image.wsdydeni.top/ConcatAdapter%E5%AE%9E%E4%BE%8B.gif)

通过上图可以直观的看到滑动过的横向 RecyclerView 在重新显示的时候滑动位置的丢失。为了解决这个问题，分为俩步走。在回收的时候保存滑动的位置，以及重新绑定时恢复滑动的位置。

{% highlight kotlin %}
class AnimalSectionAdapter(
//...
) {
	private val scrollStates: MutableMap<String, Parcelable?> = mutableMapOf()

	private fun getSectionID(position: Int): String {
        	return items[position].id
    	}

	override fun onViewRecycled(holder: BaseViewHolder<AnimalSection>) {
        	super.onViewRecycled(holder)

        	//save horizontal scroll state
        	val key = getSectionID(holder.layoutPosition)
        	scrollStates[key] =
           		holder.itemView.findViewById<RecyclerView>(R.id.titledSectionRecycler).layoutManager?.onSaveInstanceState()
    	}

    	override fun onBindViewHolder(
   	 //...
   	 ) {
		 //restore horizontal scroll state
        	val key = getSectionID(viewHolder.layoutPosition)
       		val state = scrollStates[key]
		if (state != null) {
 			titledSectionRecycler.layoutManager?.onRestoreInstanceState(state)
		} else {
    			titledSectionRecycler.layoutManager?.scrollToPosition(0)
		}
     }
      //...
}
{% endhighlight %}

在 Adapter 中创建一个 MutableMap 来持久化状态，重写 onViewRecycled 方法来保存对应的数据，key 为位置，value 为 layoutManager 调用 onSaveInstanceState() 方法后生成的序列化对象。在 onBindViewHolder 方法中，绑定时对位置进行判断，如果 state 不为空，就恢复位置，否则滑动到最前面的位置。

#### 水平方向与垂直方向滑动的冲突解决

原作者直接使用了他人的解决方案，可以到原文中查看。定义了一个 RecyclerView 的扩展函数，添加了触摸和滑动的处理，从而解决了这个问题。

{% highlight kotlin %}
fun RecyclerView.enforceSingleScrollDirection() {
    val enforcer = SingleScrollDirectionEnforcer()
    addOnItemTouchListener(enforcer)
    addOnScrollListener(enforcer)
}
{% endhighlight %}

------

### 其他的解决方案

#### 回收池

当外层的 RecyclerView 垂直滚动时，嵌套的横向 RecyclerView 会将视图重新加载一边，这是因为每个嵌套的 RecyclerView 拥有各自的 View Pool。可以通过给相同视图类型的 RecyclerView 设置一个共享的 View Pool,这样可以减少 View 的创建，提高了垂直方向滚动的性能。

在这个示例项目中，显然这样做是可以的。

{% highlight kotlin %}
class AnimalSectionAdapter(
//...
) {
    //create an instance of ViewPool
    private val viewPool = RecyclerView.RecycledViewPool()

    //and set it to each nested recycler when binding
    override fun onBindViewHolder(
    //...
    ) {
      //...
        titledSectionRecycler?.run {
            //right here
            this.setRecycledViewPool(viewPool)
            this.layoutManager = layoutManager
            this.adapter = AnimalAdapter(item.animals)
        }
      //...
    }
}
{% endhighlight %}

#### 设置预加载的数量

可以通过嵌套的 RecyclerView 的 LinearLayoutManager ，调用 setInitialPrefetechItemCount() 方法来预设可能会显示的可见数量。在垂直滑动的时候，外层 RecyclerView 会要求内层 RecyclerView 进行预绑定，但是内层 RecyclerView 并不知道应该预加载多少个 Item，直到该内层 RecyclerView 可见的时候，其他的非预加载 Item 才会被加载(默认情况下只有俩个 Item 会被预加载)，这样会导致性能问题。

可以通过下面的方式来解决这个问题：

{% highlight kotlin %}
class AnimalSectionAdapter(
//...
) {
    override fun onBindViewHolder(
    //... 
    ) {
      //...
      val layoutManager = LinearLayoutManager(itemView.context, LinearLayoutManager.HORIZONTAL, false)

      //right here
      layoutManager.initialPrefetchItemCount = 4 //estimated number of visible items
      
      val titledSectionRecycler = itemView.findViewById<RecyclerView>(R.id.titledSectionRecycler)
      titledSectionRecycler?.run {
            this.layoutManager = layoutManager
            //...
      }
      //...
    }
}
{% endhighlight %}