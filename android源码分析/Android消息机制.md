**Q1:什么是Android消息机制？**

我们知道Android是基于消息循环机制的，即在主线程上开启一个无限循环，在这个循环中有一个Looper，它的职责就是不断地从消息队列(MessageQueue)中取出消息，交给消息关联的处理器(Handler)去处理。

**Q2:为什么需要了解Android消息机制，它和我们自己写的Android代码有何关联？**

可以说Android消息机制就是Android应用程序的核心，我们所编写的应用程序代码也是运行在消息机制之上，为什么这么说？因为Android应用程序的编写主要是围绕Android四大组件展开的，更准确的说都是我们所编写的代码都位于组件的生命周期函数中，而组件的生命周期又是在ActivityThread的内部类H的handleMessage方法中被调用的。可能会有人说，我的代码是写在一个单独的类中，并不是处于组件的生命周期方法中。其实写在哪里并不重要，重要的是它在哪里被调用的，在整个调用栈中是否包含组件生命周期方法，除非你写的这个类并没有被用到，那自然不存在调用栈。

**Q3:Android消息机制是如何实现的？**

这个问题需要从源码中寻求真相，ok，read the fucking source code

先从熟悉的入手，handler是我们在平时开发中接触的比较多的，OK，就它了。

#### Handler

##### 类的注释

> ```
> A Handler allows you to send and process {@link Message} and Runnable
> objects associated with a thread's {@link MessageQueue}.  Each Handler
> instance is associated with a single thread and that thread's message
> queue.  When you create a new Handler, it is bound to the thread /
> message queue of the thread that is creating it -- from that point on,
> it will deliver messages and runnables to that message queue and execute them as they come out of the message queue.
> ```

谷歌翻译：Handler允许您发送和处理与线程{@link MessageQueue}关联的{@link Message}和Runnable 对象。每个Handler *实例都与一个线程和该线程的消息*队列相关联。当你创建一个新的Handler时，它被绑定到正在创建它的线程的线程/ *消息队列 - 从那时起，*它将消息和runnables传递给该消息队列并在它们出现时执行它们。

> ```
> There are two main uses for a Handler: (1) to schedule messages and
> runnables to be executed as some point in the future; and (2) to enqueue an action to be performed on a different thread than your own.
> ```

这里还附带说明了Handler的两个主要用途：

1. 用于调度messages和runnables在将来某个时间点执行，也就是做延时操作。
2. 可以入队一个不在当前线程执行的任务，即做线程切换。

虽然之前没注意看handler类的注释，但好像还都用对了，平时好像也就是用它来做以上两种操作。

##### 构造方法

```java
 public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
 }
```

- looper：前面提到过，消息分发器，不断从消息队列中拿消息，并分发给Handler处理

- callback：定义了``handleMessage``方法，不过它和``handler``内部方法``handleMessage``有所不同，它是有``boolean``类型的返回值，它的优先级比内部的``handleMessage``高，返回``true``代表该消息它已经处理完毕，不需要向下分发给内部的``handleMessage``方法，``false``则代表可以向下分发。

  ```java
   public interface Callback {
          /**
           * @param msg A {@link android.os.Message Message} object
           * @return True if no further handling is desired
           */
          public boolean handleMessage(Message msg);
      }
  ```

- async：如果该值为`` true``，则由此``Handler``发送的消息都被设置``msg.setAsynchronous(true)``,则代表该消息有较高的优先级，Looper在拿到消息会判断是否是屏障消息(关于什么是屏障消息后面会提到)时都会遍历消息列表，查找是否有异步消息，如果有则优先执行，系统在发出界面绘制消息时就会设置这个标记，以便该消息能够优先被处理，为了保证界面绘制任务的优先级，所以我们初始化``handler``时保留它为``false``就好;

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

我们在对``Handler``进行初始化时，基本上是对以上构造函数的重载，即不传递Looper对象，此时它会调用``Looper.myLooper()``获取当前线程的``Looper``对象，如果为空，则会抛出运行时异常，程序会挂掉。

这也解释了一个问题，为什么不能在一般的子线程创建``Handler``？这里说一般不可以，但是只要调用了``Looper.prepare``创建了子线程的``Looper``，自然就可以创建``Handler``了，比如系统的``HandlerThread``，后面也会分析到。

##### 核心方法

```java
  public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
```

添加一个Runnable到消息队列，该Runnable被处理的线程即时创建Handler的线程

```java
 public final boolean postDelayed(Runnable r, long delayMillis)
    {
        return sendMessageDelayed(getPostMessage(r), delayMillis);
    }
```

添加一个Runnable到消息队列，并添加一定的延时时间

```java
 public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```

发送一个消息到消息队列

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

上面提到的3个方法添加的消息或任务最终都会被包装成``Message``对象被添加到消息对列中，接下来看看添加过程，即``enqueueMessage``方法的实现。

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

这里对``msg.target``进行了赋值，即将消息和``Handler``关联起来了，这里比较重要，因为分析Handler的**内存泄漏**和**消息分发处理**都与此有关。然后就是设置消息异步，之前提到过。接着调用了``queue.enqueueMessage``。

```java
  boolean enqueueMessage(Message msg, long when) {
      //参数校验
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
            //标记使用中
            msg.markInUse();
            //赋值when
            msg.when = when;
            //拿到链表头节点
            Message p = mMessages;
            //是否需要唤醒事件队列，其实也就是唤醒queue.next()
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                // 如果当前头节点为空或者消息的时间最小，则新加入的消息作为头节点
                // 可以知道，消息队列其实是一个以消息时间排序的链表，时间最小的位于最前面，最先被调度
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                //这里根据时间查找新消息在链表中的位置
                for (;;) {
                    prev = p;
                    p = p.next;
                    //当遍历到链表尾部，或者找到合适的位置
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                
                //插入新消息到链表中
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            //根据情况唤醒事件队列
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

我们可以知道MessageQueue其实是基于链表实现的，而且内部依据消息时间进行了排序。

到此，已经将Handler的一部分职责梳理清楚了，就是往消息队列中添加新消息。

下面进入消息机制第二个重要成员。

#### Looper

##### 官方说明

> ```
> Class used to run a message loop for a thread.  Threads by default do
> not have a message loop associated with them; to create one, call
> {@link #prepare} in the thread that is to run the loop, and then
> {@link #loop} to have it process messages until the loop is stopped.
> ```

谷歌翻译：用于为线程运行消息循环的类。默认情况下，线程没有与它们关联的消息循环;创建一个，在运行循环的线程中调用 {@link #prepare}，然后 {@link #loop}让它处理消息，直到循环停止。

官方还是牛逼，一两句话就把Looper的职责和基本用法说清楚了，666

##### 构造方法

```java
  private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

很简单，创建了消息队列，``quitAllowed``参数表明这个消息队列是否允许退出(HandlerThread内部创建的是允许退出的，而主线程创建的是不允许退出的，因为一旦退出，应用就崩溃了)，并记录下当前的线程。但需要注意的是，构造函数是用``private``修饰的，意味着我们没法在外部通过``new Looper（）``来创建``Looper``实例。

这种情况下一般都会提供静态方法来创建实例或者得到一个单例，当然Looper也是符合我们的一般情况的。

##### 核心方法

###### prepare()

```java
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

创建一个Looper实例，并保存在``ThreadLocal``中，而``ThreadLocal``是一个线程私有的成员变量，即只有在创建``ThreadLocal``线程中才能获取到正确的值。

- prepareMainLooper

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

初始化主线程的``Looper``，这里``prepare``方法传入的是false，即消息队列是不允许退出的。

###### getMainLooper()

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

获取主线程的Looper，这个函数可以放心调用，因为``prepareMainLooper``是在``ActivityThread``的``main``函数中调用的，所以在应用生命周期的任何时间都能获得到正确的``Looper``。

###### myLooper()

```java
/**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

获得当前线程的Looper对象，这个函数可能返回空，因为当前线程可能并未调用``Looper.prepare``进行初始化。

###### looper()

```java
 /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        //验证Looper是否被初始化了
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
        //开启循环
        for (;;) {
            Message msg = queue.next(); // might block
            //next返回null，代表消息队列已经退出，可以结束循环
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            ...省去日志打印信息
          
            try {
                //调用target方法对消息进行处理
                msg.target.dispatchMessage(msg);
                end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            
            ...
            
        }
    }
```

1. 验证Looper是否初始化
2. 开启消息循环
3. 调用``queue.next()``拿消息，next可能是阻塞的，因为当消息队列中没有消息的时候，next阻塞住，直到我们或系统往队列中添加了消息，然后``next``方法返回最新的消息。
4. 调用``target``即``Handler``的``dispatchMessage``对消息进行处理。

接下来看一下Message.next()具体实现

```java
 Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        //当queue退出的时候，ptr被置为0，这时返回null，此时looper也会退出循环
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
                //拿到链表头节点
                Message msg = mMessages;
                //判断是不是屏障消息，如果则遍历链表找到异步消息，优先执行
                //1.屏障消息的target为空，并且是直接插入到消息队列头部，目的是为了让绘制任务尽快被执行
                //2.当系统有屏幕绘制请求时，会发送一个屏障消息到消息队列
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
                        //如果没到消息的执行时间，则进行等待
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            //将异步消息移除
                            prevMsg.next = msg.next;
                        } else {
                            //设置新的链头
                            mMessages = msg.next;
                        }
                        //切断消息链，即移除该消息
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                //初始化闲时任务mPendingIdleHandlers，即当消息队列没有需要处理的消息时，
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            // 处理闲时任务
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                //如果queueIdle返回true，代表保留该任务，false则执行完了就移除任务
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```

1. 拿到链头消息，判断是不是屏障消息，是就便利链表拿到异步消息返回，否则返回链头消息。
2. 当队列没有消息需要处理时，如果我们通过``Looper.myQueue().addIdleHandler()``添加了闲时任务时，就会开始处理我们的闲时任务。

这个闲时任务有些鸡肋，当主线程繁忙的时候，它可能一直得不到执行，而当主线程闲的时候，如果我们没有将它移除的话，它可能很快地被执行多次。由于它执行时机的不确定性，暂时想不到很好的应用场景，以至于不看源码几乎不知道还有这么个机制。

接下来看Handler的dispatchMessage方法

```java
/**
     * Handle system messages here.
     */
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

 private static void handleCallback(Message message) {
        message.callback.run();
    }
```

如果是任务消息，直接调用``runnable.run``直接执行，如果是普通消息，则先判断有无``mCallback``，即构造方法传入的Callback,有的则调用``mCallback.handleMessage``，再根据返回值决定是否调用自身的``handleMessage``方法，也是我们继承``Handler``时需要重写的方法。

ok,到此Handler的职责分析完毕，就两个

- 添加消息到消息队列
- 处理Looper分发的消息

在生产者消费者模型中，Handler即是生产者，也是消费者。

#### MessageQueue

##### 官方说明

> ```
> Low-level class holding the list of messages to be dispatched by a
> {@link Looper}.  Messages are not added directly to a MessageQueue,
> but rather through {@link Handler} objects associated with the Looper.
> <p>You can retrieve the MessageQueue for the current thread with
> {@link Looper#myQueue() Looper.myQueue()}.
> ```

谷歌翻译:保存由* {@link Looper}分派的消息列表的低级类。消息不会直接添加到MessageQueue，而是通过与Looper关联的{@link Handler}对象添加。 * * <p>您可以使用* {@link Looper＃myQueue（）Looper.myQueue（）}检索当前线程的MessageQueue。

我们可以得到3点信息

- ``MessageQueue``保存Looper的消息列表
- 通过Handler添加消息到``MessageQueue``
- 通过``Looper.myQueue（）``获取当前线程的``MessageQueue``。

##### 构造方法

```java
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
```

``quitAllowed``:表示该消息队列是否允许退出,主线程默认是不允许的

``nativeInit``是一个本地方法,返回一个指针对象,并由mPtr保存,具体实现暂不深究

这里我的理解是:如果单纯地维护一个消息列表,根本不需要涉及c层代码,``java``层面完全搞定,但是要做到自由地低消耗地阻塞和唤醒,则需要借助``Linux``层面的一些技术.所以这里结合了两者,``java``层负责消息存储,c层负责进行阻塞和唤醒.

##### 核心方法

enqueueMessage和next方法前面已经详介绍过了

######  postSyncBarrier

```java
 public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            //从消息池获得一个消息
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                //插入链头
                msg.next = p;
                prev.next = msg;
            } else {
                //插入链表中间
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }

```

前面反复提到一个概念,屏障消息,这个方法就是说明屏障消息是如何被插入到消息队列的.

屏障消息最重要的一个特点就是没有相关联的Handler对象,即target属性为空,屏障消息的作用就不说了,反复提到过了.

还有一个点就是,消息队列中所引用的时间都是``SystemClock.uptimeMillis()``,即从开机到当前时间点的时间长度,为什么不是我们熟悉的``System.currentMillions()``,因为这个时间并不准确,它可以受到人为干预,而对于一个以时间作为排序依据的消息队列来说,这个肯定是不能接受的.

###### removeSyncBarrier

```java
  public void removeSyncBarrier(int token) {
        // Remove a sync barrier token from the queue.
        // If the queue is no longer stalled by a barrier then wake it.
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }
            if (p == null) {
                throw new IllegalStateException("The specified message queue synchronization "
                        + " barrier token has not been posted or has already been removed.");
            }
            final boolean needWake;
            if (prev != null) {
                prev.next = p.next;
                needWake = false;
            } else {
                mMessages = p.next;
                needWake = mMessages == null || mMessages.target != null;
            }
            p.recycleUnchecked();

            // If the loop is quitting then it is already awake.
            // We can assume mPtr != 0 when mQuitting is false.
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }
```

有添加就有移除,必定是成对出现的,否则的话队列后面的消息永远没法得到处理,消息机制就崩了,这个后面可以验证一下.

###### quit

```java
void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {
                return;
            }
            //将标记置为true
            mQuitting = true;

            if (safe) {
                //如果是安全退出,则移除msg.when大于当前时间点的所有消息
                removeAllFutureMessagesLocked();
            } else {
                //移除所有的消息
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            //唤醒阻塞,退出looper循环
            nativeWake(mPtr);
        }
    }
```

退出当前消息队列,有两种模式,安全和非安全

- 安全:移除当前时间点之后的所有消息
- 不安全:移除队列中的所有消息

###### removeCallbacksAndMessages

根据Handler移除队列中的消息和回调,都是常规的链表操作,就不贴代码了

#### Message

之前提到过最多的就是它了,现在来详细了解一下

##### 官方说明

> ```
> Defines a message containing a description and arbitrary data object that can be
> sent to a {@link Handler}.  This object contains two extra int fields and an
> extra object field that allow you to not do allocations in many cases.
> ```

谷歌翻译:定义一条消息，其中包含可以*发送到{@link Handler}的描述和任意数据对象。此对象包含两个额外的int字段和一个* extra object字段，允许您在许多情况下不进行分配。

##### 构造函数

```java
/** Constructor (but the preferred way to get a Message is to call {@link #obtain() Message.obtain()}).
    */
    public Message() {
    }

```

默认的构造方法,不做任何操作,但是注释告诉我们更好的获取Message实例的方法是调用``Message.obtain()``

为什么呢?那看一下obtain方法的实现

##### 核心方法

###### obtain

```java

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
```

内部使用链表结构维护了一个消息池,每次取链头节点,如果链表为空,新创建一个消息返回.

再看一下回收方法

###### recycle

```java
 /**
     * Return a Message instance to the global pool.
     * <p>
     * You MUST NOT touch the Message after calling this function because it has
     * effectively been freed.  It is an error to recycle a message that is currently
     * enqueued or that is in the process of being delivered to a Handler.
     * </p>
     */
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

    /**
     * Recycles a Message that may be in-use.
     * Used internally by the MessageQueue and Looper when disposing of queued Messages.
     */
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        //重置参数
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
        //将当前消息添加到链表头部
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```

逻辑很简单,重置各种成员变量,然后添加到链表头部.

需要注意的是,我们并不需要手动去调用recycle方法,在消息被消费掉的时候,Looper内部自动为我们调用了.

为什么需要消息池呢?每次创建一个会产生什么问题呢?

嗯,频繁地新建和销毁对象会造成频繁``gc``,内存抖动以及界面卡顿.

到此为止,消息创建,分发,消费,回收整个流程都分析完毕,那么看了这么多源码到底有哪些收获呢?

至少可以清晰地回答以下问题:

- q1: Android消息机制是什么?涉及哪些主要的类?
- q2: MessageQueue是基于哪一个数据结构实现的,有何特点?
- q3: 为什么Message中when使用``SystemClock.uptimeMillis()``?
- q4: 什么是屏障消息? 怎么向消息队列添加一个屏障消息? 以及屏障消息的作用?
- q5: 如果添加闲时任务到消息队列?
- q6: 如何参照Message回收机制构建一个对象回收池?
- q7: 如何在子线程创建Handler实例?
- q8: 如何利用Looper构建一个消息循环系统?
- q9: 什么情况下会发生内存抖动,界面卡顿?

差不多了,大概就这些吧?最后附上一张网络上盗来的图(其实是自己不会画)

![消息循环机制](http://gityuan.com/images/handler/handler_java.jpg)

第一次写这么长的源码分析文章,且看且珍惜吧!

