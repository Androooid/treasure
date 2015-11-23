##前言
【环境】源码分析使用的版本是 4.4.2_r1。  
【假设】读者已经了解Handler，Looper，Message，MessageQueue的关系以及各自基本的功能。  
【注意】请读者自备ThreadLocal技能。

##Handler片段——Handler创建
日常开发最常用的是主线程的Handler，可以用于定时任务调度以及异步线程调度任务到主线程。通常我们会创建一个Handler的实例，之后使用sendMessage发送消息，下面看一下这个过程发生了什么事情，其构造方法如下:

```java
public Handler() {
	this(null, false);
}

public Handler(Callback callback, boolean async) {
	if (FIND_POTENTIAL_LEAKS) {
		final Class<? extends Handler> klass = getClass();
		if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
				(klass.getModifiers() & Modifier.STATIC) == 0) {
			Log.w(TAG, "The following Handler class should be static or leaks might occur: " + klass.getCanonicalName());
		}
	}

	mLooper = Looper.myLooper();
	if (mLooper == null) {
		throw new RuntimeException("Can't create handler inside thread that has not called Looper.prepare()");
	}
	mQueue = mLooper.mQueue;
	mCallback = callback;
	mAsynchronous = async;
}
```
这里重点是 __mLooper = Looper.myLooper();__ 以及后面三行代码。我们看到，新建一个Handler的时候，如果Looper.myLooper为null，则会抛出异常；如果不为null，则获取Looper中的mQueue。其余两个参数不是重点，暂不分析。

##Looper片段——Looper创建
Android中提供了相关步骤将一个普通Thread包装成一个和主线程具有相同功能的线程：具备消息Looper机制，示例代码可以在Looper源码中找到:

```java
class LooperThread extends Thread {
     public Handler mHandler;

     public void run() {
          Looper.prepare();

          mHandler = new Handler() {
             public void handleMessage(Message msg) {
                 // process incoming messages here
              }
         };
         Looper.loop();
     }
}
```

就从这里入手，分析这个线程如何具备Looper能力的。首先我们从调用的 __Looper.prepare();__ 方法入手，其源码如下:

```java
/** Initialize the current thread as a looper.
 * This gives you a chance to create handlers that then reference
 * this looper, before actually starting the loop. Be sure to call
 * {@link #loop()} after calling this method, and end it by calling
 * {@link #quit()}.
 */
public static void prepare() {
	prepare(true);
}

private static void prepare(boolean quitAllowed) {
	if (sThreadLocal.get() != null) {
		throw new RuntimeException("Only one Looper may be created per thread");
	}
	sThreadLocal.set(new Looper(quitAllowed));
}
```
我们关注一下异常，异常消息的含义是：每一个线程只能有一个Looper。而判断的条件是，从sThreadLocal中获取的对象不为null。
//TODO
如果之前没有创建Looper，则会实例化一个Looper，看一下构造方法:

```java
private Looper(boolean quitAllowed) {
	mQueue = new MessageQueue(quitAllowed);
	mThread = Thread.currentThread();
}
```
构造函数分为两步：1）创建了一个Message；2）获取当前的线程。注意mQueue这个变量，就是上一节中Handler从Looper中获取的队列对象。

##Handler和Looper的联系
在消息机制里面，Looper只是负责管理消息队列，也就是取出消息进行处理，而Handler则是负责发送消息以及处理消息的，那么Handler和Looper又是如何绑定到一起的呢？从Handler的源码看，关键代码应该是:  __mLooper = Looper.myLooper();__ ，源码如下:

```java
/**
 * null if the calling thread is not associated with a Looper.
 */
public static Looper myLooper() {
	return sThreadLocal.get();
}
```
这里把注释也Copy过来了，myLooper方法是直接从sThreadLocal中读取的变量，注释说到：myLooper()方法在当前调用线程没有绑定到一个Looper上的时候会返回null。搜索整个Looper类，会发现只有前面提到的prepare()方法中有sThreadLocal.set()调用。即在为一个线程实例化Handler之前，必须在该线程中调用Looper.prepare()方法——这解释了前面LooperThread代码的构建逻辑。

##Looper片段——Looper.loop()
LooperThread构建的最后一步就是调用Loop.loop()方法，它的源码如下:

```java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
	final Looper me = myLooper();
	if (me == null) {
		throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
	}
	final MessageQueue queue = me.mQueue;

	// Make sure the identity of this thread is that of the local process,
	// and keep track of what that identity token actually is.
	Binder.clearCallingIdentity();
	final long ident = Binder.clearCallingIdentity();

	for (;;) {
		Message msg = queue.next(); // might block
		if (msg == null) {
			// No message indicates that the message queue is quitting.
			return;
		}

		// This must be in a local variable, in case a UI event sets the logger
		Printer logging = me.mLogging;
			if (logging != null) {
  				logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
			}

			msg.target.dispatchMessage(msg);

			if (logging != null) {
				logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
			}

			// Make sure that during the course of dispatching the
			// identity of the thread wasn't corrupted.
			final long newIdent = Binder.clearCallingIdentity();
			if (ident != newIdent) {
				Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
			}

			msg.recycle();
	}
}
```
首先，同样要求myLooper不能为null（根据前面的分析，实现调用Looper.prepare（)方法即可）并从这个Looper中拿到相应的消息队列。之后我们进入一个无限循环，这个循环中最重要的事情就是queue.next()方法的调用，这个方法根据注释：可能会引起阻塞。相关的实现可以深入到MessageQueue中，这里涉及到Native代码，具体的理解不影响到原理讲述，暂且不表。

这里有一个很重要的判断：如果Message拿出来为空，则认为这个MessageQueue已经被丢弃了，整个循环会被打破返回。

再之后，就延伸到msg.target.dispatchMessage方法。后面继续阐述。这里总结一下:  
>我们在让一个线程具备消息循环调度能力的时候，首先需要在这个线程中调用Looper.prepare()，这个方法很重要，它会为一个线程绑定一个Looper，且要求只能绑定一个。绑定Looper的同时，Looper会创建一个MessageQueue。之后在线程中创建Handler的时候，会从Looper中引用MessageQueue放到Handler实例中去。至此Handler、Looper、MessageQueue三个类的关系建立。
>
>关系建立之后，调用Looper.loop()方法会让Loop运行起来：不断从MessageQueue中获取消息并dispatchMessage。至此，整个消息循环的核心部分构建完毕。

##消息的发送和处理
整个消息循环核心部分已经运行起来，剩下来的就是我们日常最长使用到的消息的发送和处理，接下来看看消息从发送到处理之间的流程。首先是sendMessage方法源码:

```java
/**
 * Enqueue a message into the message queue after all pending messages
 * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
 * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
 * You will receive it in {@link #handleMessage}, in the thread attached
 * to this handler.
 * 
 * @param uptimeMillis The absolute time at which the message should be
 *         delivered, using the
 *         {@link android.os.SystemClock#uptimeMillis} time-base.
 *         
 * @return Returns true if the message was successfully placed in to the 
 *         message queue.  Returns false on failure, usually because the
 *         looper processing the message queue is exiting.  Note that a
 *         result of true does not mean the message will be processed -- if
 *         the looper is quit before the delivery time of the message
 *         occurs then the message will be dropped.
 */
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
	MessageQueue queue = mQueue;
		if (queue == null) {
			RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
			Log.w("Looper", e.getMessage(), e);
			return false;
		}
	return enqueueMessage(queue, msg, uptimeMillis);
}
```
看源码就会发现，同名&功能相似的有好几个方法，但最终都会调用sendMessageAtTime方法，首先获取mQueue：注意这个方法在Handler新建的时候就获取了。之后调用了一个enqueueMessage方法，源码如下:

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
	msg.target = this;
	if (mAsynchronous) {
		msg.setAsynchronous(true);
	}
	return queue.enqueueMessage(msg, uptimeMillis);
}
```
首先，将msg.target赋值为本身，之后根据Handler新建时候传入的参数(前面忽略了它)设置msg属性，之后就调queue的enqueueMessage向队列中压入消息——__完成消息的发送__。

这里很重要的一点是__msg.target = this;__。查看Message的源码就可以看到，target是一个Handler变量。而在前面讲述的Looper.loop()方法实现中，取出消息后调用的方法是msg.target.dispatchMessage(msg);。嘿！这难道不是调用的Handler的dispatchMessage方法么？看源码果然发现一枚:

```java
public void dispatchMessage(Message msg) {
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
```
这里我们可以看到很多我们平时不常用的方法:

【 1 】Message是可以自带一个callback变量的，查看Meesgae源码可知这是一个Runnable变量，即我们可以使用一个Message实现post一个Runable对象的功能，因为handleCallback的实现如下：

```java
private static void handleCallback(Message message) {
	message.callback.run();
}
```
实际上读者可以查看Handler.postDelayed(Runnable)方法，内部正式做了这一层转换。

【 2 】如果实例化Handler的时候设置了mCallback对象（日常开发很少用，但的确存在这种用法），那么所有的小弟都先交给mCallback处理，mCallback是一个CallBack对象，这是一个接口:

```java
public interface Callback {
	public boolean handleMessage(Message msg);
}
```
这边可以通过返回true来拦截handleMessage的执行。

【 3 】以上都绕过了，才轮到handleMessage方法执行——关于这个方法，就不多说了，写Handler的时候十有八九重载了它；

到此，消息的处理也分析完毕。对于整个消息处理发送、循环、发送的机制基本清楚。剩下一块比较模糊：MessageQueue的分析比较少，原因是这块涉及到一些Native代码，且对理解整个Handler机制的理解影响不大，在这篇文章中暂不分析。

##一些问题
###主线程和主Handler
在所有的线程中，主线程是非常特殊的，开发时在主线程中新建Handler的实例不需要走上面的流程，直接创建即可，官方文档给出的解释是，主线程本身就已经启动Looper了。其实在Looper中，是有一个专门的方法做这件事的：

```java
/**
 * Initialize the current thread as a looper, marking it as an
 * application's main looper. The main looper for your application
 * is created by the Android environment, so you should never need
 * to call this function yourself.  See also: {@link #prepare()}
 */
public static void prepareMainLooper() {
	prepare(false);
	synchronized (Looper.class) {
		if (sMainLooper != null) {
			throw new IllegalStateException("The main Looper has already been prepared.");
		}
		sMainLooper = myLooper();
	}
}
```
看注释，看注释，看注释！！！重要的事情说三遍。这个方法就是创建主线程的Looper的。注释中说，这个创建是由Android Environment执行的，所以开发者__从来不需要__手动调用这个方法。而下面这个方法则是用于获取主Looper的：

```java
/** 
 * Returns the application's main looper, which lives in the main thread of the application.
 */
public static Looper getMainLooper() {
	synchronized (Looper.class) {
		return sMainLooper;
	}
}
```

###消息的recycle
Message内部维护着一个链表，所有可用的Message都挂在这个链表上，可以将它看做一个缓存池。关键的方法如下:

```java
private static final Object sPoolSync = new Object();
private static Message sPool;
private static int sPoolSize = 0;
private static final int MAX_POOL_SIZE = 50;

/**
 * Return a new Message instance from the global pool. Allows us to
 * avoid allocating new objects in many cases.
 */
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

public void recycle() {
	if (isInUse()) {
		if (gCheckRecycle) {
			throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
		}
		return;
	}
	recycleUnchecked();
}

void recycleUnchecked() {
	// Mark the message as in use while it remains in the recycled object pool.
	// Clear out all other details.
	flags = FLAG_IN_USE;
	what = 0;
	arg1 = 0;
	arg2 = 0;
	obj = null;
	replyTo = null;
	sendingUid = -1;
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

public static void updateCheckRecycle(int targetSdkVersion) {
	if (targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
		gCheckRecycle = false;
	}
}
```
代码很简单，不赘述。补充一点：消息会在被压入队列时置为FLAG_IN_USE。  

>【注意】在5.0以下的时候，gCheckRecycle开关是关闭的，这意味着回收时不会去检查Message是否在使用中。而在5.0以上就会检查，此处很坑爹的是：FLAG_IN_USE标志位的重置是在obtain的时候重置的，一旦进入消息队列则会被置位，之后再掉用recycle必然导致异常抛出，即5.0以及以上版本禁止recyle。