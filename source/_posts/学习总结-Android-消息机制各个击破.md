---
title: 学习总结 -- Android 消息机制各个击破
date: 2017-04-19 16:16:18
categories: 学习笔记
tags: 消息机制
---

提到消息机制大家都不陌生，因为在日常开发中少不了和它打交道，郭婶也有过相关的篇章来讲述：[Android异步消息处理机制完全解析，带你从源码的角度彻底理解](http://blog.csdn.net/guolin_blog/article/details/9991569)，看完不禁感叹写得很棒很适用。说实话，郭婶这篇文章看了不下 3 遍，甚至还打印出来认真仔细研究，每次都会随笔记点东西，但都是零零散散的，这次结合《Android 开发艺术探索》来记录自己的学习过程。

<!-- more -->

Android 规定访问 UI 只能在主线程中进行，如果在子线程中访问 UI 就会出现异常，这是为什么呢？

> Android 系统为什么不允许子线程中访问 UI 呢？这是因为 Android 的 UI 控件是线程不安全的，如果在多线程中并发访问可能会导致 UI 控件处于不可预期的状态。

Android 消息机制的解析通过上面郭婶的文章就差不多了，但是这次我再来各个击破。

## Handler 工作原理

Handler 提供了一系列的 send() 和 post() 方法，post() 系列的方法最终都会调用 send() 系列的方法来发送消息。在一系列的 send() 方法中除了 sendMessageAtFrontOfQueue() 方法之外，其它的发送消息方法都会辗转调用到 sendMessageAtTime() 方法，但是最终都会调用 Handler 本身的 enqueueMessage() 方法。下面看看这几个方法：

```java
    // Handler.sendMessageAtTime()
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

	// Handler.sendMessageAtFrontOfQueue()
    public final boolean sendMessageAtFrontOfQueue(Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }

	// Handler.enqueueMessage()
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

在 enqueueMessage() 方法中首先把当前的 handler 对象存储起来：`msg.target = this`，然后再调用 MessageQueue 的 enqueueMessage() 方法将消息入队列，这就是通过 Handler 发送消息。

当调用 `Looper.loop()` 时就将消息出队列，最终消息由 Looper 交给 Handler 处理，即调用 Handler 的 dispatchMessage() 方法。dispatchMessage() 方法的实现如下：

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

首先检查 Message 的 callback 即 `msg.callback` 是否为 null，不为 null 就通过 handleCallback() 方法来处理消息。`msg.callback` 是一个 Runnable 对象，实际上就是通过 Handler 的 post 方法所传递的 Runnable 参数。handleCallback() 方法的逻辑也很简单：

```java
    private static void handleCallback(Message message) {
        message.callback.run();
    }
```

所以通过 post() 方法来发送消息就可以写成下面这样：

``` java
	Handler handler = new Handler();
	new Thread(new Runnable() {
		@override
      	public void run() {
          	// 这里进行耗时操作
        	handler.post(new Runnable() {
          		@override
          		public void run() {
            		// 这里进行 UI 操作
          		}
        	});
      	};
	}).start();
```

这样一来对 handleCallback() 方法的理解就更深刻了。回到 dispatchMessage() 方法继续判断 mCallback 是否为 null，如果不为 null 就调用 `mCallback.handleMessage()` 来处理消息，Callback 的定义如下：

``` java
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
```

Callback 是个接口，它的意义是可以用来创建一个 Handler 实例而不需要派生 Handler 的子类。来个例子最直观：

``` java
	class callback implements Handler.Callback {
		@Override
        public boolean handleMessage(Message msg) {
			// 3.进行 UI 操作
			return false;
		}
	}
	Handler handler = new Handler(new callback());
	new Thread(new Runnable() {
		@Override
		public void run() {
			// 1.进行耗时操作
			// 2.通过 handler 的一系列方法发送消息
	}
	}).start();
```

到这  `mCallback.handleMessage()` 也搞清楚了。这样 dispatchMessage() 到最后就调用 Handler 的 handlerMessage() 方法来处理消息。

至此通过 Handler 发送消息和处理消息算是过了一遍了。刚刚有说到 Handler 的 enqueueMessage() 方法最终是调用 MessageQueue 的 enqueueMessage() 方法来入队列的，接下来就看看消息是怎么入队列的。

## MessageQueue 工作原理

MessageQueue 被称为消息队列，它主要包含两个操作：插入( enqueueMessage )和删除( next )，但是实际上 MessageQueue 内部是通过一个单链表的数据结构来维护消息的，因为单链表在插入和删除上比较有优势。

enqueueMessage 的作用是往消息队列中插入一条消息；而 next 的作用是从消息队列中取出一条消息并将其从消息队列中移除，因为 MessageQueue 的读取操作本身会伴随着删除操作。下面来看看 MessageQueue 的 enqueueMessage() 方法：

```java
    boolean enqueueMessage(Message msg, long when) {
        ...
        synchronized (this) {
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
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
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
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

enqueueMessage() 方法的主要操作就是单链表的插入操作，它并没有把所有消息都存储起来，而是用 mMessage 来表示当前待处理的消息，然后根据时间顺序调用 `msg.next` 来指定下一条消息。

在 Handler 部分有说到当调用 `Looper.loop()` 时会将消息出队列，实际上 loop() 方法里面是调用了 MessageQueue 的 next() 方法来将消息出队列，看看 next() 方法：

```java
    Message next() {
       	...
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.
                  	// Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.
                      	// Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = 
                          				(int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
				...
            }
			...
        }
    }
```

next() 方法里面有一个无限循环的方法，如果 MessageQueue 中没有消息，那么 next() 方法就会一直阻塞在这，直到有新的消息到来；如果 MessageQueue 中存在 mMessage (待处理消息)，就将其出队列并将其从单链表中移除，然后把下一条消息赋值给 mMessage。

所以 MessageQueue 才是最终将消息入队列和出队列的，那就再来看看 `Looper.loop()` 的实现。

## Looper 工作原理

Looper 在 Android 消息机制中扮演着消息循环的角色，具体来说就是会不停地从 MessageQueue 中查看是否有新消息，如果有新消息就立刻处理，否则就会一直阻塞在那里。下面是 loop() 方法：

```java
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
			...
            msg.target.dispatchMessage(msg);
			...
        }
    }
```

首先判断 myLooper() 方法的返回值，如果返回值为 null 就抛出异常，意味着该线程还没有 Looper，否则就往下执行。

接着会无限循环的调用 `queue.next()`，所以 MessageQueue 的 next() 方法就是在这被调用的。每当有一个消息出队列后就将其传递到 msg.target 的 dispatchMessage() 方法中， msg.target 已经说过了，它其实就是 Handler，这样通过 Handler 发送的消息又回到了它自己的 dispatchMessage() 方法中。

唯一跳出死循环的时机是 MessageQueue 的 next() 方法返回 null。其实 Looper 提供了 quit() 和 quitSafely() 两个方法来退出一个 Looper，这两个方法的区别是：quit() 方法会直接退出 Looper；而 quitSafely() 方法只是设定一个退出标记，当消息队列中已有的消息处理完后才安全退出。

当 Looper 的 quit() 方法或 quitSafely() 方法被调用时，这两个方法内部就会去调用 MessageQueue 的 quit() 方法来通知消息队列退出，此时 MessageQueue 的 next() 方法就会返回 null。因此建议在不需要 Looper 的时候通过 quit() 或 quitSafely() 方法终止 Looper。

刚刚说到 myLooper() 方法，现在看看 myLooper() 方法：

``` java
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

`sThreadLocal.get()` 是个什么？请继续往下看：

## ThreadLocal 工作原理

用这么一句话来描述 ThreadLocal 再适合不过：**ThreadLocal 是一个线程内部的数据存储类，同一个 ThreadLocal 对象在不同线程中拥有不同的值。**

其实大家都知道，在子线程中创建 Handler 时必须先执行 `Looper.prepare()`，因为在 Handler 的无参构造方法中会通过 `(Looper)sThreadLocal.get()` 检查是否存在 looper，如果不存在就会抛异常。调用 prepare() 方法正是因为该方法内部会通过 `sThreadLocal.set(new Looper())`给当前线程设置 looper。

先来个小例子见识下 ThreadLocal 的厉害之处：

``` java
public class ThreadLocalActivity extends AppCompatActivity {

    private static final String TAG = "ljuns";
    private ThreadLocal<Boolean> mThreadLocal = new ThreadLocal<>();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_thread_local);

        // 主线程
        mThreadLocal.set(true);
        Log.d(TAG, "main : " + mThreadLocal.get());
        
        // 子线程 1
        new Thread(new Runnable() {
            @Override
            public void run() {
                mThreadLocal.set(false);
                Log.d(TAG, "thread 1 : " + mThreadLocal.get());
            }
        }).start();
        
        // 子线程 2
        new Thread(new Runnable() {
            @Override
            public void run() {
                Log.d(TAG, "thread 2 : " + mThreadLocal.get());
            }
        }).start();
    }
```

很简单的小例子，先创建出一个 ThreadLocal 对象：mThreadLocal，然后分别在三个线程中对它进行操作，按照刚刚那句话，打印的结果应该是：主线程的 mThreadLocal 是 true，子线程 1 的 mThreadLocal 是 false，子线程 2 由于没有赋值所以应该是 null 。实际的结果怎样呢？来看看：

![](http://oaydqd1yy.bkt.clouddn.com/image/tmp1f7f3501.png)

结果当然在意料之中，因为在此之前已经先试验过了，这就是 ThreadLocal 的奇妙之处。知其然知其所以然，接下来看看它的内部实现，首先看看 set() 方法：

```java
    public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        values.put(this, value);
    }
```

这个方法很简洁，首先获取当前线程，然后通过 values() 方法获取当前线程的 ThreadLocal 数据。为什么这么说呢？先来看看 values() 方法：

``` java
    Values values(Thread current) {
        return current.localValues;
    }
```

localValues 又是什么呢？其实它是 Thread 类内部的一个成员，localValues 内部有一个数组：`private Object[] table`，这个 table 数组才是真正存储数据的结构，ThreadLocal 的值就存在这个 table 数组中。所以实际上通过 values() 获取到的是 localValues 对象，而 localValues 里面的 table 数组才真正拥有 ThreadLocal 的数据。

回到 set() 方法中，如果拿到的 values 的值为 null，那就对其进行初始化，初始化后再通过 put() 方法进行存储。继续看 put() 方法：

``` java
	void put(ThreadLocal<?> key, Object value) {
      	cleanUp();
      	int firstTombstone = -1;

      	for (int index = key.hash & mask;; index = next(index)) {
        	Object k = table[index];

        	if (k == key.reference) {
          		table[index + 1] = value;
          		return;
        	}

        	if (k == null) {
          		if (firstTombstone == -1) {
            		table[index] = key.reference;
            		table[index + 1] = value;
            		size++;
            		return;
          	}
            table[firstTombstone] = key.reference;
            table[firstTombstone + 1] = value;
            tombstones--;
            size++;
            return;
        	}
          
			if (firstTombstone == -1 && k == TOMBSTONE) {
            	firstTombstone = index;
        	}
      	}
	}
```

这里只关注 put() 方法的存储规则，可以发现 ThreadLocal 的值存在 table 数组中 reference 的下一个位置。比如 reference 在 table 数组中的索引为 index，那么 ThreadLocal  的值就存在 table 数组中索引为 index + 1 的位置。

看完 set() 方法后再来看看 get() 方法：

``` java
    @SuppressWarnings("unchecked")
    public T get() {
        // Optimized for the fast path.
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values != null) {
            Object[] table = values.table;
            int index = hash & values.mask;
            if (this.reference == table[index]) {
                return (T) table[index + 1];
            }
        } else {
            values = initializeValues(currentThread);
        }
        return (T) values.getAfterMiss(this);
    }
```

get() 方法同样先是获取当前 localValues 对象，如果该对象不为 null，那就取出它的 table 数组并且找到 reference 在 table 数组中的位置，然后把 table 数组中 reference下一个位置的数据返回，因为该数据就是当前线程中 ThreadLocal 的值；如果 localValues 对象为 null 则返回初始值，默认情况下为 null。

ThreadLocal 的 set() 和 get() 方法操作的对象都是当前线程的 table 数组，因此也印证了刚刚对 ThreadLocal 的描述。理解完 ThreadLocal，相信对上面 Looper 工作原理的理解又会再次加深。

到这里才几乎算是把 Android 的消息机制走了一遍，最后贴上时序图一张来理解这个过程（刚开始学习使用时序图）:

![](http://oh0xbzubu.bkt.clouddn.com/image/tmp3a2fc0aa.png)

通过这个时序图可以很清晰的知道 **一个线程对应一个 Looper 对象，一个 Looper 对象对应一个 MessageQueue 对象**。为什么使用异步消息处理的方式就可以对 UI 进行操作了呢？这是因为 **Handler总是依附于创建时所在的线程**。所以在子线程中进行耗时操作后通过 Handler 把消息发送出去，最后消息又到了主线程中 Handler 的 handlerMessage() 方法进行 UI 操作。