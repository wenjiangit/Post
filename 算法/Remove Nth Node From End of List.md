### Remove Nth Node From End of List

### 题目描述:

> 给定一个链表,删除链表的倒数第n个节点,并返回头结点

```
即给定 1->2->3->4->5  ,n=2
得到 1->2->3->5
```

### 解题思路1:

>  获取链表的总长度``len``,拿到要删除节点的前一个节点``len-n-1``,删除要删除的节点,并返回头节点

### 参考代码:

```java
public class ListNode {
    int val;
    ListNode next;

    public ListNode(int val) {
        this.val = val;
    }

}

public static ListNode solution1(ListNode head, int n) {
        if (head == null) {
            return null;
        }

        //创建一个临时头部
        ListNode tempHead = new ListNode(0);
        tempHead.next = head;

        //计算链表长度
        int len = 0;
        ListNode next = head;
        while (next != null) {
            len++;
            next = next.next;
        }

        //获取要删除元素的前一个节点
        ListNode p = tempHead;
        for (int i = 0; i < len - n; i++) {
            p = p.next;
        }

        //删除要删除的元素
        ListNode deleteNode = p.next;
        p.next = deleteNode.next;
        deleteNode.next = null;

        //获得新链表的头节点
        ListNode newHead = tempHead.next;
        tempHead.next = null;

        return newHead;


    }

```

这里使用了临时头部解决了当需要删除的元素是链表头部需要进行特殊处理的情况

这种解法需要扫描链表两次,第一次获取链表长度,第二次定位要删除的节点,效率不是很高.

### 解题思路2

> 定义两个指针,一个快,一个慢,快的先走n+1步,然后慢的开始走,当快的走到头的时候,慢的也走到了要删除节点的前一个节点

### 参考代码:

```java
 public static ListNode solution2(ListNode head, int n) {
        if (head == null) {
            return null;
        }

        ListNode tempHead = new ListNode(0);
        tempHead.next = head;
        ListNode fast = tempHead;
        ListNode slow = tempHead;
       //快的先走n+1步
        for (int i = 0; i < n + 1; i++) {
            fast = fast.next;
        }
       //然后两个一起走
        while (fast != null) {
            fast = fast.next;
            slow = slow.next;
        }
        //删除指定节点
        ListNode deleteNode = slow.next;
        slow.next = deleteNode.next;
        deleteNode.next = null;
        //取出头节点,并把临时头节点链接中断
        ListNode ret = tempHead.next;
        tempHead.next = null;
        return ret;

    }
```

