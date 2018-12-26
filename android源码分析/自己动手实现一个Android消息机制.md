## 自己动手实现一个Android消息机制

之前分析了Android消息机制的相关源码,[原文链接](https://www.cnblogs.com/wenjianes/p/9943917.html),整个系统中的每个类的职责都十分清晰,想着自己是否可以动手实现一个这样的功能,说干就干.

### Message实现

```java
public class Message {
    int what;
    long when;

    Object object;
    //下一个
    Message next;

    Runnable callback;

    boolean inUse;

    Handler target;

    private static final Object poolLock = new Object();

    private static Message sPool;

    private static int size;

    private static final int MAX_SIZE = 50;

    @Override
    public String toString() {
        return "Message{" +
                "what=" + what +
                ", object=" + object +
                ", next=" + next +
                ", callback=" + callback +
                ", when=" + when +
                ", inUse=" + inUse +
                ", target=" + target +
                '}';
    }

    public static Message obtain() {
        synchronized (poolLock) {
            Message p = Message.sPool;
            if (p != null) {
                sPool = p.next;
                p.next = null;
                size--;
                return p;
            }
        }
        return new Message();
    }


    public void recycle() {
     /*   if (inUse) {
            throw new IllegalStateException("");
        }*/

        inUse = false;
        object = null;
        target = null;
        callback = null;
        next = null;
        what = 0;
        when = 0;

        synchronized (poolLock) {
            if (size < MAX_SIZE) {
                next = sPool;
                sPool = this;
                size++;
            }
        }
    }

```

这里没有添加那么多参数,不过基本功能还是都有的,也实现了类似的对象回收复用机制.

### Handler实现

```java
public class Handler {

    private final Looper looper;

    public Handler(Looper looper) {
        this.looper = looper;
    }

    public Handler() {
        Looper looper = Looper.myLooper();
        if (looper == null) {
            throw new RuntimeException("please call Loop.prepare first");
        }
        this.looper = looper;
    }

    public void sendMessage(Message message) {
        sendMessageAtTime(message, System.currentTimeMillis());
    }

    public void sendEmptyMessage(int what) {
        Message message = Message.obtain();
        message.what = what;
        sendMessageAtTime(message, System.currentTimeMillis());
    }


    private void sendMessageAtTime(Message message, long time) {
        message.when = time;
        message.target = this;
        looper.myQueue().enqueueMessage(message);
    }

    public void post(Runnable runnable) {
        this.postDelay(runnable, 0);
    }

    public void postDelay(Runnable runnable,long delay) {
        Message message = Message.obtain();
        message.callback = runnable;
        sendMessageAtTime(message, System.currentTimeMillis() + delay);
    }


    public void dispatchMessage(Message message) {
        if (message.callback != null) {
            message.callback.run();
        } else {
            handleMessage(message);
        }
    }


    public void handleMessage(Message msg) {

    }

```

具备发送消息和最终处理消息的能力,去掉了``CallBack`` 处理消息的流程,只是为了让类的职责更加明确.

### Looper实现

```java
public class Looper {

    private final MessageQueue queue;

    private static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<>();

    public Looper() {
        this.queue = new MessageQueue(true);
    }


    public static void loop() {
        Looper looper = myLooper();
        MessageQueue queue = looper.queue;
        for (; ; ) {
            Message message = queue.next();
            if (message == null) {
                //退出了
                System.out.println("loop 退出循环");
                return;
            }

            message.inUse = true;
            message.target.dispatchMessage(message);
            message.recycle();
        }
    }


    public static void prepare() {
        Looper looper = myLooper();
        if (looper != null) {
            throw new IllegalStateException("looper has initial");
        }
        sThreadLocal.set(new Looper());
    }


    public static Looper myLooper() {
        return sThreadLocal.get();
    }


    public MessageQueue myQueue() {
        return queue;
    }


}

```

实现了核心方法,具备退出的功能,细节就不讲了

### MessageQueue实现

```java
public class MessageQueue {

    private final PriorityBlockingQueue<Message> queue;

    private volatile boolean quitting = false;

    private final boolean allowQuit;

    MessageQueue(boolean allowQuit) {
        this.allowQuit = allowQuit;
        this.queue = new PriorityBlockingQueue<>(11, Comparator.comparingLong(o -> o.when));
    }

    public void enqueueMessage(Message message) {
        if (quitting) {
            throw new IllegalStateException("queue already quit");
        }
        queue.offer(message);
        //每次添加新消息,唤醒处于等待阻塞中的next方法.
        synchronized (this) {
            notifyAll();
        }
    }

    public Message next() {
        if (quitting) {
            return null;
        }

        long nextPollTimeoutMillis = 0;
        for (; ; ) {
            //进行超时等待,直到有新的消息插入或者超时
            synchronized (this) {
                try {
                    wait(nextPollTimeoutMillis > 0 ? nextPollTimeoutMillis : Integer.MAX_VALUE);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            //获取当前时间,这里为了简便,用的是System.currentTimeMillis(),毕竟java层没有开机计时的api
            long now = System.currentTimeMillis();
            //取出队头元素,也就是when最小的
            final Message msg = queue.peek();
            //为空说明队列中还没有消息,进行无限等待,等待时间是Integer.MAX_VALUE
            if (msg == null) {
                nextPollTimeoutMillis = -1;
            } else {
                long when = msg.when;
                //如果还没到消息处理的时间点,则计算需要等待的时间,便于下一次等待
                if (when > now) {
                    nextPollTimeoutMillis = Math.min(when - now, Integer.MAX_VALUE);
                } else {
                    //取出消息,之前的peek是查看消息,并不进行移除操作
                    return queue.poll();
                }
            }
            //如果队列已经退出,返回null
            if (quitting) {
                return null;
            }
        }
    }


    public void quit() {
        if (!allowQuit) {
            throw new UnsupportedOperationException("cant quit this queue");
        }
        if (quitting) {
            return;
        }
        quitting = true;
        queue.clear();
        synchronized (this) {
            notifyAll();
        }
    }
}

```

这个类可能是整个系统中最难处理的了,主要是``next``方法的逻辑.

这里保存消息的列表用的是``PriorityBlockingQueue``,它是一个线程安全且自带排序功能的队列,只需要在构造时传入比较器Comparator,即可对元素进行排序,不用自己去维护线程安全和元素排序.这是用的是线程等待和唤醒的机制,也能达成类似效果,可能性能没有使用c层代码来的高吧.

最难理解的部分都有详细注释,就不多说了.

### HandlerThread实现

```java
public class HandlerThread extends Thread {

    public HandlerThread() {
        super("HandlerThread");
    }

    private Looper looper;

    @Override
    public void run() {
        super.run();
        Looper.prepare();
        synchronized (this) {
            looper = Looper.myLooper();
            notifyAll();
        }
        Looper.loop();
    }


    public Looper getLooper() {
        synchronized (this) {
            while (looper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
        return looper;
    }


    public void quit() {
        looper.myQueue().quit();
    }
}
```

最后顺便实现了一下``ThreadHandler``,用于在子线程创建消息循环机制,后面有机会还是会单独写一篇的.

