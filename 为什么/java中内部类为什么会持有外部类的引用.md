java中内部类为什么会持有外部类的引用?

当我们分析内存泄漏的场景时,总会想到不能在内部类中做耗时操作,因为它会持有外部类的因为,导致外部类的实例在生命周期结束的时候没有办法及时释放,这就造成了内存泄漏.

好像这就是一个公理一样,就是人们说着说着就都认可它了,却没有人能说出个为什么.

今天我们就来分析一下为什么吧

首先来看一个例子

```java
public class Outer {

    private int count;

    //匿名内部类1
    private StaticInner si1 = new StaticInner(){
        @Override
        public void doAction() {
            count++;

        }
    };

    private StaticInner si2;

    private Inner i3;


    public void setInner(StaticInner inner) {
        this.si2 = inner;
    }


    public Outer() {
        i3 = new Inner();
        //匿名内部类2
        setInner(new StaticInner(){
            @Override
            public void doAction() {
                super.doAction();
                count++;
            }
        });
    }

    public void doAction() {
        si1.doAction();
        si2.doAction();
        i3.doSomething();
    }


    /**
     * 内部类
     */
    private class Inner {

        public void doSomething() {
            count++;
        }
    }


    /**
     * 静态内部类
     */
    private static class StaticInner{

        public void doAction() {

        }
    }
}
```

以上代码概述了我们平时写代码时常见的几种内部类和匿名内部类以及静态内部类的写法,并访问了外部类的成员变量,当然静态内部类没法访问,编译器会报错.

然后通过javac命令编译一下.java源文件,得到几个.class字节码文件,这时如果你尝试打开字节码文件,会发现一堆乱码,可以用[jd-gui](http://jd.benow.ca/)字节码反编译工具打开.

我们可以发现好几个.class文件,一个一个来看

### Outer$1.class

```java
class Outer$1
  extends Outer.StaticInner
{
  Outer$1(Outer paramOuter)
  {
    super(null);
  }
  
  public void doAction()
  {
    Outer.access$108(this.this$0);
  }
}

```

这个类代表我们声明成员变量``si1``的匿名内部类,可以看到它继承自静态内部类``StaticInner``,它还定义了一个构造函数,并传入了内部类的实例作为参数,虽然不知道super(null)具体实现,但是我们知道它要访问``paramOuter``实例的成员变量,必须得使用它的引用来访问,所以它肯定是持有了外部类实例的引用.

### Outer$2.class

```java
class Outer$2
  extends Outer.StaticInner
{
  Outer$2(Outer paramOuter)
  {
    super(null);
  }
  
  public void doAction()
  {
    super.doAction();
    Outer.access$108(this.this$0);
  }
}

```

这个和上面提到的``Outer$1.class``类似,所以这两种匿名内部类是没有任何本质区别的,不管是定义成员变量还是方法传参.

### Outer$Inner.class

```java
class Outer$Inner
{
  private Outer$Inner(Outer paramOuter) {}
  
  public void doSomething()
  {
    Outer.access$108(this.this$0);
  }
}

```

这是定义的内部类,可以看到编译器也为它单独生成了一个``.class``文件,构造函数被定义为私有的,且同样传入了外部类的实例,所以它也是持有了外部类实例的引用.

### Outer$StaticInner.class

```
class Outer$StaticInner
{
  public void doAction() {}
}
```

这是静态内部类编译后的字节码文件,编译器并没有为它添加额外的构造函数,所以它其实和我们的外部类没有任何关系,这是写在同一个``.java``源文件中而已.

分析到这里,应该算可以证明为什么静态内部类会持有外部类的实例了

在分析内存泄漏的时候,它应该是我们应该重点关注的对象.