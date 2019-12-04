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

红黑树概念理解的相关文章**强烈推荐新人入门**：
https://www.jianshu.com/p/b7dda385f83d


***  

在贴代码之前，我讲一下红黑树基本概念理解，并且画了一些图来描述：  
![image](https://wx1.sinaimg.cn/mw1024/007R8l8Fly1g9ggy00xrbj30vd0tjwfo.jpg)  


![image](https://wx1.sinaimg.cn/mw1024/007R8l8Fly1g9ggy02hvfj30u00x5jtw.jpg) 


![image](https://wx4.sinaimg.cn/mw1024/007R8l8Fly1g9ggy01llvj30nx0twq4h.jpg)  



![image](https://wx3.sinaimg.cn/mw1024/007R8l8Fly1g9ggy01z52j30sz0tf76j.jpg)


***
下面就是链表转红黑树的代码了，相当地难啃  
- treeifyBin的逻辑  
``` 
  /**
     * Replaces all linked nodes in bin at index for given hash unless
     * table is too small, in which case resizes instead.
     */
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;  //初始化一些变量
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY) //数组为空或者map的内容<64
            resize();   //先扩容
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null; //红黑树转换逻辑，hd代表头，tl代表尾
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null); //创建一个新的TreeNode节点,e.hash,e.key,e.value,next=null
                if (tl == null) //尾巴节点为空，证明还没有根节点
                    hd = p; //p指向根节点
                else {
                    p.prev = tl;    //指定p的前节点是tl
                    tl.next = p;    //指定tl的后节点是p，这两步就形成了一个双向链表
                }
                tl = p; //令p指向tl
            } while ((e = e.next) != null); //这个do...while的目的是将链表的数据结构转成红黑树的TreeNode
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```  

**这里我们可以看到，它做的事情是将链表转换成一个双向链表，方便我们下面的操作**     

***
再下面就是红黑树的转换逻辑，我们可以看到**hd.treeify(tab)**方法，这个方法的作用是，将双向链表转换成红黑树。  

``` 
 /**
         * Forms tree of the nodes linked from this node.
         * @return root of tree 红黑树的转换逻辑
         */
        final void treeify(Node<K,V>[] tab) {
            TreeNode<K,V> root = null;  //初始化root指针
            for (TreeNode<K,V> x = this, next; x != null; x = next) {   //遍历循环node链表
                next = (TreeNode<K,V>)x.next;   //找到x的next节点
                x.left = x.right = null;
                if (root == null) {
                    x.parent = null;
                    x.red = false;
                    root = x;   //首次循环的时候，将链表的头元素设置成红黑树的root根节点
                }
                else {
                    K k = x.key;    //红黑树节点的key
                    int h = x.hash; //红黑树节点的hash
                    Class<?> kc = null;
                    for (TreeNode<K,V> p = root;;) {    //来个死循环，并且令p=根节点root
                        int dir, ph;    //初始化变量
                        K pk = p.key;
                        if ((ph = p.hash) > h)
                            dir = -1;   //root节点的hash 大于 红黑树节点hash，dir=-1
                        else if (ph < h)
                            dir = 1;     //root节点的hash 小于 红黑树节点hash，dir=1
                        else if ((kc == null &&
                                  (kc = comparableClassFor(k)) == null) ||
                                 (dir = compareComparables(kc, k, pk)) == 0)    //看红黑树节点的key有没有实现compareable接口
                            dir = tieBreakOrder(k, pk); //没有实现compareable接口进入这个逻辑，由native底层方法返回-1或者1，为了分辨节点是在root右边还是左边

                        TreeNode<K,V> xp = p;   //令p = xp
                        if ((p = (dir <= 0) ? p.left : p.right) == null) {  //dir<0，p=p.left;否则p=p.right
                            x.parent = xp;
                            if (dir <= 0)
                                xp.left = x;    //小于0，x就在xp的左边
                            else
                                xp.right = x;   //大于0，x就在xp的右边
                            root = balanceInsertion(root, x);   //调整红黑树的颜色
                            break;
                        }
                    }
                }
            }
            moveRootToFront(tab, root);
        }

```  

**这里就是循环双向链表，然后用hash值进行比较，hash值比root小就在root左边，否则在右边，之后就调用的balanceInsertion(root, x)方法，这个方法的作用无非就是修复红黑树的结构，根据红黑树的特点，进行变色，左旋，右旋的操作，之后再调用moveRootToFront方法，来调整根节点**  

- balanceInsertion方法
```
    static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {  //注意，这个方法返回的是根节点
            x.red = true;   //新增的节点默认是红色
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) { //初始化一些变量，xp父节点，xpp祖父节点，xppl祖父左节点 xppr祖父右节点
                if ((xp = x.parent) == null) {  //令xp=x的父节点，假设xp为空
                    x.red = false;  //x就是根节点，且为黑色，直接返回
                    return x;
                }
                else if (!xp.red || (xpp = xp.parent) == null)
                    return root;    //假设父节点为黑色，而且祖父节点为空的情况下，直接返回root
                if (xp == (xppl = xpp.left)) {  //左子树的情况
                    if ((xppr = xpp.right) != null && xppr.red) { //x节点的叔叔节点不为空且是红色的情况下，走变色的逻辑
                        xppr.red = false;   //将叔叔节点设置成黑色
                        xp.red = false; //将父亲节点设置成黑色
                        xpp.red = true; //将祖父节点设置成红色
                        x = xpp;    //将当前节点设置成祖父节点
                    }
                    else {
                        if (x == xp.right) {  //x节点 的叔叔节点为空或者是黑色的情况下，走左旋的逻辑
                            root = rotateLeft(root, x = xp);    //注意，这里的旋转节点是xp(x的父亲节点)
                            xpp = (xp = x.parent) == null ? null : xp.parent;   //xpp = x的父亲为空？null:x父亲的父亲
                        }
                        //右旋的逻辑是，将父节点变成黑色，将祖父节点变成红色，以祖父节点开始旋转
                        if (xp != null) {   //x的父亲不为空
                            xp.red = false; //将x的父亲涂黑
                            if (xpp != null) {  //xpp不为空的情况
                                xpp.red = true; //将祖父xpp涂红
                                root = rotateRight(root, xpp);  //以祖父节点开始右旋
                            }
                        }
                    }
                }
                else {  //右子树的的情况
                    if (xppl != null && xppl.red) {
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;    //这4行统一是变色的逻辑
                    }
                    else {
                        if (x == xp.left) {
                            root = rotateRight(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
        }
```  

然后底层调用了左旋和右旋的方法，所以还要看看这两个方法  

- 左旋  
```

        /* ------------------------------------------------------------ */
        // Red-black tree methods, all adapted from CLR

        static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {    //这里需要注意的是p是新增元素的父节点，新增元素用x表示
            TreeNode<K,V> r, pp, rl;    //r为左旋上来的节点，pp为x的祖父节点，p的父亲节点，rl为父亲节点的右子树的左子树
            if (p != null && (r = p.right) != null) {   //假设说父节点不为空 且 父节点的右子树不为空的情况下,r=父节点的右子树
                if ((rl = p.right = r.left) != null)   //rl=p.right=r.left的逻辑可能有点难理解，这么说把，就是下面这个情况,B代表黑色，R代表红色，
                                                        // 12的左子树是7B，左旋后，7B变成了5R的右子树，并且把7B的parent设置成5R
                                                    //，
                    rl.parent = p;                  //          5 R                                     12R
                                                    //  1 B                12R                  5R
                                                    //                 7B      13B  ====>   1B      7B         13B
                                                    //
                if ((pp = r.parent = p.parent) == null)
                    (root = r).red = false; //假设左旋后,r的父亲为空，那么证明r就是根节点，这时候将根节点设置成黑色
                else if (pp.left == p)  //假设pp祖父节点的左子树是p的情况
                    pp.left = r;    //r的父亲不为空的情况，将r设置成祖父的左子树（原本左子树是p，但是左旋后p下去了）
                else
                    pp.right = r;   //pp祖父节点的右子树是p的情况
                r.left = p; //if..elseif..else结束后，将左旋上来的节点r的左子树设置成左旋下去的节点p
                p.parent = r;   //左旋下去的节点p的父亲设置成r
            }
            return root;    //最后返回root节点
        }
```  

- 右旋 
```   
        static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {   //这里的p代表的是祖父节点
            TreeNode<K,V> l, pp, lr;   //l是祖父节点的左子树，右旋转后上来的顶节点，如下图的12B,pp是p的父节点（可能为空）,lr是原本祖父节点p的子树的右子树（13B），
            if (p != null && (l = p.left) != null) {
                if ((lr = p.left = l.right) != null)    //画图说明吧。。其实道理和左旋是一样的，现在的情况就是13B，原本是l.right，现在移到19R的left去了
                    //              19R                             12B
                    //      12B            30B ====>        5R              19R
                    //  5R      13B
                    //                                                  13B
                    //
                    lr.parent = p;
                if ((pp = l.parent = p.parent) == null)
                    (root = l).red = false; //如果右旋转后，祖父节点p的父亲为空，证明订单就是l节点，那么root=l，并且设置成黑色
                else if (pp.right == p)
                    pp.right = l;   //p原本假设在右子树的话，那么pp的右边就设置成l点
                else
                    pp.left = l;    //p原本假设在左子树的话，那么pp左边就设置为l点
                l.right = p;    //设置l的右边为p
                p.parent = l;   //p的父亲为l
            }
            return root;
        }

```  

左旋和右旋的话结合下我上面贴的图理解或许会更好。  



- moveRootToFront方法  

```
     /**
         * Ensures that the given root is the first node of its bin.，
         */
        static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {    //把给定节点设置为桶中的第一个元素
            int n;
            if (root != null && tab != null && (n = tab.length) > 0) {
                int index = (n - 1) & root.hash;
                TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
                if (root != first) {    //如果root不是第一个节点，则将root放到第一个首节点位置
                    Node<K,V> rn;
                    tab[index] = root;
                    TreeNode<K,V> rp = root.prev;
                    if ((rn = root.next) != null)
                        ((TreeNode<K,V>)rn).prev = rp;
                    if (rp != null)
                        rp.next = rn;
                    if (first != null)
                        first.prev = root;
                    root.next = first;
                    root.prev = null;
                }   //这里面的操作就是把root放到桶的第一个元素上去
                assert checkInvariants(root);   //这里是防御性编程，校验更改后的结构是否满足红黑树和双链表的特性.因为HashMap并没有做并发安全处理，可能在并发场景中意外破坏了结构
            }
        }
```

这个方法的作用主要是保证我们的根节点是在数组位桶的第一个  

***

自己整了一下流程图：  
![image](https://wx1.sinaimg.cn/mw1024/007R8l8Fgy1g9krwubivnj30u025jaiw.jpg)


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




















