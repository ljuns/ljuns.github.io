---
title: 学习总结 -- Android 消息机制各个击破
date: 2017-04-19 16:16:18
categories: 学习笔记
tags: 消息机制
---

提到消息机制大家都不陌生，因为在日常开发中少不了和它打交道，郭婶也有过相关的篇章来讲述：[Android异步消息处理机制完全解析，带你从源码的角度彻底理解](http://blog.csdn.net/guolin_blog/article/details/9991569)，看完不禁感叹写得很棒很适用。说实话，郭婶这篇文章看了不下 3 遍，甚至还打印出来认真仔细研究，每次都会随笔记点东西，但都是零零散散的，这次结合《Android 开发艺术探索》来记录自己的学习过程。

<!— more —>

Android 规定访问 UI 只能在主线程中进行，如果在子线程中访问 UI 就会出现异常，这是为什么呢？

> Android 系统为什么不允许子线程中访问 UI 呢？这是因为 Android 的 UI 控件是线程不安全的，如果在多线程中并发访问可能会导致 UI 控件处于不可预期的状态。

Android 消息机制的解析通过上面郭婶的文章就差不多了，所以这次来个各个击破。

## ThreadLocal 工作原理

在子线程中创建 Handler 时必须先执行 `Looper.prepare()`，因为 prepare() 方法内部会通过 `sThreadLocal.set(new Looper())`给当前线程设置 looper；而在 Handler 的无参构造方法中则会通过 `(Looper)sThreadLocal.get()` 检查是否存在 looper，如果不存在就会抛异常。接下来开始分析 ThreadLocal 的工作原理：

用这么一句话来描述 ThreadLocal 再适合不过：**ThreadLocal 是一个线程内部的数据存储类，同一个 ThreadLocal 对象在不同线程中拥有不同的值。**

为了验证这句话，先来看个小例子：

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

get() 方法同样先是获取当前 localValues 对象，如果该对象不为 null，那就取出它的 table 数组并且找到 reference 在 table 数组中的位置，然后把 table 数组中 reference下一个位置的数据返回，因为该数据就是当前线程中 ThreadLocal 的值；如果 localValues 对象为 null，则返回 `values.getAfterMiss(this)`，在这种情况下 getAfterMiss() 方法最终返回的是 null。

至此，ThreadLocal 的 get()、set() 方法就算是说完了，对于前面的小例子也就更加深刻了。

##  MessageQueue 工作原理

MessageQueue 被称为消息队列，它主要包含两个操作：插入和删除，但是实际上 MessageQueue 内部是通过一个单链表的数据结构来维护消息的，因为单链表在插入和删除上比较有优势。enqueueMessage 的作用是往消息队列中插入一条消息；而 next 的作用是从消息队列中取出一条消息并将其从消息队列中移除，因为 MessageQueue 的读取操作本身会伴随着删除操作。

下面分别来看看 enqueueMessage() 和 next() 方法，首先是 enqueueMessage() 方法：

``` java
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

enqueueMessage() 方法的主要操作就是单链表的插入操作，它并没有把所有消息都存储起来，而是用 mMessage 来表示当前待处理的消息，然后根据时间顺序调用 `msg.next` 来指定下一条消息。接下来是 next() 方法：

``` java
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
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
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
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

这个方法比较长，这里留下主要的操作，它的简单逻辑：next() 方法首先判断是否有调用 quit() 方法，如果调用了 quit() 方法那么 ptr 为 0，直接返回 null，否则就往下执行。next() 里面有一个无限循环的方法，如果 MessageQueue 中没有消息，那么 next() 方法就会一直阻塞在这，直到有新的消息；如果 MessageQueue 中存在 mMessage (待处理消息)，就将其出队列并从单链表中移除，然后把下一条消息赋值给 mMessage。

## Looper 工作原理





## Handler 工作原理



