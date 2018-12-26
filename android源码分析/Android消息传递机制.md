# Android消息传递机制

到目前为止，讨论的线程通信选项都是常规的``Java`` ，可用于任何Java应用程序。这些机制 - 管道，共享内存和阻塞队列 - 适用于``Android`` 应用程序，但由于其阻塞倾向而给``UI`` 线程带来了问题。使用具有阻塞行为的机制时，``UI`` 线程响应性处于危险之中，因为这可能偶尔挂起线程。

Android中最常见的线程通信用例在``UI`` 线程和工作线程之间。因此，Android平台为线程之间的通信定义了自己的消息传递机制。``UI`` 线程可以通过发送数据消息给后台线程处理来卸载长期任务。消息传递机制是一个非阻塞的消费者 - 生产者模式，生产者线程和消费者线程都不会在消息传递期间阻塞。消息处理机制是``Android`` 平台的基础，``API`` 位于``android.os`` 包中，并具有一组类，如图所示，用于实现这些功能。

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter04/handlerlooper_classdiagram.png)

- android.os.Looper

  与唯一的消费者线程关联的消息分发器。

- android.os.Handler

  消费者线程消息处理器，以及生产者线程将消息插入队列的接口。循环器可以有许多关联的处理程序，但它们都将消息插入同一个队列中。

- android.os.MessageQueue

  消费者线程上要处理的消息的**无限链接** 列表。每个``Looper`` 和``Thread`` -最多只有一个``MessageQueue`` 。

- android.os.Message

  要在消费者线程上执行的消息。

消息由生产者线程插入并由消费者线程处理，如图所示。

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter04/message_flow_overview.png.jpg)

​       多个生产者线程和一个消费者线程之间消息传递机制的概述。每条消息都是指队列中的下一条消息。

1. 插入

   生产者线程通过使用``Handler`` 连接到消费者线程在队列中插入消息，如``Handler`` 所示。

2. 取出

   ``Looper`` 在消费者线程中运行，并按顺序从队列中检索消息。

3. 分发

   ``Handler`` 负责处理消费者线程上的消息。一个线程可能有多个``Handler`` 实例来处理消息; ``Looper`` 确保消息被分派到正确的``Handler`` 。

### 基本消息传递示例

在我们详细剖析组件之前，让我们看看一个基本的消息传递示例，以使我们熟悉代码设置。

下面的代码实现了可能是最常见的用例之一。用户按下屏幕上可能触发长时间操作（例如网络操作）的按钮。为了避免延误``UI`` 的渲染，必须在工作线程上执行由虚拟``doLongRunningOperation（）`` 方法表示的长耗时操作。因此，设置仅仅是一个生产者线程（``UI`` 线程）和一个消费者线程（``LooperThread`` ）。

我们的代码设置了一个消息队列。 IT部门像往常一样在``onClick（）`` 回调中处理按钮点击，该回调在``UI`` 线程上执行。在我们的实现中，回调会在消息队列中插入一个虚拟消息。为了简洁起见，布局和``UI`` 组件已被排除在示例代码之外。

```java
public class LooperActivity extends Activity {

        LooperThread mLooperThread;

        private static class LooperThread extends Thread { 1

                public Handler mHandler;

                public void run() {
                        Looper.prepare(); 2
                        mHandler = new Handler() { 3
                                public void handleMessage(Message msg) { 4
                                        if(msg.what == 0) {
                                                doLongRunningOperation();
                                        }
                                }
                        };
                        Looper.loop(); 5
                }
        }

    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mLooperThread = new LooperThread(); 6
        mLooperThread.start();
    }

    public void onClick(View v) {
        if (mLooperThread.mHandler != null) { 7
            Message msg = mLooperThread.mHandler.obtainMessage(0); 8
                        mLooperThread.mHandler.sendMessage(msg); 9
        }
    }

        private void doLongRunningOperation() {
                // Add long running operation here.
        }

    protected void onDestroy() {
        mLooperThread.mHandler.getLooper().quit(); 10
    }
}
```

1. 定义作为消息队列使用者的工作者线程。
2. 将一个``Looper`` （隐式地与一个``MessageQueue`` ）关联起来。
3. 设置一个``Handler`` 供生产者用来在队列中插入消息。这里我们使用默认的构造函数，以便它绑定到当前线程的``Looper`` 。因此，这个``Handler`` 只能在``Looper.prepare（）`` 之后创建，否则它不会绑定任何东西。
4. 在消息已分派给工作线程时运行的回调。它检查什么参数，然后执行长耗时操作。
5. 开始将消息从消息队列分发到消费者线程。这是一个阻塞调用，所以工作线程不会完成。
6. 启动辅助线程，以便它可以处理消息。
7. 后台线程上``mHandler`` 的设置与``UI`` 线程上的这种用法之间存在竞争条件。因此，必须验证``mHandler`` 是否可用。
8. 使用``what`` 参数任意设置为0初始化消息对象。
9. 将消息插入队列中。
10. 终止后台线程。对``Looper.quit（）`` 的调用阻止了消息的分派并从阻塞中释放了``Looper.loop（）`` ，所以``run`` 方法可以完成，导致线程终止。

### 在消息传递机制中使用的类

现在让我们更详细地看看消息传递的具体组件及其使用。

#### ``MessageQueue`` 

消息队列由``android.os.MessageQueue`` 类表示。它使用链接消息构建，构成一个无边界的单向链接列表。生产者线程插入稍后将分派给消费者的消息。消息根据时间戳进行排序。具有最低时间戳值的待处理消息首先分派给消费者。但是，仅当时间戳值小于当前时间时才会分派消息。如果没有，则分派过程将等待直到当前时间已经超过时间戳值。

下图表示具有三个等待消息的消息队列，其中使用时间戳排序，其中``t1 <t2 <t3`` 。只有一条消息通过了调度屏障，当前时间。符合发送条件的消息的时间戳值小于当前时间，即图中的“Now”。

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter04/messages.png)

​                                    等待队列中的消息。最右边的消息首先在队列中进行处理。

如果Looper准备检索下一条消息时没有消息通过调度屏障，则消费者线程会阻塞。只要消息通过调度屏障，执行就会恢复。

生产者可以随时在队列中的任何位置插入新消息。队列中的插入位置基于时间戳值。如果新消息与队列中的待处理消息相比具有最低的时间戳值，则它将占用队列中的第一个位置，该位置将在下一个被分派。插入始终符合时间戳排序顺序。消息插入在``Handler`` 中进一步讨论。

#### MessageQueue.IdleHandler

如果没有要处理的消息，则消费者线程有一些空闲时间。例如，图``4-7`` 显示了消费者线程空闲的时间段。默认情况下，消费者线程只是在空闲时间等待新消息，但不是等待，而是可以利用该线程在这些空闲插槽期间执行其他任务。这个特性可以用来让非关键任务推迟执行，直到没有其他消息竞争执行时间。

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter04/idlehandler.png)

图4-7.如果没有消息通过调度屏障，则在需要执行下一个待处理消息之前可以使用时隙来执行

当一个挂起的消息已经被分派，并且没有其他消息已经通过调度屏障时，就会出现一个时隙，用户线程可以被用来执行其他任务。``android.os.MessageQueue.IdleHandler`` 接口获取此时间片;它是一个在消费者线程空闲时生成回调的侦听器。侦听器被附加到``MessageQueue`` 并通过以下调用与它分离：

```java
// Get the message queue of the current thread.
MessageQueue mq = Looper.myQueue();
// Create and register an idle listener.
MessageQueue.IdleHandler idleHandler = new MessageQueue.IdleHandler();
mq.addIdleHandler(idleHandler)
// Unregister an idle listener.
mq.removeIdleHandler(idleHandler)
```

空闲处理程序接口只包含一个回调方法：

```java
interface IdleHandler {
    boolean queueIdle();
}
```

当消息队列检测到消费者线程的空闲时间时，它会在所有已注册的``IdleHandler`` 实例上调用``queueIdle（）`` 。实施回调取决于应用程序。您通常应避免长时间运行的任务，因为它们会在运行时延迟待处理消息。

*true*

空闲处理程序保持活动状态;它将继续接收后续空闲时隙的回调。

*false*

空闲处理程序不活动;它将不会再接收到连续空闲时隙的回调。这与通过``MessageQueue.removeIdleHandler（）`` 移除侦听器是一回事。

#### 示例：使用IdleHandler来终止未使用的线程

当一个线程有空闲时隙,，并等待新消息处理时，所有注册到``MessageQueue`` 的`IdleHandlers` 都会被调用.空闲插槽可以出现在第一条消息之前，消息之间和最后一条消息之后。如果多个内容生成器应该在消费者线程上按顺序处理数据，则可以使用``IdleHandler`` 在处理所有消息时终止使用者线程，以便未使用的线程不会留在内存中。使用``IdleHandler`` ，不需要跟踪最后插入的消息，以了解线程何时可以终止。

> 警告 : 这个用例仅适用于生产线程无延迟地在``MessageQueue`` 中插入消息，以便消费者线程永远不会闲置，直到插入最后一条消息。

``ConsumeAndQuitThread`` 方法显示带有``Looper``和``MessageQueue`` 的消费线程的结构，在没有更多消息要处理时终止该线程。

```java
public class ConsumeAndQuitThread extends Thread implements MessageQueue.IdleHandler {

    private static final String THREAD_NAME = "ConsumeAndQuitThread";

    public Handler mConsumerHandler;
    private boolean mIsFirstIdle = true;

    public ConsumeAndQuitThread() {
        super(THREAD_NAME);
    }

    @Override
    public void run() {
        Looper.prepare();

        mConsumerHandler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                    // Consume data
            }
        };
        Looper.myQueue().addIdleHandler(this);
        Looper.loop();
    }


    @Override
    public boolean queueIdle() {
        if (mIsFirstIdle) { 
            mIsFirstIdle = false;
            return true; 
        }
        mConsumerHandler.getLooper().quit(); 
        return false;
    }

    public void enqueueData(int i) {
        mConsumerHandler.sendEmptyMessage(i);
    }
}
```

### Message

``MessageQueue`` 上的每个子项都是``android.os.Message`` 类。这是一个携带数据项或任务的容器对象，从来不会同时包含两者。数据由消费者线程处理，而任务仅在出队时执行，而您没有其他处理。

**Data message** 

数据集有多个参数可以传递到消费者线程。

**Task message** 

该任务由在消费者线程上执行的``java.lang.Runnable`` 对象表示。任务消息不能包含任务本身之外的任何数据。

``MessageQueue`` 可以包含数据和任务消息的任意组合。消费者线程按顺序处理它们，而与类型无关。如果消息是数据消息，则消费者处理数据。任务消息是通过让消费者线程上的``Runnable`` 执行来处理的，但消费者线程没有像处理数据消息一样接收到要在``Handler.handleMessage（Message）`` 中处理的消息。

消息的生命周期很简单：生产者创建消息，最终由消费者处理。这个描述对于大多数使用情况已经足够，但是当出现问题时，对消息处理的更深入的理解是非常宝贵的。让我们来看看消息在其生命周期中实际发生了什么，它可以分解成如图4-8所示的四种主要状态。运行时将消息对象存储在应用程序范围内的池中，以便重用以前的消息;这可以避免为每次传递创建新实例的开销。消息对象的执行时间通常非常短，许多消息按时间单位进行处理。

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter04/message_lifecycle.png.jpg)

​                                                              图4-8. 消息生命周期状态。

状态切换部分由应用程序部分控制，部分由平台控制。请注意，状态不可观察，并且应用程序无法遵循从一个状态到另一个状态的更改（尽管有方法可以跟踪消息的移动，稍后在观察消息队列中进行了解释）。因此，应用程序在处理消息时不应对当前状态做出任何假设。

#### Initialized

在初始化状态下，已创建一个具有可变状态的消息对象，并且如果它是数据消息，则会填充数据。应用程序负责使用以下调用之一创建消息对象。他们从对象池中获取一个对象。

- 明确的构造对象

- 工厂方法

  - 空消息

    ```java
    Message m  = new Message();
    ```

  - 数据消息

    ```java
    Message m = Message.obtain(Handler h);
    Message m = Message.obtain(Handler h, int what);
    Message m = Message.obtain(Handler h, int what, Object o);
    Message m = Message.obtain(Handler h, int what, int arg1, int arg2);
    Message m = Message.obtain(Handler h, int what, int arg1, int arg2, Object o);
    ```

  - 任务消息

    ```java
    Message m = Message.obtain(Handler h, Runnable task);
    ```

  - 拷贝构造函数

    ```java
    Message m = Message.obtain(Message originalMsg);
    ```

#### Pending

消息已被生产者线程插入到队列中，并且正在等待分派给消费者线程。

#### Dispatched

在这种状态下，``Looper`` 已经从队列中检索并删除消息。该消息已分派给消费者线程并正在处理中。此操作没有应用程序``API`` ，因为分发由``Looper`` 控制，没有应用程序的影响。当``Looper`` 调度消息时，它检查消息的传递信息，并将消息传递给正确的接收者。一旦派发，消息就在消费者线程上执行。

#### Recycled

在生命周期的这一点上，消息状态被清除并且实例返回到消息池。当消息线程完成执行时，Looper会处理消息的回收。消息的回收是运行时的处理程序，不应该由应用程序明确完成。

*注意*

> 一旦将消息插入到队列中，则不应更改内容。从理论上讲，在发送消息之前更改内容是有效的。但是，由于状态是不可观察的，消费者线程可能会处理消息，而生产者试图更改数据，从而引发线程安全问题。如果消息已被回收，则会更糟糕，因为消息已经返回到消息池，并可能被另一个生产者用于在另一个队列中传递数据。

### Looper

``android.os.Looper`` 类处理队列中的消息到相关处理程序的分派。所有通过调度屏障的消息（如图4-6所示）都可以通过Looper发送。只要队列中的消息符合发送条件，Looper就会确保消费者线程接收消息。当没有消息通过调度屏障时，消费者线程将阻塞，直到消息通过调度屏障。

Looper充当队列和线程之间的媒介。 Looper安装程序在线程的运行方法中完成：

```java
class ConsumerThread extends Thread {
    @Override
    public void run() {
        Looper.prepare(); 

        // Handler creation omitted.

        Looper.loop(); 
    }
}
```

1. 第一步是创建``Looper`` ，它使用静态``prepare（）`` 方法完成;它将创建一个消息队列并将其与当前线程关联。此时，消息队列已准备好插入消息，但不会分派给消费者线程。
2. 开始处理消息队列中的消息。这是一种确保``run（）`` 方法不会结束的阻塞方法; 在``run（）`` 块时，``Looper`` 将消息分派给消费者线程进行处理。

**Looper终结**

请求``Looper`` 使用``quit`` 或``quitSafely`` 停止处理消息：``quit（）`` 停止循环者从队列中分派更多消息;队列中的所有待处理消息（包括已通过调度屏障的消息）都将被丢弃。``quitSafely`` ，另一方面，只丢弃没有通过调度障碍的消息。在Looper终止之前，将处理可用于分派的待处理消息。

终止循环不会终止线程;它只是退出``Looper.loop（）`` 并让线程继续在调用循环调用的方法中运行。但是你不能启动旧的``looper`` 或新的``looper`` ，所以线程不能再入队或处理消息。如果你调用``Looper.prepare（）`` ，它会抛出``RuntimeException`` ，因为该线程已经有一个附加的``Looper`` 。如果调用``Looper.loop（）`` ，它将会阻塞，但是不会从队列中分派消息。

#### 主线程Looper

``UI`` 线程``Looper`` 和其他应用程序线程``Loopers`` 之间存在一些实际差异：

- 它可以通过``Looper.getMainLooper（）`` 方法从任何地方访问。
- 它不能被终止。 ``Looper.quit（）`` 抛出``RuntimeException`` 。
- 运行时通过``Looper.prepareMainLooper（）`` 将``Looper`` 关联到``UI`` 线程。这只能在每个应用程序中完成一次。因此，试图将主循环附加到另一个线程将引发异常。