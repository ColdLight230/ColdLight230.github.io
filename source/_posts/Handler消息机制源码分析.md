---
title: Handler消息机制源码分析
date: 2017-9-23 15:01:17
tags: Android
categories: Android
---

Handler 消息机制是我们安卓中最常见的一种线程间通讯，通过使用 Handler 消息机制，我们可以实现在子线程中处理耗时操作，在主线程中更新UI。那么为什么能实现这种操作，我们来看一下 Handler 的源码，来深入了解一下 Handler 消息机制的原理。

### 消息的发送与排序
首先，我们先从` mHandler.sendMessage(message)` 这里看进去。
```java
public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```
<!--more-->
```java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```
```java
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
依次调用这三个方法，这些方法是处理发送实时消息和延迟消息的区别，拿到每个消息被传递时的具体时间戳。这样就能按照严格的顺序来发送对应的消息。
```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
这里，MessageQueue 就是一个队列，用于承载 Message 。这里的 ` queue.enqueueMessage(msg, uptimeMillis)` 就是将消息按照时间顺序依次排好队，等待被取用。
```java
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

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
```
在这里，我们可以知道 Message 的底层结构是一个链表的形式，至于为什么要使用链表这种数据结构，那是由于我们的消息需要频繁的插入取出。通过 ` enqueueMessage(Message msg, long when) `这个方法处理后，我们的 MessageQueue 就是一个有序的队列了。

### Looper 处理消息队列
```java
public Handler(Callback callback, boolean async) {
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
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
从 Handler 的构造方法中，可以看出初始化 mLooper 和 mQueue 的操作，纵观 Looper 类，我们可以发现，Looper 维护着一个存放 Looper 对象的 ThreadLocal。ThreadLocal 是一个线程存储区域，确保着每个线程有自己的存储区域，跨线程是不能访问的。这样，就确保了每个线程有本线程的 Looper。这样，在 Looper 没有初始化之前，Handler是不能创建出来的，否则会出现下面的错误 `java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()`。对于主线程，app在启动的时候，在 ActivityThread 的 `main` 方法中就有这主线程的 Looper 初始化的操作。再接着就是 `Looper.loop()` 方法的执行。
```java
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
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

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

            msg.recycleUnchecked();
        }
```
主要代码就是开启了一个死循环，去调用 `queue.next()` 从消息队列里取消息，如果消息队列里存在未处理的消息，那么就拿出来，交给这个 Message 所对应的 Handler 的 `handleMessage(Message msg)`。这个就是我们自己的 Handler 中重写的方法。我们在这里就可以根据对应的消息来进行对应的UI处理了。

### Message 中的消息池
在使用过程中，我们获取 Message 对象的方法一般都是使用 `Message.obtain()`
```java
private static final int MAX_POOL_SIZE = 50;

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

```
从中我们可以看到 `obtain` 是从一个大小为50的消息池中去取消息对象。在使用完后，回收消息，重新放入消息池。
