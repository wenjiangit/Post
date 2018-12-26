# Android内存管理

### 线程执行



![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter06/gc_thread.png)

​                                                         线程生命周期及其GC roots

### 内部类

内部类是封闭对象的成员，即外部类，并且可以访问外部类的所有其他成员。因此，内部类隐含地引用了外部类（参见图6-3）。因此，定义为内部类的线程会保持对外部类的引用，只要线程正在执行，外部类将永远不会被标记为垃圾回收。在以下示例中，只要该线程正在运行，外部类中的任何对象都必须留在内存中，以及内部``SampleThread`` 类中的对象。

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter06/outer_reference.png.jpg)

​                                      图6-3.具有内部类线程的对象依赖关系树

```java
public class Outer {

    public void sampleMethod() {
        SampleThread sampleThread = new SampleThread();
        sampleThread.start();
    }

    private class SampleThread extends Thread {
        public void run() {
            Object sampleObject = new Object();

            // Do execution

        }
    }
```

定义为本地类和匿名内部类的线程与外部类具有与内部类相同的关系，从而在执行期间保持外部类可从``GC`` 根访问。

### 静态内部类

静态内部类是封闭对象的类实例的成员。因此，在静态内部类中定义的线程保持对外部对象的类的引用，但不保留对外部对象本身的引用（图6-4）。因此，一旦其他引用消失，外部对象就可以被垃圾收集。例如，该规则适用于以下代码。

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter06/outer_static_reference.png.jpg)

​                                   图6-4.具有静态内部类线程的对象依赖关系树

```java
public class Outer {

    public void sampleMethod() {
        SampleThread sampleThread = new SampleThread();
        sampleThread.start();
    }

    private static class SampleThread extends Thread {
        public void run() {
            Object sampleObject = new Object();

            // Do execution

        }
    }
}
```

但是，在大多数情况下，程序员希望将执行环境（``Thread`` ）与任务（``Runnable`` ）分开。如果创建一个新的``Runnable`` 作为内部类，它将在执行期间持有对外部类的引用，即使它是由静态内部类运行的。下面的代码产生图6-5中的情况。

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter06/outer_static_reference_runnable.png.jpg)

​                     图6-5.具有静态内部类的线程执行外部Runnable的对象依赖关系树

```java
public class Outer {

    public void sampleMethod() {

        SampleThread sampleThread = new SampleThread(new Runnable() {
            @Override
            public void run() {
                Object sampleObject = new Object();

                // Do execution

            }
        });
        sampleThread.start();
    }

    private static class SampleThread extends Thread {
        public SampleThread(Runnable runnable) {
            super(runnable);
        }
    }
}
```

### 生命周期不匹配

Android上泄漏的根本原因是组件，对象和线程之间的生命周期不匹配。对象在堆上分配，可以有资格进行垃圾回收，并在被线程引用时保存在内存中。然而，在Android中，不仅是应用程序的生命周期需要处理，而且也是其组件的生命周期也需要处理。所有组件（Activity，Service，``BroadcastReceiver`` 和``ContentProvider`` ）都有自己的生命周期，不符合其对象的生命周期。

泄漏``activity`` 对象是最严重的 -- 也可能是最常见的组件泄漏。例如，一个``Activity`` 将引用保存到可能包含大量堆分配的视图层次结构中。图6-6说明了一个Activity的组件和对象生命周期。

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter06/activity_lifecycle.png)

​                                     Activity组件和实例的生命周期

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter06/activity_lifecycle_with_thread.png)

​                                                   执行线程的活动activity周期

### 线程间通信

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter06/handler_references.png.jpg)

​                                             可以接收消息的线程以及引用的对象

#### 发送数据消息

数据消息可以以各种方式传递;所选择的实现决定了内存泄漏的风险和大小。以下代码示例说明了实现陷阱.该示例包含一个带有Handler的Outer类来处理消息。处理程序连接到创建外部类的线程。

```java
public class Outer {

    Handler mHandler = new Handler() {
        public void handleMessage(Message msg) {
            // Handle message
        }
    };

    public void doSend() {
        Message message = mHandler.obtainMessage();
        message.obj = new SampleObject();
        mHandler.sendMessageDelayed(message, 60 * 1000);
    }
}
```

图6-9显示了执行线程中的对象引用树，从消息被发送到消息队列直到它被回收的时间，即Handler处理它之后。为简洁起见，引用链已缩短，仅涵盖我们想要追踪的关键对象。

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter06/data_message_reference.png.jpg)

​                                         图6-9.发送数据消息时的对象引用树。

#### 发送任务消息

发布一个``Runnable`` ，在一个使用``Looper`` 的消费者线程上执行，引发了和发送消息一样的顾虑，但是引入了额外的外部类引用：

```java
public class Outer {

    Handler mHandler = new Handler() {
        public void handleMessage(Message msg) {
            // Handle message
        }
    };

    public void doPost() {
        mHandler.post(new Runnable() {
            public void run() {
                // Long running task
            }
        });
    }
}
```

这个简单的代码示例将一个``Runnable`` 发布到调用``doPos`` t的线程。 ``Handler`` 和``Runnable`` 实例都引用外部类并增加潜在内存泄漏的大小，如图6-10所示。

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter06/task_message_reference.png.jpg)

​                                      图6-10.发布任务消息时的对象引用树

内存泄漏的风险随着任务的长度而增加。短期任务可以更好地避免风险。

**注意**  

> 一旦Message对象被添加到消息队列中，消息就会从消费者线程间接引用。消息等待的时间越长，队列中的时间越长，或者在接收线程上执行冗长的执行，内存泄漏的风险就越高。

### 避免内存泄漏

如前所述，大多数涉及线程的内存泄漏是由于内存在内存中的滞留时间超过所需时间而造成的。线程和``Handler`` 程序可能会 - 无意中 - 保持从线程``GC root`` 目录可访问的对象，即使它们不再被使用。让我们看看如何避免或减轻这些内存泄漏。

#### 使用静态内部类

本地类，内部类和匿名内部类都持有对它们声明的外部类的隐式引用。因此，他们不仅可以泄漏他们自己的对象，还可以泄漏来自外部类的引用。通常情况下，一个Activity及其视图层次结构可能通过外部类引用导致严重泄漏。

替代使用具有外部类引用的嵌套类，最好使用静态内部类，因为它们只引用全局class对象而不引用实例对象。这只是缓解了泄漏，因为在线程执行时，所有对来自静态内部类的实例对象的明确引用仍然存在。

#### 使用弱引用

正如我们所看到的，静态内部类不能访问外部类的实例字段。如果应用程序想要在工作线程上执行任务并访问或更新外部实例的实例字段，则这可能是一种限制。为了这个需要，``java.lang.ref.WeakReference`` 来解决：

```java
public class Outer {
    private int mField;
    private static class SampleThread extends Thread {

        private final WeakReference<Outer> mOuter;

        SampleThread(Outer outer) {
            mOuter = new WeakReference<Outer>(outer);
        }

        @Override
        public void run() {
            // Do execution and update outer class instance fields.
            // Check for null as the outer instance may have been GC'd.
            if (mOuter.get() != null) {
                mOuter.get().mField = 1;
            }
        }
    }
}
```

在代码示例中，外部类是通过弱引用引用的，这意味着静态内部类拥有对外部类的引用，并且可以访问外部实例字段。弱引用不是垃圾收集器引用计数的一部分，因为所有强引用即正常引用都是。因此，如果对外部对象的唯一剩余引用是来自内部类的弱引用，那么垃圾收集器将此对象视为符合垃圾回收的条件，并且可以从堆中释放分配的外部实例。

#### 停止工作线程执行

将``Thread`` ，``Runnable`` 和``Handler`` 作为静态内部类实现，使显式强引用无效或使用弱引用将缓解内存泄漏，但不能完全阻止它。正在执行的线程仍然可以保存一些不能被垃圾回收的引用。所以为了防止线程延迟对象的重新分配，只要不再需要它就应该终止。

#### 保留工作线程

#### 清理消息队列

发送到线程的消息可能在消息队列中处于等待状态，或者发送的执行延迟太长，或者时间戳较低的消息尚未执行完毕。如果消息在不再需要时挂起，则应将其从消息队列中删除，以便可以将其所有引用的对象解除分配。

```java
removeCallbacks(Runnable r)
removeCallbacks(Runnable r, Object token)
removeCallbacksAndMessages(Object token)
removeMessages(int what)
removeMessages(int what, Object object)
```

