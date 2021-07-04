---
layout: post
title:  浅谈Android消息机制
date:   2020-07-29 18:21:31 +0800
image:  https://image.wsdydeni.top/mael-balland-dOVo51t6ie4-unsplash.jpg
tags:   Android
---

## 概览

Android 消息机制用于多线程通信

### 进程与线程

既然要讨论的是多线程通信，那么就得知道什么是线程。

说到线程，必须得了解进程是什么。

> 进程是操作系统对一个正在运行的程序的一种抽象。
>
> 进程运行时所需的所有状态信息称之为上下文。
>
> 进程可以提供俩个关键抽象：独立逻辑控制流，私有地址空间
>
> 摘录自深入理解计算机系统(第三版)

线程是运行在进程的上下文中的执行单元，共享全局数据。在现在的大多数主流操作系统中，一个进程由多个线程组成，不可避免的要面对一个问题，怎么运行调度这些线程？这也是文章讨论的主题，Android 世界中的消息机制。

### ThreadLocal

ThreadLocal 用于线程内部的数据存储，学习消息机制之前先了解一下这个前置知识。

{% highlight java %}
public class ThreadLocal<T> {
    
	static class ThreadLocalMap {
        
        private ThreadLocal.ThreadLocalMap.Entry[] table;
        
        ThreadLocalMap(ThreadLocal<?> var1, Object var2) {}
        
        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;

            Entry(ThreadLocal<?> var1, Object var2) {
                super(var1);
                this.value = var2;
            }
        }
    }
}

public class Thread implements Runnable {
    ThreadLocalMap threadLocals = null;
}
{% endhighlight %}

通过上面的源码，可以看到 Thread 持有 ThreadLocalMap 实例，ThreadLocalMap 是 ThreadLocal 的静态内部类，ThreadLocalMap 中维护着私有变量 Entry 数组，Entry  属于 key-value 的形式，key 为 ThreadLocal 的弱引用。

{% highlight java %}
public class ThreadLocal<T> {

	public void set(T var1) {
        Thread var2 = Thread.currentThread();
        ThreadLocal.ThreadLocalMap var3 = this.getMap(var2);
        if (var3 != null) {
            var3.set(this, var1);
        } else {
            this.createMap(var2, var1);
        }

    }
    
    void createMap(Thread var1, T var2) {
        var1.threadLocals = new ThreadLocal.ThreadLocalMap(this, var2);
    }
    
    public T get() {
        Thread var1 = Thread.currentThread();
        ThreadLocal.ThreadLocalMap var2 = this.getMap(var1);
        if (var2 != null) {
            ThreadLocal.ThreadLocalMap.Entry var3 = var2.getEntry(this);
            if (var3 != null) {
                Object var4 = var3.value;
                return var4;
            }
        }

        return this.setInitialValue();
    }

    private T setInitialValue() {
        Object var1 = this.initialValue();
        Thread var2 = Thread.currentThread();
        ThreadLocal.ThreadLocalMap var3 = this.getMap(var2);
        if (var3 != null) {
            var3.set(this, var1);
        } else {
            this.createMap(var2, var1);
        }

        return var1;
    }
    
    protected T initialValue() {
        return null;
    }
}
{% endhighlight %}

ThreadLocal 的 set 方法，首先获取当前线程，再获取线程的 ThreadLocalMap，对其进行操作。get 方法也是获取线程 ThreadLocalMap 中对应的值，如果没有初始化，那么会返回 initialValue  方法的值，默认为 null。

那么可以得出的结论是，不同的线程访问同一个 ThreadLocal 的数据是不相同的。

下面我们来看一个示例：

{% highlight kotlin %}
private val myBooleanTreadLocal = ThreadLocal<Boolean>()

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    myBooleanTreadLocal.set(true)
    Log.e(TAG,"MainThread myBooleanTreadLocal value : ${myBooleanTreadLocal.get()}")
    Thread({
        myBooleanTreadLocal.set(false)
        Log.e(TAG,"Thread #1 myBooleanTreadLocal value : ${myBooleanTreadLocal.get()}")
    },"Thread #1").start()

    Thread({
        Log.e(TAG,"Thread #2 myBooleanTreadLocal value : ${myBooleanTreadLocal.get()}")
    },"Thread #2").start()
}
{% endhighlight %}

在主线程中创建一个泛型为 Boolean  类型的 ThreadLocal 对象，在 onCreate() 方法中，首先在 Android 主线程将 myBooleanTreadLocal 的值设置为 true，再新建一个匿名线程，将 myBooleanTreadLocal 的值设置为 false，最后再新建一个匿名线程，不对其进行操作。

![](https://image.wsdydeni.top/ThreadLocalTest.png)

结果如上图所示，这就应证了得到的结论是正确的。

## 创建过程

众所周知，Android 那是有主线程的。

在 Activity 启动流程中，会创建 ActivityTread 对象，这就是应用的主线程。

目光看向 ActivityTread 的 main() 方法：

{% highlight java %}
public static void main(String[] args) {
	Looper.prepareMainLooper();
	Looper.loop();
}
{% endhighlight %}

其中比较关键的俩行代码，分别调用 Looper 的 prepareMainLooper() 和 Looper.loop() 方法。

so,what is Looper,look look.

### Looper

在 Looper 类源码的最上方，有这么一段注释。

> Class used to run a message loop for a thread.
>
> Threads by default donot have a message loop associated with them;
>
> to create one, call @link #prepare in the thread that is to run the loop, 
>
> and then @link #loop to have it process messages until the loop is stopped.

这是用于创建线程消息循环的类。默认情况下，线程没有与之关联的消息循环；创建一个消息循环，在线程中调用 prepare() 方法，调用 loop() 方法来开始处理消息，直到停止为止。

接下来看一下 Looper 的主要源码部分：

{% highlight java %}
public final class Looper {

	static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
	private static Looper sMainLooper;
	final MessageQueue mQueue;
    final Thread mThread;
    
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed); 
        mThread = Thread.currentThread();
    }
    
	private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) { 
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed)); 
    }
    
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
    
    public static Looper myLooper() {
        return sThreadLocal.get(); 
    }
    
}
{% endhighlight %}

首先看向 prepareMainLooper() ，调用了 prepare() 方法。在该方法中，如果当前线程的 ThreadLocal 中的 Looper 对象副本不为空，抛出 RuntimeException ，保证只会初始化一次；接着创建一个 Looper 对象，quitAllowed 标识是能否退出的意思，在 Looper 构造函数中，还初始化了本身的 MessageQueue，最后把 Looper 的变量副本添加自身的 ThreadLocal 中。

prepare() 方法调用完毕后，下方是一个同步代码块，sMainLooper 如果不为空，抛出 IllegalStateException，保证每个线程只允许执行一次，再调用 myLooper() 方法 取出 ThreadLocal 中的 Looper 副本，并设置给 sMainLooper 变量。

上文说过 Android 主线程在初始化过程中调用了 Looper 的 prepareMainLooper() 方法，那么意味着 Android 主线程已经有了当前线程的 Looper 变量的副本。

那么接下来看看 Looper 构造中 MessageQueue 初始化又是怎么一回事。

### MessageQueue 

{% highlight java %}
public final class MessageQueue {

	private final boolean mQuitAllowed;
	private long mPtr; // used by native code
	
	private native static long nativeInit();

	MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
    
}
{% endhighlight %}

在 MessageQueue  的构造函数中，主要是调用一个 native 层方法来对 Native 层 MessageQueue  进行初始化。

{% highlight java %}
 private native static void nativeDestroy(long ptr);
 private native void nativePollOnce(long ptr, int timeoutMillis);
 private native static void nativeWake(long ptr);
 private native static boolean nativeIsPolling(long ptr);
 private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
{% endhighlight %}

MessageQueue 中充斥着大量的 native 层方法，可以知道它的本质上是链接 Java 层和 C++ 层的桥梁，主要工作还是 native 层在干活，如果有兴趣了解 natice 层 的 MessageQueue  的内容，可以看一下[这篇文章](http://gityuan.com/2015/12/26/handler-message-framework/)。

------

总结一下创建过程，在 Android 主线程中创建了一个不可退出的 Looper ,里面有一个消息循环类 MessageQueue 。

Looper 源码中还有这么一段注释：

> Most interaction with a message loop is through the {@link Handler} class.

消息循环的大多数交互是通过 Handler 来进行的。

so,what is Handler,look look.

## 发送消息

Handler 类一段注释如下：

> A Handler allows you to send and process {@link Message} and Runnable objects associated with a thread's {@link MessageQueue}. Each Handler instance is associated with a single thread and that thread's message queue. When you create a new Handler it is bound to a {@link Looper}. It will deliver messages and runnables to that Looper's message queue and execute them on that Looper's thread.

大概意思就是，Handler 会和一个线程以及线程中的消息队列相关联，同时会绑定到该线程的 Looper。Handler 能够发送 Message 和 传递 Runnable 到 Looper 的消息队列中，可以在该线程上执行它们。

### Handler

Handler 部分源码如下：

{% highlight java %}
public class Handler {
	/*
     * Set this flag to true to detect anonymous, local or member classes
     * that extend this Handler class and that are not static. These kind
     * of classes can potentially create leaks.
     */
    private static final boolean FIND_POTENTIAL_LEAKS = false;
    
    final Looper mLooper;
    final MessageQueue mQueue;
    final Callback mCallback;
    final boolean mAsynchronous;
    
    public interface Callback {
        boolean handleMessage(@NonNull Message msg);
    }
    
    public Handler(@Nullable Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
}
{% endhighlight %}

首先看 Handler 的构造函数，Callback 是 Handler 内部声明的接口，async 是否需要异步发送。FIND_POTENTIAL_LEAKS 变量，源码的注释已经写的很清楚了，当该标识设置为 True 时，用于来检测这个 Handlder  是否为成员类、静态类和本地类，以防止内存泄漏的发生。接着获取当前线程的 Looper ,如果为空，抛出 RuntimeException，Hquandler必须运行在已经初始化 Looper 的线程上；再获取  Looper  中的 MessageQueue，如同注释中所说的那样，Handler 与线程的 Looper、MessageQueue 相关联。

下面写一个示例来验证：

{% highlight kotlin %}
override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        Log.e(TAG, "current looper : $mainLooper")
        Thread({
            Looper.prepare()
            val handler = Handler(Looper.myLooper()!!)
            Log.e(TAG,"current looper : ${handler.looper}" )
        },"Thread #3").start()
    }
}
{% endhighlight %}

打印结果如下图所示：

![](https://image.wsdydeni.top/LooperTest.png)

可以看到的是，Handler 是和 Looper 相关联的。

上文中提到 Handler 能够发送 Message 和 传递 Runnable

so,what is Message ,look look.

### Message 

Message 类的一段注释如下：

> Defines a message containing a description and arbitrary data object that can be sent to a {@link Handler}.

Message 是任意数据对象和描述的消息载体。

Message 的主要源码字段如下：

{% highlight java %}
public final class Message implements Parcelable {

	public int what; // 消息标识
	
	public int arg1; // 用于存储 int 类型数据
	public int arg2;
	
	public Object obj; // 任意对象数据
	
	public long when; // 消息触发时间
	
	Bundle data; // Bundle 类型数据
	Handler target; // 用于消息处理
	Runnable callback; // 回调接口
}

{% endhighlight %}

Message 构造函数为空，类似于 Builder 形式，通过设置方法来填充参数。

接下来进入正题，怎么用 Handler 发送 Message 和 传递 Runnable 到 MessageQueue 呢？

------

### 发送流程

有很多发送消息的方法，最终都会调用 MessageQueue 的 enqueueMessage() 方法。

![](https://image.wsdydeni.top/Handler%E5%8F%91%E9%80%81%E6%B6%88%E6%81%AF%E8%B0%83%E7%94%A8%E9%93%BE.png)

这里选择一条比较长的调用链来做说明。

{% highlight java %}
public class Handler {
    private static Message getPostMessage(Runnable r, Object token) {
        Message m = Message.obtain();
        m.obj = token;
        m.callback = r;
        return m;
    }
    
	public final boolean postDelayed(Runnable r, Object token, long delayMillis) {
        return sendMessageDelayed(getPostMessage(r, token), delayMillis);
    }
    
    public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    
    public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
    
    private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        msg.target = this;
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
}
{% endhighlight %}

调用 postDelayed() 方法，首先会将 Runnable 对象包装成 Message，退出消息队列时可以使用 token  调用  removeCallbacksAndMessages() 方法将这个 Message 从消息队列中移出，delayMillis 是当该 Runnable 执行时的延迟时间，单位为毫秒。随后调用 sendMessageDelayed() 方法，这个方法会做一个判断，如果延迟时间小于 0 毫秒，都做 0 毫秒处理，也就是立即执行的消息。接着调用 sendMessageAtTime 方法，如果当前 Handler 没有相关联的 MessageQueue ，那么会抛出 RuntimeException。最后调用 enqueueMessage() 方法，将 Message 的 target 字段 (上文提了 Message 属性)设置为 Handler 自身，最终调用 MessageQueue 的 enqueueMessage() 方法。

下方将目光看向 MessageQueue 的 enqueueMessage() 方法：

{% highlight java %}
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }

        synchronized (this) {
            if (msg.isInUse()) {
                throw new IllegalStateException(msg + " This message is already in use.");
            }

            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
}
{% endhighlight %}

首先要进行一些判断，如果 Message 关联的 Handler 为空或者 Message 已经使用、所在的线程正在退出，都会抛出异常。将 Message  标记为已经使用，设置 Message 触发时间为 when。接着取出当前 MessageQueue 的 头消息，如果当前 MessageQueue 为空、加入Message 是立即执行的或者小于当前头部  Message 的触发时间，那么会直接插入到消息链表的头部。不符合上述条件的 Message ，那么会对消息链表进行一个遍历操作，如果后驱结点为空或者后驱结点的触发时间大于要插入的 Message,那么就退出循环，将 Message 插入到消息链表中。

在这个过程中，还有一些需要知道的知识点。

#### 消息池

{% highlight java %}
public final class Message implements Parcelable {
    
    Message next;
    
	public static final Object sPoolSync = new Object();
    private static Message sPool;
    private static int sPoolSize = 0;

    private static final int MAX_POOL_SIZE = 50;

    private static boolean gCheckRecycle = true;
    
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
    
}
{% endhighlight %}

在上面发送消息的过程中，调用过这个 Message 的 obtain() 方法，如果 sPoolSync 不为空，取出一个 Messsge,并断开消息链表，将 flags 标记置为 0 ，没用使用过的，链表可用长度减 1，返回这个 Messsge；否则的话直接 new 一个 Message 返回。

这样的做法是减少了创建对象的开支，提高了效率。MAX_POOL_SIZE 为消息池的默认长度，通过静态成员变量 sPool 和 成员变量 next 的形式，构建了一个链表，这个链表就是消息池。

#### 时间精度

在 sendMessageAtTime() 方法中，将延时 Message 的触发事件设置为 SystemClock.uptimeMillis() + delayMillis。

{% highlight java %}
/**
* Returns milliseconds since boot, not counting time spent in deep sleep.
*
* @return milliseconds of non-sleep uptime since boot.
*/
@CriticalNative
native public static long uptimeMillis();
{% endhighlight %}

注释的意思是返回自启动以来的毫秒数，不计算深度睡眠所花费的时间。

## 处理消息

经过上文的铺垫，我们已经知道了 Looper 中维护着一个 MessageQueue ,MessageQueue  维护着一个消息链表，那么我们只需要对消息链表进行遍历，根据消息触发的时间顺序，来处理消息就可以了。

所以，回到开始提到过的 Looper.loop() 方法，去除了一些其他的代码。

#### Looper -> loop() 

{% highlight java %}
public final class Looper {
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                return;
            }
            try {
                msg.target.dispatchMessage(msg);
                
            } catch (Exception exception) {
                throw exception;
            }
            msg.recycleUnchecked();
        }
	}
}	
{% endhighlight %}

首先会取出当前线程的 Looper 变量副本，如果为空，会抛出 RuntimeException，提示没有调用 Looper.prepare() 方法进行初始化。接着取出  Looper 中的 MessageQueue，也就是消息队列，调用 next() 进行循环遍历遍历，达到条件的 Message 会被返回，调用 Message 自身相关联的 Handler 的 dispatchMessage() 来处理这个消息。

##### Message -> recycleUnchecked()

{% highlight java %}
void recycleUnchecked() {
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = UID_NONE;
    workSourceUid = UID_NONE;
    when = 0;
    target = null;
    callback = null;
    data = null;

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
{% endhighlight %}

对于不再使用的消息，消息标记为 FLAG_IN_USE ，所有变量重置，消息没有满，小于 MAX_POOL_SIZE ，那么就会加入到消息池中。

------

接着来看 MessageQueue 的 next() 方法干了什么。

####  MessageQueue  ->  next()

{% highlight java %}
public final class MessageQueue {
	Message next() {
        final long ptr = mPtr;
        if (ptr == 0) { //当消息循环已经退出，则直接返回
            return null;
        }

        int pendingIdleHandlerCount = -1; // 循环迭代的首次为-1
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
			//阻塞操作，当等待nextPollTimeoutMillis时长，或者消息队列被唤醒，都会返回
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                //当消息的Handler为空时，则查询异步消息
                if (msg != null && msg.target == null) {
                    //当查询到异步消息，则立刻退出循环
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        //当异步消息触发时间大于当前时间，则设置下一次轮询的超时时长
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                      	// 获取一条消息，并返回
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                       	//设置消息的使用状态，即flags |= FLAG_IN_USE
                        msg.markInUse();
                        return msg; //成功地获取MessageQueue中的下一条即将要执行的消息
                    }
                } else {
                    //没有消息
                    nextPollTimeoutMillis = -1;
                }

                //消息正在退出，返回null
                if (mQuitting) {
                    dispose();
                    return null;
                }

                //当消息队列为空，或者是消息队列的第一个消息时
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    //没有idle handlers 需要运行，则循环并等待。
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            //只有第一次循环时，会运行idle handlers，执行完成后，重置pendingIdleHandlerCount为0.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; //去掉handler的引用

                boolean keep = false;
                try {
                    keep = idler.queueIdle(); //idle时执行的方法
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

             //重置idle handler个数为0，以保证不会再次重复运行
            pendingIdleHandlerCount = 0;

            //当调用一个空闲handler时，一个新message能够被分发，因此无需等待可以直接查询pending message.
            nextPollTimeoutMillis = 0;
        }
    }
}
{% endhighlight %}

这里由于涉及到 native 层，所以直接照搬大佬的讲解，其实也是对源码注释的翻译。

直接看中间的一步操作，如果消息时间小于等于当前时间，那么就会从消息队列中取出这条消息返回。这里可以想象的是，并没有严格的按照约定好的时间来严格处理消息，而是只要不在当前时间之前就可以了，这样就会有触发时间不准确的问题。

#### Handler ->  dispatchMessage()

{% highlight java %}
public class Handler {
	public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
}
{% endhighlight %}

终于到达了消息机制的终点，最后的处理消息。

首先判断如果 message 的 callback 对象不为空，就调用 handleCallback 方法处理。这个 callback 是什么呢？ 就是我们常见的 Runnable 接口, handleCallback 方法其实也就是调用 run 方法。接下来看 handler 自身的 mCallback 是否为空，如果不为空，调用 mCallback 的 handleMessage 来处理消息，其次 handleMessage 返回 true 就是不需要进行下一步处理；最后就是调用自身的 handleMessage 方法来处理。

现在下面举一个例子：

{% highlight kotlin %}
const val MESSAGE_RUNNABLE_DISPOSE = 1 // Message Runnable 处理 
const val HANDLER_CALLBACK_DISPOSE = 2 // Handler Callback 处理
const val HANDLER_ITSELF_DISPOSE = 3   // Handler handleMessage 处理

fun test() {
    val handler = TestHandler(Handler.Callback {
        return@Callback when(it.what) {
            MESSAGE_RUNNABLE_DISPOSE -> throw IllegalStateException("Something that shouldn't happen")
            HANDLER_ITSELF_DISPOSE -> false
            HANDLER_CALLBACK_DISPOSE -> {
                Log.e("TestHandler","Handler CallBack handleMessage : ${it.what}")
                true
            }
            else -> true
        }
    })
    
    handler.sendMessage(Message().apply { what = MESSAGE_RUNNABLE_DISPOSE }) // 1

    handler.sendMessage(Message.obtain(handler) { // 2
        Log.e("TestHandler","Message runnable run")
    }.apply {
        what = MESSAGE_RUNNABLE_DISPOSE
    })

    handler.sendMessage(Message().apply { what = HANDLER_CALLBACK_DISPOSE }) // 3

    handler.sendMessage(Message().apply { what = HANDLER_ITSELF_DISPOSE }) // 4
}

class TestHandler(callback: Callback) : Handler(callback) {
    override fun handleMessage(msg: Message) {
        super.handleMessage(msg)
        Log.e("TestHandler","Handler handleMessage : ${msg.what}")
    }
}
{% endhighlight %}

在这里给 Message 搞了三个消息类型，也就是 what 变量。继承重写 Handler 的 handleMessage 方法，再声明一个带有 Handler Callback 的对象来处理不同的消息。在第一处，消息类型为 `MESSAGE_RUNNABLE_DISPOSE` ，而这个 Message 并没有实现 Runnable，根据上文中处理消息的逻辑，Message 自身的 callback 为空时，会判断 handler 对象是有 mCallback 对象，这里传入了一个 Callback，如果消息类型为 `MESSAGE_RUNNABLE_DISPOSE` 会抛出一个异常，因为这不是我们期望的结果，它应该被自身实现的 Runnable 接口所消费了；第二处，这里通过 Message 的 obtain 方法为这个 Message 实现了 Runnable 接口，同时消息的类型为 `MESSAGE_RUNNABLE_DISPOSE`,这里会预期的打印出 `"Message runnable run"`；第三处，发送了一个类型为 `HANDLER_CALLBACK_DISPOSE` 的 Message，此时 Message 没有实现 Runnable 接口，所以会预期的打印出 `Handler CallBack handleMessage : 2`,同时返回 true，不希望进行下一步处理，那么这里暂时改成 false 呢？那么还会打印出`"Handler handleMessage : 3"`；第四处也是一样的。现在，消息处理的优先级就不难理解了。

------

到这里，消息机制就差不多了，芜湖~

## 常见问题

子线程中调用 Looper.prepare()、Looper.loop() 后可以使用 Toast

{% highlight java %}
/**
 * Make a standard toast to display using the specified looper.
 * If looper is null, Looper.myLooper() is used.
 * @hide
 */
public static Toast makeText(@NonNull Context context, @Nullable Looper looper,
        @NonNull CharSequence text, @Duration int duration) {
    Toast result = new Toast(context, looper);

    LayoutInflater inflate = (LayoutInflater)
            context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
    TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
    tv.setText(text);

    result.mNextView = v;
    result.mDuration = duration;

    return result;
}
{% endhighlight %}

源码的注释说的非常清楚，默认会获取当前线程 TSL 中的 Looper。

所以只要在子线程中初始化 Looper ，就可以使用 Toast。

## 感谢

[Android消息机制1-Handler(Java层)](http://gityuan.com/2015/12/26/handler-message-framework/)

[Android消息机制2-Handler(Native层)](http://gityuan.com/2015/12/27/handler-message-native/)

