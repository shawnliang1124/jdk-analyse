##### 1.数据结构  

我们看到LinkedList的数据结构，明显就没有那么复杂  

![image](https://wx1.sinaimg.cn/mw1024/007R8l8Fly1g9jjels48wj30k005a0t9.jpg)  

很简单，双向链表嘛，记录一个首节点，一个尾部节点，还有linkedList的长度  

这个Node<E>内部类，我们可以看一下底层的结构：  
```
  private static class Node<E> {
        E item; //记录节点的当前内容
        Node<E> next;   //节点的后一个内容
        Node<E> prev;   //节点的前一个内容

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```  



***  


##### 2.add(E e)方法   
下面看下add(E e)方法：  
```  
 public boolean add(E e) {
        linkLast(e);
        return true;
    } 
```  

```  
 /**
     * Links e as last element.
     */
    void linkLast(E e) {
        final Node<E> l = last;     //找到之前尾部节点
        final Node<E> newNode = new Node<>(l, e, null); //pre前结点 ,要插入的内容,next后节点
        last = newNode; //将最新的节点指向last
        if (l == null)
            first = newNode;    //如果之前的尾部节点为空，证明linkedList之前就是空的，那么newNode也指向first头节点
        else
            l.next = newNode;   //之前尾部节点不为空，要把前尾部节点的引用置为新的节点
        size++;    //链表长度++
        modCount++; //AbstractList的modCount++（List的结构变动次数）
    }

```  

**这个add方法也是比较简单，调用了底层的linkLast(E e)方法。
无非就是找到原本linkedList链表的尾巴节点，原本尾巴节点为空，证明LinkedList链表为空，新的值赋给头部节点和尾巴节点，否则，将原本的尾巴节点的next设置为新的节点node即可。**  

***  

##### 3.add(int index,E element)  
这个方法是往指定的链表下标插入值。具体的逻辑如下：  
```  
    public void add(int index, E element) {
        checkPositionIndex(index);  //检查下标是否合法，这种检查的东西不用看

        if (index == size)
            linkLast(element);  //如果索引的值和linkedList的大小一样，意味着是在最后插入
        else
            linkBefore(element, node(index));   //否则在last的之前插入
    }

```  
linkLast(element)方法之前已经分析过，现在看 linkBefore(element, node(index))方法，这里我们先注意这个==node(index)==方法。实际上这个==node(index)==方法也是LinkedList的get方法的底层，我们先进去看看node(index)方法底层。   

```  
  /**
     * Returns the (non-null) Node at the specified element index.
    * 返回具体索引下标的node节点
     */
    Node<E> node(int index) {   //返回具体索引下标的node节点
        // assert isElementIndex(index);

        if (index < (size >> 1)) {  //假如index下标 小于 linkedList大小/2
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next; //从0遍历到index，list越长效率越低
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)  //否则就从尾部节点开始遍历
                x = x.prev;
            return x;
        }
    }
```  

我们可以看到，这个方法是将链表一分为2，在链表越长的情况下，越靠近中间的下标索引，就越难寻找。  

所以实际上，说ArrayList插入慢，LinkedList插入快，实际上是不一定的，如果我要插入的下标正好是链表长度的/2，时间复杂度直接就是o(n)   

以下这个文章做了验证： 
https://blog.csdn.net/hzj1998/article/details/97410514  

我们再看看linkBefore(element,node(index))这个方法.  


```  
  /**
     * Inserts element e before non-null Node succ.
     */
    void linkBefore(E e, Node<E> succ) {    //e要插入的值，succ是之前该下标的node节点,以下简称老节点
        // assert succ != null;
        final Node<E> pred = succ.prev; //找到老节点的pre节点
        final Node<E> newNode = new Node<>(pred, e, succ);  //创建一个新的节点
        succ.prev = newNode;    //将老节点的pre设置成新节点
        if (pred == null)
            first = newNode;    //老节点之前的pred为空的话，证明链表之前就是空的
        else
            pred.next = newNode;    //老节点的pre节点的next节点为新的node
        size++; //链表长度++
        modCount++; //AbstractList的结构调整次数++
    }

```  

实际上无非就是找到原本下标索引的节点，然后改下prev和next的引用，在老节点前的prev设置成新节点，新节点的next设置成老节点，老节点原本的prev的next设置成新节点即可。  



***  

##### 4.remove(int index)方法  

```  
    public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}


```  

remove方法也是直接看unlink(index)方法吧   

``` 
 /**
     * Unlinks non-null node x.
     */
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;  //拿到要删除元素的内容
        final Node<E> next = x.next;   //删除元素的后节点
        final Node<E> prev = x.prev;    //删除元素的前节点

        if (prev == null) {
            first = next;   //这种情况就是x为链表的头元素，prev为空，直接将后节点设置为链表的第一位
        } else {
            prev.next = next;   //将x的前节点，的next设置为x的后节点
            x.prev = null;  //将x的prev设置为null,help gc
        }

        if (next == null) {
            last = prev; //这种情况就是x为链表尾元素，next为空，直接将前节点设置为链表的最后一位
        } else {
            next.prev = prev;   //将x的尾节点，的prev设置x的前节点
            x.next = null;  //将x的后节点设置为null,help gc
        }

        x.item = null;  //将x的内容设置为Null,help gc
        size--; //链表长度-1
        modCount++;
        return element; //返回删除的元素内容
    }

``` 

这里无非也是改下引用关系，注释写的很清楚了。。   




***  
##### 5.get(int index)  
查询方法  
```
  public E get(int index) {
        checkElementIndex(index);   //检查索引下标是否合法，不是重点
        return node(index).item;    //遍历node链表，找到对应的索引的内容
    }

```  

查询方法直接调用的之前提到的node(index)，比起ArrayList的get(时间复杂度o(1))，这里的get的时间复杂度是o(n)

  











