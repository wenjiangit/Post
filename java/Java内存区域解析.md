# Java内存区域解析

`` java``虚拟机在执行``java``程序的过程中会把它所管理的内存区域划分为若干个不同的数据区域.每个区域都有各自的用途,如下图所示:

![](https://img2018.cnblogs.com/blog/1530337/201812/1530337-20181226162557191-1594989536.png)

## 程序计数器

是一块较小的区域,用来记录当前执行的字节码指令的位置,在多线程环境中线程的挂起与恢复都要依赖它来完成,属于线程私有的,每个线程独有一个,生命周期与线程一致,是唯一一个不会发生``OutOfMemoryError``异常的区域.

## Java虚拟机栈

和程序计数器一样,也是线程私有的,生命周期与线程一致.

每个方法在执行时会创建一个栈帧,用于存**储局部变量表,操作数栈,方法出口**等信息.方法从调用到返回伴随着一个栈帧入栈和出栈的过程.

当线程请求的深度大于虚拟机请求的深度是会抛出``StackOverFlowError``异常.这种情况很容易模拟,如下:

```java
public class JavaVMStackSOF {

    private int stackLength = 1;

    public void stackLeak() {
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) {
        JavaVMStackSOF oom = new JavaVMStackSOF();

        try {
            oom.stackLeak();
        } catch (Exception e) {
            System.out.println("stack length : " + oom.stackLength);
            throw e;
        }
    }
 }
```

运行结果如下

```java
Exception in thread "main" java.lang.StackOverflowError
	at chapter2.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:15)
	at chapter2.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:16)
	at chapter2.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:16)
	at chapter2.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:16)
	...
```

当递归调用某一方法,而没有设置退出条件时,很容易就栈溢出的,所以写类似递归的代码时,一定要确定递归基,即递归退出的条件.

除此之外,这个区域也可能抛出``OutOfMemoryError``异常,但是出现条件很苛刻,基本上遇不到.

虚拟机参数设置

```
-Xss128k 设置虚拟机栈的大小为128k
```

如果是基于idea开发,可以通过如下步骤配置当前程序的虚拟机参数

![](https://img2018.cnblogs.com/blog/1530337/201812/1530337-20181226165513619-257269441.png)

![](https://img2018.cnblogs.com/blog/1530337/201812/1530337-20181226165739248-991921477.png)

点击``Edit Configurations``----> 编辑``VM options``即可.

## 本地方法栈

与``java``虚拟机栈类似,主要区别是``java``虚拟机栈为``java``方法服务,而本地方法栈为``native``方法服务.该区域也会抛出``OutOfMemoryError``和``StackOverFlowError``异常.

## Java堆

这个区域应该是整个虚拟机管理的内存区域中占比最大的一个,我们使用``new``关键字生成的对象实例都存在这个区域,该区域是线程共享的,和虚拟机的生命周期一致.当堆无法为新的对象实例分配内存时,则会抛出``OutOfMemoryError``异常,这个很容易测试

虚拟机参数配置

```
-Xms20m 设置堆区域的最小大小为20M
-Xmx20m 设置堆区域的最大大小为20M
```

在设置堆的大小为``20m``后,运行以下程序

```java
public class HeapOOM {

    static class OOMObject {

    }

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<>();

        while (true) {
            list.add(new OOMObject());
        }
    }
  }
```

运行结果

```java
java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3210)
	at java.util.Arrays.copyOf(Arrays.java:3181)
	at java.util.ArrayList.grow(ArrayList.java:265)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:239)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:231)
	at java.util.ArrayList.add(ArrayList.java:462)
	at chapter2.HeapOOM.main(HeapOOM.java:23)
```

## 方法区

方法区和``java``堆一样,是线程共享区域,它用于存储已被虚拟机加载的**类信息,常量,静态变量,即时编译器编译**后的代码数据,开发中遇到的``class``对象的相关信息以及静态变量就存储在该区域,当方法区无法满足内存分配要求时,也会抛出``OutOfMemoryError``异常.

### 运行时常量池

运行时常量池是方法区的一部分,用于存放编译器生成的**各种字面量和符号引用**.这些信息一般是编译期确定的,但我们也可以在运行时将新的常量放到常量池,调用``String.intern()``方法.

参考:<<深入理解java虚拟机>>