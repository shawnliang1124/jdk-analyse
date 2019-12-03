### HashMap的源码分析   

关于HashMap的源码，自己已经学习了不长时间，每段时间自己都会有新的不同的理解。   
Anyway，人本身就是一个不断学习的过程，折腾起来。  

##### 一、HashMap的特性   
从简单的代码写起：  
```  
    HashMap<Object, Object> map = new HashMap<>();
        map.put("liuyi", "刘一");
        map.put("wanger", "王二");
        map.put("zhangsan", "张三");
        map.put("lisi", "李四");
        map.put("wangwu", "王五");
        map.put("zhaoliu", "赵六");
        map.put("chenqi", "陈七");
        map.put("yangba", "杨八");
        map.put("zhoujiu", "周九");
        map.forEach( (k ,v )->{
            System.out.println("key is "+ k + " and value is "+v);
        });
    }  
    
//输出的结果  
key is lisi and value is 李四
key is zhoujiu and value is 周九
key is yangba and value is 杨八
key is zhaoliu and value is 赵六
key is zhangsan and value is 张三
key is wangwu and value is 王五
key is liuyi and value is 刘一
key is wanger and value is 王二
key is chenqi and value is 陈七

```    

这里看到输出的结果是无序的。不仅如此，==我还把hashmap一些通用的特性展示出来了。==  
- 实现了Map接口，里面的方法全部被HashMap实现。  
- 允许null键和null值。这点和Hashtable不同，Hashtable不允许。  
- 非线程安全，这点也和Hashtable不同，Hashtable为线程安全的。  
- 不保证有序，即元素的插入和读取顺序不保证一致，这一点从开始的示例程序和图就能看出来。  
- 也不保证元素的顺序始终不变，在扩容时，元素的顺序会重新被打乱。  
 
***  

##### 二、HashMap的数据结构

从debug开始了解数据结构，如图所示：  

![image](https://upload-images.jianshu.io/upload_images/7432604-6a836ae7704f9ab1.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)  

**实际上底层的值是存放在table指针里面，也就是核心的Node<K,V>数组**  

**来看下Node<K,V>的数据结构：**  
```
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next; 
    
    ...
}
        
```  

  
**这么就一目了然了，HashMap内部用一个table（Entry元素数组）来存储元素，索引中的每个元素又采用链表的结构进行存储，用于存储相同索引的元素。其中Entry类就是一个节点类，具有链接下一个节点的功能，内部存储了元素的key,value,hash,next引用，Entry的结构如上述代码，采用内部类的方式实现。**   

另外，HashMap会有以下几个关键属性，面试也是常常会问到：   

![image](https://wx3.sinaimg.cn/mw1024/007R8l8Fgy1g9dqs2g3ktj30nr0fzdu4.jpg)  

先做一个伏笔，待会详细进行说明。  
-  DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16  table数组默认大小    
-  DEFAULT_LOAD_FACTOR = 0.75f //负载因子
- TREEIFY_THRESHOLD = 8; // 红黑树转换阈值

*****

##### 三.代码分析  

从put方法开始分析：  
```   
 public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```   

**可以看到，先使用了hash运算，算了key的hash，并且调用hashMap底层的putVal方法()**  

**下面，就进入hash(key)方法：**    
```   
  static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```  

这个方法流程不复杂，key为null，hash计算就是0（这就解释了为啥==Key可以为Null==）,否则就调用
==(h = key.hashCode()) ^ (h >>> 16)== 算法

**(h = key.hashCode()) ^ (h >>> 16)**  我把它理解成一个算法，右移16位之后再进行异或运算，目的是进一步<font color="red">**降低hash冲突**</font>的概率。   

hash算法参考资料：  
https://www.cnblogs.com/wang-meng/p/9b6c35c4b2ef7e5b398db9211733292d.html   

好了，计算完hash值以后，继续调用putVal()方法。。   

**putVal(..)方法如下:**   
```   
    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;     //初始化一些变量
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;          //table为空，走resize()扩容逻辑，并且返回扩容后大小
        if ((p = tab[i = (n - 1) & hash]) == null)      //(n-1)& hash 理解为取余算法定位table数组的下标
            tab[i] = newNode(hash, key, value, null);       //在数组对应的下标创建新的节点对象
        else {      //走到这一步说明数组对应的下标已经有值了
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;  //传入的key和数组该下标的key相等！将p指向e局部变量，这里的p是原数组下标的值值值！！,e!=null(PS 这里就是key键相等，走替换的逻辑，替换的逻辑在下面)
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); //p是红黑树，走putTreeVal逻辑
            else {  //链表的逻辑
                for (int binCount = 0; ; ++binCount) {  //死循环，遍历链表，e为链表下一个节点，p为当前链表节点
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);    //当前节点的链表next节点为空，直接创建节点
                        if (binCount >= TREEIFY_THRESHOLD - 1){ // -1 for 1st
                            treeifyBin(tab, hash);  }//大于8就直接转换成红黑树
                        break;  //创建完就直接跳出循环，这里的逻辑e==null！！！
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break; //假设上一步没break出来，会进入这个逻辑，在链表中找到key键相等，
                    p = e; // e指向p，进行下一个链表节点的遍历
                }
            }
            if (e != null) { // 存在即覆盖的逻辑
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;    //将新的值覆盖旧的值
                afterNodeAccess(e); //先不管，功能是回调以允许LinkedHashMap后置操作
                return oldValue;    //返回老的value值
            }
        }
        ++modCount; //操作次数++
        if (++size > threshold)
            resize();   //如果size大小(key-value对的大小)大于临界值则要进行resize()
        afterNodeInsertion(evict);  //回调以允许LinkedHashMap后置操作
        return null;    //创建新的节点返回Null
    }
```    


流程图：  

![image](https://wx3.sinaimg.cn/mw1024/007R8l8Fgy1g9dvqu5i0vj30u01hqwhy.jpg)  

以上的put方法进行详解：   
- 取余算法它是tab[i = (n-1)& hash]，这里面也有为什么Hashmap默认长度是16的原因。   
使用它的原因：  https://www.cnblogs.com/ysocean/p/9054804.html   


- 涉及红黑树的代码：  
==putTreeVal()==  
==treeifyBin()==   
这里面的代码后续会分析  

- resize() (扩容代码)   
这里面的代码后续会分析  

流程图应该讲的很详细了，底层是一个Node数组，发生hash冲突的时候，会在对应的数组索引上往后新增链表节点，当链表节点的长度超过8的时候会转换成红黑树。  

***  


resize()方法的分析：   
```  
 final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;     //将老的Node数组放到oldTab指针上
        int oldCap = (oldTab == null) ? 0 : oldTab.length;  //获得老的Node数组的长度
        int oldThr = threshold; //老的阈值
        int newCap, newThr = 0; //定义新的Node数组和新的阈值
        if (oldCap > 0) {   //证明老的Node不为空
            if (oldCap >= MAXIMUM_CAPACITY) {   //大于最大容量直接返回了，不能继续扩容
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold   //小于最大容量并且老的容量大于16，扩容2倍
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;     //老的阈值大于0，新的Node数组大小等于老的阈值
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;  //默认的无参构造，Node大小为16
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);     //新的阈值为16*0.75=12
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;  //计算指定了无参构造传入负载因子，新的扩容阈值
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr; //将方法的局部变量赋予全局变量
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; //创建新的Node数组，并且指定新的数组大小
        table = newTab;     //把新的Node数组执行全局的table指针
        if (oldTab != null) {   //旧的Node数组不为空逻辑
            for (int j = 0; j < oldCap; ++j) {  //遍历旧的Node数组，将旧数组(包括链表红黑树)的值复制到新的数组上
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;   //将旧的数组对应下标指向null，方便GC回收
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;  //如果旧数组索引上链表的next为空，通过hash&(length-1)算法重新找到新数组的位置
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);  //红黑树。。。再说吧
                    else { // preserve order        //链表的重新赋值
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {    // do..while目的是遍历数组索引桶中的链表,这里就重点分析
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

```

链表移动的分析:   
loHead, loTail ,hiHead , hiTail，这四个变量从字面意思可以看出应该是两个头节点，两个尾节点。那么为什么需要两个链表的头尾节点呢？看一张图就明白了：   

![image](https://img-blog.csdnimg.cn/20190621145827456.png)  

这张图中index=2的桶中有四个节点，在未扩容之前，它们的 hash& cap 都等于2。在扩容之后，它们之中2、18还在一起，10、26却换了一个桶。这就是这句代码的含义：选择出扩容后在同一个桶中的节点。   


```  
 if ((e.hash & oldCap) == 0)
```  

我们这时候的oldCap = 8，2的二进制为：0010，8的二进制为：1000，0010 & 1000 =0000
10的二进制为：1010，1010 & 1000 = 1000，
18的二进制为：10010, 10010 & 1000 = 0000，
26的二进制为：11010，11010 & 1000 = 1000，
从与操作后的结果可以看出来，2和18应该在同一个桶中，10和26应该在同一个桶中。

所以lo和hi这两个链表的作用就是保存原链表拆分成的两个链表。



*** 
红黑树代码分析：   
想要了解HashMap底层的红黑树逻辑，我们首先要掌握红黑树相关的概念，这里我已经列举出来了。。  






***

上述讲的是HashMap的1.8原理，下面讲以下1.7的原理吧，这里1.7没有仔细研究，就贴下上课的分析吧。。    

**想请问一下，hashmap1.7的hash计算有什么问题？**  
1、首先，1.7的hashmap底层如果发生Hash冲突了，就是链表，而1.8使用了红黑树区域解决链表查询的问题 。  


2.HashMap1.7中的hash计算的算法和1.8的算法也是不一样的，
1.7用的是抖动算法，通过4次抖动，减少hash冲突。看下源码，hash(..)就行。  

3.HashMap中加载因子为啥是0.75？   
0.75是比较一个适中的数值。  
假设负载因子过大，时间复杂度会增加，空间复杂度会减少。  假设负载因子过小，时间复杂度会减少，空间复杂度会增加。  

4.为什么说hashmap1.7扩容会发生死循环的问题？   
1.7版本的resize扩容方法的时候使用扩容会发生死循环，产生的原因如下：   
**面试就这么说把，因为JDK1.7的HashMap使用的是链表头插赋值法，在多线程的情况下回导致一个死循环的问题**。下面是具体分析！！  

首先，resize方法下面是一个transfer方法，该方法的作用是，重新移动老数组的元素到新数组里面去，以下是transfer方法的部分代码：   
```   
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
           e.next = newTable[i];    //重点关注
            newTable[i] = e;
            e = next;
        }
    }
}

```  

下面，我们重点关注，
==e.next = newTable[i]== 这一行的代码。  
为什么说这行代码会产生死循环的问题呢？   

![image](https://wx3.sinaimg.cn/mw1024/007R8l8Fly1g9hoa1fn66j31aq0u0tb9.jpg)  


![image](https://wx2.sinaimg.cn/mw1024/007R8l8Fly1g9hoa1exknj31h80mo75m.jpg)  



![image](https://wx4.sinaimg.cn/mw1024/007R8l8Fly1g9hoa1f1d3j31dw0ldjsq.jpg)  


![image](https://wx2.sinaimg.cn/mw1024/007R8l8Fly1g9hoa1g7efj319n0qa759.jpg)




***
HashMap的源码分析比较好的文章：  
https://blog.csdn.net/weixin_41565013/article/details/93190786  




















