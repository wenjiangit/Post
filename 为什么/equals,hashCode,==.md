#  equals,hashCode,==

对于这几个概念我一直感觉有些模糊,对于以下几个问题有些疑问

1. 到底什么时候该复写``equals``方法?
2. 为什么说复写``equals``方法的同时一定要复写``hashcode``? 不这样做会出现什么情况?
3. 哪些集合方法内部是用``equals``方法来比较两个对象相等?
4. ==到底比较的什么?

针对以上问题,我们来一一分析,我们知道``equals``和``hashcode``都是``Object``的成员方法,来看一下它们的实现

### Object的hashCode和equals

```java
  public boolean equals(Object obj) {
        return (this == obj);
   }
   
   
  public native int hashCode();
```

可以看到Object对象equals方法的默认实现是==,那么==到底比较的是什么呢?可以写个小例子来验证一下

```java
 static class Person {

        private String name;
        private int id;

        public Person(String name, int id) {
            this.name = name;
            this.id = id;
        }

     /*   @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            Person person = (Person) o;
            return id == person.id &&
                    Objects.equals(name, person.name);
        }

        @Override
        public int hashCode() {
            return Objects.hash(name, id);
        }*/
    }


    public static void main(String[] args) {
        Person p1 = new Person("wenjian", 111);
        Person p2 = new Person("wenjian", 111);
        Person p3 = p2;

        System.out.println("p1 == p2 :" + (p1 == p2));
        System.out.println("p1.equals(p2): " + p1.equals(p2));

        System.out.println("p2 == p3 :" + (p2 == p3));
        System.out.println("p2.equals(p3): " + p2.equals(p3));
    }

   //运行结果
   p1 == p2 :false
   p1.equals(p2): false
   p2 == p3 :true
   p2.equals(p3): true

```

当我们没有复写``equals``方法时,``equals``方法也就是使用==来比较,可以看到它们比较的结果是一致的,``p1``和``p2``引用的是处于堆区域中两个不同的对象,所以``==``比较为false,而``p2``和``p3``引用的是处于堆区域中的同一个对象,所以它们的``==``比较的结果为true,可以理解为``==``比较的是对象的内存地址,只有当内存地址一致代表引用的是同一个对象.

ok,此时我们可以复写equals方法,这个复写规则可以根据实际情况来自定义,有些对象只有当它们所有的属性都相等时我们才认为它们相等,而有些对象是具有唯一标识的,我们只需要比较它的唯一标识即可,这个需要根据不同情况来做不同的处理.

我这里采取的是前一种情况,即比较所有属性,看一下运行结果

````java
p1 == p2 :false
p1.equals(p2): true
p2 == p3 :true
p2.equals(p3): true
````

p1和p2引用的是两个不同的对象,所以它们的内存地址是不同的,==的结果为false,但它们的所有属性都是一致的,所以equals方法的结果为true.

这里可以有一个结论:

**两个对象==,则一定equals,而两个对象equals,却不一定==.**

### 集合中的哪些方法用到equals来比较两个对象是否相等

首先看一下最常用的``ArrayList``

``indexOf``

```java
   public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

``remove``

```java
  public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
```

``contains``

```java
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
```

可以看到``ArrayList``的关键性的查找,移除操作都是调用equals方法来进行对象比较的,所以当使用到这些集合时,需要特别注意你到底想如何定义两个对象相等,复写对应的equals方法.

### hashCode

官方对于Object的hashCode方法有这样一段解释

> ```
> Returns a hash code value for the object. This method is
> supported for the benefit of hash tables such as those provided by
> {@link java.util.HashMap}.
> ```

谷歌翻译: 返回对象的哈希码值。支持此方法是为了哈希表的好处，例如* {@link java.util.HashMap}提供的哈希表。

即主要是为了提供哈希表的支持,那我们来看一下熟悉的HashMap

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

/**
 * Implements Map.get and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @return the node, or null if none
 */
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

看一下``HashMap``的get方法,这里是基于``java1.8``的版本的,内部调用了``getNode``方法,

1. 通过key的hash值与数组长度-1进行与运算,定位table的index,取到first
2. 检查first是不是我们要找的,首先比较的hash值,然后比较的是key的==,最后比较的是key的equals.
3. 如果first不是我们要找的,则检查它是树节点还是链表节点,按照对应的结构查找

现在可以明白为什么复写equals方法必须复写``hashCode``方法了,因为当你定义的对象作为``HashMap``的键的时候,它先比较的是``hashCode``,如果``hashCode``不相等,那么就认为两个对象不相等.

如果不这样做的话那么像``HashMap``这样以``hash``表为基础的数据结构就没法正常工作了.

ok,到这里以上提出的几个疑问得到解答了.

Happy Coding, See you next time!

