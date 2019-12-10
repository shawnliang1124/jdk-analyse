##### 一、数据结构   
关于ConcurrentHashMap(下面简称CHM)，在1.8做了很大的变化，相对于1.7来说，取消了分段锁的概念，使用synchornized关键字对Node节点进行上锁，实现线程安全的操作。CHM1.8底层的数据结构和HashMap1.8实际上是一样的，统一的也是**数组+链表+红黑树**。发生hash冲突的时候，会在tab[i]下面生成一个单向链表。   


##### 二、源码分析    
下面直接就进入源码分析，从put开始：   

```
 public V put(K key, V value) {
        return putVal(key, value, false);
    }
```  

下面再看看putVal的详解：   

```
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());  //hash算法生成hash值
        int binCount = 0;       //变量记录链表的长度
        for (Node<K,V>[] tab = table;;) {   //自选操作，关注break的条件
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();  //数组为空，就初始化数组
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,  //数组下标tab[i]为空，cas给tab[i]赋值，为什么不使用tab[i]的原因，是因为考虑到多线程的情况
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount); //值赋值后，可能涉及到数组的扩容，
        return null;
    }

```  

这个put方法可以分为以下几个逻辑：   
1 数组为空：初始化数组  
2 数组不为空，但是数组上的内容为空  
3 hash为-1  
4 数组上有值，进行链表覆盖或者新增  
5 完成以上4个步骤后，调用 addCount(1L, binCount);方法，将CHM的size+1,或者涉及到相关的扩容。   

**注意：CHM的put的逻辑其实和HashMap是很像的，整体的流程也是先赋值再扩容，并且里面的流程也是基本相同**    






###### 1.initTable方法，table为空会调用这个方法初始化：  
sizeCtl 这个要单独说一下，如果没搞懂这个属性的意义，可能会被搞晕
这个标志是在 Node 数组初始化或者扩容的时候的一个控制位标识，负数代表正在进行初始
化或者扩容操作
-1 代表正在初始化  
-N 代表有 N-1 有二个线程正在进行扩容操作，这里不是简单的理解成 n 个线程，sizeCtl 就
是-N,这块后续在讲扩容的时候会说明  
0 标识 Node 数组还没有被初始化，正数代表初始化或者下一次扩容的大小  

```
   private final Node<K,V>[] initTable() { //sizeCtl理解为一个标志位即可,0代表没有被初始化，负数代表正在进行扩容的操作
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0) //sizeCtl<0，代表别的线程抢占了table数组的初始化，那么请yelid让出cpu线程
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {  //将SIZECTL标识置为-1，标识当前线程抢到了初始化的资格
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2); //这个if里面的操作有,初始化了数组的长度，并且sc = 0.75*sc，即下次数组要扩容的长度
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;  //跳出while循环
            }
        }
        return tab;
    }
```  

###### 2.数组不为空，但是数组上的内容为空  
```
  else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,  //数组下标tab[i]为空，cas给tab[i]赋值，为什么不使用tab[i]的原因，是因为考虑到多线程的情况
                        new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
```  
使用tab[i]赋值，在多线程的环境下会有相关的风险，CAS操作保证线程安全性  



###### 3.节点的hash值为-1:  
```
    else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
```  

###### 3.节点的hash值为-1: 稍后看


######  4.数组上有值，进行链表覆盖或者新增:稍后看  


因为第一次put的时候肯定是分别走1,2的逻辑，3和4待会再看，之后就到**addCount(1L, binCount)**;    

addCount方法分析：  

```
private final void addCount(long x, int check) {
CounterCell[] as; long b, s;
判断 counterCells 是否为空，
1. 如果为空，就通过 cas 操作尝试修改 baseCount 变量，对这个变量进行原子累加操
作(做这个操作的意义是：如果在没有竞争的情况下，仍然采用 baseCount 来记录元素个
数)
2. 如果 cas 失败说明存在竞争，这个时候不能再采用 baseCount 来累加，而是通过
CounterCell 来记录
if ((as = counterCells) != null ||
!U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x))
{
CounterCell a; long v; int m;
boolean uncontended = true;//是否冲突标识，默认为没有冲突
这里有几个判断
1. 计数表为空则直接调用 fullAddCount
2. 从计数表中随机取出一个数组的位置为空，直接调用 fullAddCount
3. 通过 CAS 修改 CounterCell 随机位置的值，如果修改失败说明出现并发情况（这里又
用到了一种巧妙的方法），调用 fullAndCount
Random 在线程并发的时候会有性能问题以及可能会产生相同的随机
数 ,ThreadLocalRandom.getProbe 可以解决这个问题，并且性能要比 Random 高
if (as == null || (m = as.length - 1) < 0 ||
(a = as[ThreadLocalRandom.getProbe() & m]) == null ||

!(uncontended =
U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
fullAddCount(x, uncontended);//执行 fullAddCount 方法
return;
}
if (check <= 1)//链表长度小于等于 1，不需要考虑扩容
return;
s = sumCount();//统计 ConcurrentHashMap 元素个数
}
//….
}

```  

CounterCells数据结构的分析：  

ConcurrentHashMap 是采用 CounterCell 数组来记录元素个数的，像一般的集合记录集合大
小，直接定义一个 size 的成员变量即可，当出现改变的时候只要更新这个变量就行。为什么
ConcurrentHashMap 要用这种形式来处理呢？
问题还是处在并发上，ConcurrentHashMap 是并发集合，如果用一个成员变量来统计元素个
数的话，为了保证并发情况下共享变量的的难全兴，势必会需要通过加锁或者自旋来实现，
如果竞争比较激烈的情况下，size 的设置上会出现比较大的冲突反而影响了性能，所以在
ConcurrentHashMap 采用了分片的方法来记录大小，具体什么意思，我们来分析    



```
private transient volatile int cellsBusy;// 标识当前 cell 数组是否在初始化或扩容中的
CAS 标志位
/**
* Table of counter cells. When non-null, size is a power of 2.
*/
private transient volatile CounterCell[] counterCells;// counterCells 数组，总数
值的分值分别存在每个 cell 中
@sun.misc.Contended static final class CounterCell {
volatile long value;
CounterCell(long x) { value = x; }
```   


我们看看CHM的size方法:  

```
 public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }  
    
    
```  

再看看sumCount方法：看到这段代码就能够明白了，CounterCell 数组的每个元素，都存储一个元素个数，而实际我们调用
size 方法就是通过这个循环累加来得到的  

```
  final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```  


所以，我们回到addCount方法里面，看一下**fullAddCount**方法：  
主要是用来初始化 CounterCell，来记录元素个数，里面包含扩容，初始化等
操作  

```
 private final void fullAddCount(long x, boolean wasUncontended) {
        int h;
//获取当前线程的 probe 的值，如果值为 0，则初始化当前线程的 probe 的值,probe 就是随机数
        if ((h = ThreadLocalRandom.getProbe()) == 0) {
            ThreadLocalRandom.localInit(); // force initialization
            h = ThreadLocalRandom.getProbe();
            wasUncontended = true; // 由于重新生成了 probe，未冲突标志位设置为 true
        }
        boolean collide = false; // True if last slot nonempty
        for (; ; ) {//自旋
            CounterCell[] as;
            CounterCell a;
            int n;
            long v;
//说明 counterCells 已经被初始化过了，我们先跳过这个代码，先看初始化部分
            if ((as = counterCells) != null && (n = as.length) > 0) {
                if ((a = as[(n - 1) & h]) == null) {// 通过该值与当前线程 probe 求与，获得
                    cells 的下标元素，和 hash 表获取索引是一样的
                    if (cellsBusy == 0) { //cellsBusy=0 表示 counterCells 不在初始化或者扩
                        容状态下
                        CounterCell r = new CounterCell(x); //构造一个 CounterCell 的值，
                        传入元素个数
                        if (cellsBusy == 0 &&//通过cas设置cellsBusy标识，防止其他线程来  对 counterCells 并发处理
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)){
                            boolean created = false;
                            try { // Recheck under lock
                                CounterCell[] rs;
                                int m, j;
//将初始化的 r 对象的元素个数放在对应下标的位置
                                if ((rs = counterCells) != null &&
                                        (m = rs.length) > 0 &&
                                        rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {//恢复标志位
                                cellsBusy = 0;
                            }
                            if (created)//创建成功，退出循环
                                break;
                            continue;//说明指定 cells 下标位置的数据不为空，则进行下一次循环
                        }
                    }
                    collide = false;
                }
//说明在 addCount 方法中 cas 失败了，并且获取 probe 的值不为空
                else if (!wasUncontended) // CAS already known to fail
                    wasUncontended = true; // 设置为未冲突标识，进入下一次自旋
// 由于指定下标位置的 cell 值不为空，则直接通过 cas 进行原子累加，如果成功，则直接
                退出
else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))//
                    break;
//如果已经有其他线程建立了新的 counterCells 或者 CounterCells 大于 CPU 核心数
（很巧妙，线程的并发数不会超过 cpu 核心数）
else if (counterCells != as || n >= NCPU)
                    collide = false; //设置当前线程的循环失败不进行扩容
                else if (!collide)//恢复 collide 状态，标识下次循环会进行扩容
                    collide = true;
//进入这个步骤，说明 CounterCell 数组容量不够，线程竞争较大，所以先设置一个标识
                表示为正在扩容
else if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                    try {
                        if (counterCells == as) {// Expand table unless stale
// 扩容一倍 2 变成 4 ，这个扩容比较简单
                            CounterCell[] rs = new CounterCell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            counterCells = rs;
                        }
                    } finally {
                        cellsBusy = 0;//恢复标识
                    }
                    collide = false;
                    continue;// 继续下一次自旋
                }
                h = ThreadLocalRandom.advanceProbe(h);//更新随机数的值
            }
            初始化 CounterCells 数组
//cellsBusy=0 表示没有在做初始化，通过 cas 更新 cellsbusy 的值标注当前线程正在做初
                    始化操作
else if (cellsBusy == 0 && counterCells == as &&
                    U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                boolean init = false;
                try { // Initialize table
                    if (counterCells == as) {
                        CounterCell[] rs = new CounterCell[2]; //初始化容量为 2
                        rs[h & 1] = new CounterCell(x);//将 x 也就是元素的个数放在指定的数组
                        下标位置
                                counterCells = rs;//赋值给 counterCells
                        init = true;//设置初始化完成标识
                    }
                } finally {
                    cellsBusy = 0;//恢复标识
                }
                if (init)
                    break;
            }
//竞争激烈，其它线程占据 cell 数组，直接累加在 base 变量中
            else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                break; // Fall back on using base
        }
    }

```   

代码这么长，看的有点累，直接看CountersCell初始化图解就好了：   
![image](https://wxt.sinaimg.cn/mw1024/007R8l8Fly1g9p930rlfuj308e0433yc.jpg?tags=%5B%5D)    

**首先会有个全局变量baseCount，假设cas失败的话，证明存在并发的扩容
这里就会初始化CounterCell，并且利用ThreadLocalRandom生成一个随机数，并且进行取模算法(与运算)，并且赋值给CounterCell数组里面。如果CounterCell有值的话就+1好了**   

###### CHM的扩容逻辑：  
判断是否需要扩容，也就是当更新后的键值对总数 baseCount >= 阈值 sizeCtl 时，进行
rehash，这里面会有两个逻辑。
1. 如果当前正在处于扩容阶段，则当前线程会加入并且协助扩容
2. 如果当前没有在扩容，则直接触发扩容操作  
```
if (check >= 0) {//如果 binCount>=0，标识需要检查扩容
Node<K,V>[] tab, nt; int n, sc;
//s 标识集合大小，如果集合大小大于或等于扩容阈值（默认值的 0.75）
//并且 table 不为空并且 table 的长度小于最大容量
while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
(n = tab.length) < MAXIMUM_CAPACITY) {
int rs = resizeStamp(n);//这里是生成一个唯一的扩容戳，这个是干嘛用的
呢？且听我慢慢分析
if (sc < 0) {//sc<0，也就是 sizeCtl<0，说明已经有别的线程正在扩容了
//这 5 个条件只要有一个条件为 true，说明当前线程不能帮助进行此次的扩容，
直接跳出循环
//sc >>> RESIZE_STAMP_SHIFT!=rs 表示比较高 RESIZE_STAMP_BITS 位
生成戳和 rs 是否相等，相同
//sc=rs+1 表示扩容结束
//sc==rs+MAX_RESIZERS 表示帮助线程线程已经达到最大值了
//nt=nextTable -> 表示扩容已经结束
//transferIndex<=0 表示所有的 transfer 任务都被领取完了，没有剩余的
hash 桶来给自己自己好这个线程来做 transfer
if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
transferIndex <= 0)
break;
if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))//当前线程

尝试帮助此次扩容，如果成功，则调用 transfer
transfer(tab, nt);
}
// 如果当前没有在扩容，那么 rs 肯定是一个正数，通过 rs<<RESIZE_STAMP_SHIFT 将 sc 设置
为一个负数，+2 表示有一个线程在执行扩容
else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs <<
RESIZE_STAMP_SHIFT) + 2))
transfer(tab, null);
s = sumCount();// 重新计数，判断是否需要开启下一轮扩容
}
}
```  


###### resizeStamp  

这块逻辑要理解起来，也有一点复杂。
resizeStamp 用来生成一个和扩容有关的扩容戳，具体有什么作用呢？我们基于它的实现来
做一个分析
static final int resizeStamp(int n) {
return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
Integer.numberOfLeadingZeros 这个方法是返回无符号整数 n 最高位非 0 位前面的 0 的个
数  
比如 10 的二进制是 0000 0000 0000 0000 0000 0000 0000 1010  
那么这个方法返回的值就是 28    

根据 resizeStamp 的运算逻辑，我们来推演一下，假如 n=16，那么 resizeStamp(16)=32796
转化为二进制是  
[0000 0000 0000 0000 1000 0000 0001 1100]  
接着再来看,当第一个线程尝试进行扩容的时候，会执行下面这段代码
U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2)  
rs 左移 16 位，相当于原本的二进制低位变成了高位 1000 0000 0001 1100 0000 0000 0000
0000
然后再+2 =1000 0000 0001 1100 0000 0000 0000 0000+10=1000 0000 0001 1100 0000 0000
0000 0010。   

**高 16 位代表扩容的标记、低 16 位代表并行扩容的线程数**

1. 首先在 CHM 中是支持并发扩容的，也就是说如果当前的数组需要进行扩容操作，可以
由多个线程来共同负责，这块后续会单独讲
2. 可以保证每次扩容都生成唯一的生成戳，每次新的扩容，都有一个不同的 n，这个生成
戳就是根据 n 来计算出来的一个数字，n 不同，这个数字也不同
➢ 第一个线程尝试扩容的时候，为什么是+2
因为 1 表示初始化，2 表示一个线程在执行扩容，而且对 sizeCtl 的操作都是基于位运算的，
所以不会关心它本身的数值是多少，只关心它在二进制上的数值，而 sc + 1 会在
低 16 位上加 1。   


**transfer**  
transfer就是扩容的意思，是 ConcurrentHashMap 的精华之一，这里先说明，CHM的扩容是支持**多线程一起扩容的**。  
使用CAS无锁机制，实现同步。  
**实现原理**：简单来说，它把 Node 数组当作多个线程之间共享的任务队列，然后通过维护一个指针来划
分每个线程锁负责的区间，每个线程通过区间逆向遍历来实现扩容，一个已经迁移完的
bucket会被替换为一个ForwardingNode节点，标记当前bucket已经被其他线程迁移完了。  

接下来分析一下它的源码实现
1、fwd:这个类是个标识类，用于指向新表用的，其他线程遇到这个类会主动跳过这个类，因
为这个类要么就是扩容迁移正在进行，要么就是已经完成扩容迁移，也就是这个类要保证线
程安全，再进行操作。  
2、advance:这个变量是用于提示代码是否进行推进处理，也就是当前桶处理完，处理下一个
桶的标识  
3、finishing:这个变量用于提示扩容是否结束用的  

``` 

 private final void transfer(Node<K, V>[] tab, Node<K, V>[] nextTab) {
        int n = tab.length, stride;
//将 (n>>>3 相当于 n/8) 然后除以 CPU 核心数。如果得到的结果小于 16，那么就使用 16
// 这里的目的是让每个 CPU 处理的桶一样多，避免出现转移任务不均匀的现象，如果桶较少
        的话，默认一个 CPU（一个线程）处理 16 个桶，也就是长度为 16 的时候，扩容的时候只会有一
                个线程来扩容
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) <
                MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
//nextTab 未初始化， nextTab 是用来扩容的 node 数组
        if (nextTab == null) { // initiating
            try {
                @SuppressWarnings("unchecked")
//新建一个 n<<1 原始 table 大小的 nextTab,也就是 32
                        Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n << 1];
                nextTab = nt;//赋值给 nextTab
            } catch (Throwable ex) { // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE; //扩容失败，sizeCtl 使用 int 的最大值
                return;
            }
            nextTable = nextTab; //更新成员变量
            transferIndex = n;//更新转移下标，表示转移时的下标
        }
        int nextn = nextTab.length;//新的 tab 的长度
// 创建一个 fwd 节点，表示一个正在被迁移的 Node，并且它的 hash 值为-1(MOVED)，也
        就是前面我们在讲 putval 方法的时候，会有一个判断 MOVED 的逻辑。它的作用是用来占位，表示
        原数组中位置 i 处的节点完成迁移以后，就会在 i 位置设置一个 fwd 来告诉其他线程这个位置已经
        处理过了，具体后续还会在讲
        ForwardingNode<K, V> fwd = new ForwardingNode<K, V>(nextTab);
// 首次推进为 true，如果等于 true，说明需要再次推进一个下标（i--），反之，如果是
        false，那么就不能推进下标，需要将当前的下标处理完毕才能继续推进
        boolean advance = true;
      
//判断是否已经扩容完成，完成就 return，退出循环
        boolean finishing = false; // to ensure sweep before committing
        nextTab
        通过 for 自循环处理每个槽位中的链表元素，默认 advace 为真，通过 CAS 设置
        transferIndex 属性值，并初始化 i 和 bound 值，i 指当前处理的槽位序号，bound 指需要处理
        的槽位边界，先处理槽位 15 的节点；
        for (int i = 0, bound = 0; ; ) {
// 这个循环使用 CAS 不断尝试为当前线程分配任务
// 直到分配成功或任务队列已经被全部分配完毕
// 如果当前线程已经被分配过 bucket 区域
// 那么会通过--i 指向下一个待处理 bucket 然后退出该循环
            Node<K, V> f;
            int fh;
            while (advance) {
                int nextIndex, nextBound;
//--i 表示下一个待处理的 bucket，如果它>=bound,表示当前线程已经分配过
                bucket 区域
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {//表示所有 bucket 已经
                    被分配完毕
                            i = -1;
                    advance = false;
                }
//通过 cas 来修改 TRANSFERINDEX,为当前线程分配任务，处理的节点区间为
                (nextBound, nextIndex) -> (0, 15)
else if (U.compareAndSwapInt
                        (this, TRANSFERINDEX, nextIndex,
                                nextBound = (nextIndex > stride ?
                                        nextIndex - stride : 0))) {
                    bound = nextBound;//0
                    i = nextIndex - 1;//15
                    advance = false;
                    咕泡学院 - 做技术人的指路明灯，职场生涯的精神导师
                }
            }
//i<0 说明已经遍历完旧的数组，也就是当前线程已经处理完所有负责的 bucket
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {//如果完成了扩容
                    nextTable = null;//删除成员变量
                    table = nextTab;//更新 table 数组
                    sizeCtl = (n << 1) - (n >>> 1);//更新阈值(32*0.75=24)
                    return;
                }
// sizeCtl 在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2 ( 详细介
                绍点击这里)
// 然后，每增加一个线程参与迁移就会将 sizeCtl 加 1，
// 这里使用 CAS 操作对 sizeCtl 的低 16 位进行减 1，代表做完了属于自己的任
                务
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    第一个扩容的线程，执行 transfer 方法之前，会设置 sizeCtl =
                            (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)
                    后续帮其扩容的线程，执行 transfer 方法之前，会设置 sizeCtl = sizeCtl + 1
                    每一个退出 transfer 的方法的线程，退出之前，会设置 sizeCtl = sizeCtl - 1
                    那么最后一个线程退出时：必然有
                    sc == (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)，即(sc - 2)
                            == resizeStamp(n) << RESIZE_STAMP_SHIFT
// 如果 sc - 2 不等于标识符左移 16 位。如果他们相等了，说明没有线程在
                    帮助他们扩容了。也就是说，扩容结束了。
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
// 如果相等，扩容结束了，更新 finising 变量
                    finishing = advance = true;
// 再次循环检查一下整张表
                    i = n; // recheck before commit
                    咕泡学院 - 做技术人的指路明灯，职场生涯的精神导师
                }
            }
// 如果位置 i 处是空的，没有任何节点，那么放入刚刚初始化的 ForwardingNode ”空节点“
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
//表示该位置已经完成了迁移，也就是如果线程 A 已经处理过这个节点，那么线程 B 处理这个节点
            时，hash 值一定为 MOVED
else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
        }
    }
```   

代码太长了，也真心不好看，就画个图来说明吧：  

ConcurrentHashMap 支持并发扩容，实现方式是，把 Node 数组进行拆分，让每个线程处理
自己的区域，假设 table 数组总长度是 64，默认情况下，那么每个线程可以分到 16 个 bucket。
然后每个线程处理的范围，按照倒序来做迁移
通过 for 自循环处理每个槽位中的链表元素，默认 advace 为真，通过 CAS 设置 transferIndex
属性值，并初始化 i 和 bound 值，i 指当前处理的槽位序号，bound 指需要处理的槽位边界，
先处理槽位 31 的节点； （bound,i） =(16,31) 从 31 的位置往前推动。  
![image](https://wx4.sinaimg.cn/mw1024/007R8l8Fly1g9pabwvgkmj30it061wei.jpg)

假设这个时候 ThreadA 在进行 transfer，那么逻辑图表示如下  

![image](https://wx4.sinaimg.cn/mw1024/007R8l8Fly1g9pabwly86j30hk0agt8r.jpg)  

![image](https://wx4.sinaimg.cn/mw1024/007R8l8Fly1g9pabwly86j30hk0agt8r.jpg)






###### sizeCtl 扩容退出机制  

在扩容操作 transfer 的第 2414 行，代码如下
if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
每存在一个线程执行完扩容操作，就通过 cas 执行 sc-1。
接着判断(sc-2) !=resizeStamp(n) << RESIZE_STAMP_SHIFT ; 如果相等，表示当前为整个扩
容操作的 最后一个线程，那么意味着整个扩容操作就结束了；如果不想等，说明还得继续
这么做的目的，一方面是防止不同扩容之间出现相同的 sizeCtl，另外一方面，还可以避免
sizeCtl 的 ABA 问题导致的扩容重叠的情况  



###### 数据迁移阶段的实现分析  

```

    synchronized (f)

    {//对数组该节点位置加锁，开始处理数组该位置的迁移工作
        if (tabAt(tab, i) == f) {//再做一次校验
            Node<K, V> ln, hn;//ln 表示低位， hn 表示高位;接下来这段代码的作用
            是把链表拆分成两部分，0 在低位，1 在高位
            if (fh >= 0) {//下面部分代码原理点击这里
                int runBit = fh & n;
                Node<K, V> lastRun = f;
//遍历当前 bucket 的链表，目的是尽量重用 Node 链表尾部的一部分
                for (Node<K, V> p = f.next; p != null; p = p.next) {
                    int b = p.hash & n;
                    if (b != runBit) {
                        runBit = b;
                        lastRun = p;
                    }
                }
                if (runBit == 0) {
                    如果最后更新的 runBit 是 0，设置低位节点
                            ln = lastRun;
                    hn = null;
                } else {//否则，设置高位节点
                    hn = lastRun;
                    ln = null;
                }
//构造高位以及低位的链表
                for (Node<K, V> p = f; p != lastRun; p = p.next) {
                    int ph = p.hash;
                    K pk = p.key;
                    V pv = p.val;
                    if ((ph & n) == 0)
                        ln = new Node<K, V>(ph, pk, pv, ln);
                    else
                        hn = new Node<K, V>(ph, pk, pv, hn);
                }
                setTabAt(nextTab, i, ln);//将低位的链表放在 i 位置也就是不动
                setTabAt(nextTab, i + n, hn);//将高位链表放在 i+n 位置
                咕泡学院 - 做技术人的指路明灯，职场生涯的精神导师
                setTabAt(tab, i, fwd); // 把旧 table 的 hash 桶中放置转发节
                点，表明此 hash 桶已经被处理
                        advance = true;
            }
//红黑树的扩容部分暂时忽略
        }   
```   

###### 高低位原理分析：  

ConcurrentHashMap 在做链表迁移时，会用高低位来实现，这里有两个问题要分析一下
1. 如何实现高低位链表的区分
假如我们有这样一个队列  

![image](https://wx2.sinaimg.cn/mw1024/007R8l8Fly1g9pajiysglj30vy0dlt8y.jpg)

第 14 个槽位插入新节点之后，链表元素个数已经达到了 8，且数组长度为 16，优先通过扩容
来缓解链表过长的问题，扩容这块的图解稍后再分析，先分析高低位扩容的原理    


假如当前线程正在处理槽位为 14 的节点，它是一个链表结构，在代码中，首先定义两个变量
节点 ln 和 hn，实际就是 lowNode 和 HighNode，分别保存 hash 值的第 x 位为 0 和不等于
0 的节点  

通过 fn&n 可以把这个链表中的元素分为两类，A 类是 hash 值的第 X 位为 0，B 类是 hash 值
的第 x 位为不等于 0（至于为什么要这么区分，稍后分析），并且通过 lastRun 记录最后要处
理的节点。最终要达到的目的是，A 类的链表保持位置不动，B 类的链表为 14+16(扩容增加
的长度)=30    

我们把 14 槽位的链表单独伶出来，我们用蓝色表示 fn&n=0 的节点，假如链表的分类是这
样  
```
for (Node<K,V> p = f.next; p != null; p = p.next) {
int b = p.hash & n;
if (b != runBit) {
runBit = b;
lastRun = p;
}
}
```   




通过上面这段代码遍历，会记录 runBit 以及 lastRun，按照上面这个结构，那么 runBit 应该
是蓝色节点，lastRun 应该是第 6 个节点
接着，再通过这段代码进行遍历，生成 ln 链以及 hn 链  


```
for (Node<K,V> p = f; p != lastRun; p = p.next) {
int ph = p.hash; K pk = p.key; V pv = p.val;
if ((ph & n) == 0)
ln = new Node<K,V>(ph, pk, pv, ln);
else
hn = new Node<K,V>(ph, pk, pv, hn);
} 
```    


接着，通过 CAS 操作，把 hn 链放在 i+n 也就是 14+16 的位置，ln 链保持原来的位置不动。
并且设置当前节点为 fwd，表示已经被当前线程迁移完了


![image](https://wx3.sinaimg.cn/mw1024/007R8l8Fly1g9pajiy7ypj30z306kmxa.jpg)    


###### 为什么要做高低位的划分
要想了解这么设计的目的，我们需要从 ConcurrentHashMap 的根据下标获取对象的算法.   

```
(f = tabAt(tab, i = (n - 1) & hash)) == null
通过(n-1) & hash 来获得在 table 中的数组下标来获取节点数据，【&运算是二进制运算符，1
& 1=1，其他都为 0】

```   

假设我们的 table 长度是 16，二进制是【0001 0000】，减一以后的二进制是 【0000 1111】  
假如某个 key 的 hash 值=9，对应的二进制是【0000 1001】，那么按照（n-1） & hash 的算法
0000 1111 & 0000 1001 =0000 1001 ， 运算结果是 9  
当我们扩容以后，16 变成了 32，那么(n-1)的二进制是 【0001 1111】  
仍然以 hash 值=9 的二进制计算为例  
0001 1111 & 0000 1001 =0000 1001 ，运算结果仍然是 9  
我们换一个数字，假如某个 key 的 hash 值是 20，对应的二进制是【0001 0100】，仍然按照(n-1) & hash  
算法，分别在 16 为长度和 32 位长度下的计算结果  
16 位： 0000 1111 & 0001 0100=0000 0100  
32 位： 0001 1111 & 0001 0100 =0001 0100  
从结果来看，同样一个 hash   值，在扩容前和扩容之后，得到的下标位置是不一样的，这种情况当然是  
不允许出现的，所以在扩容的时候就需要考虑，
而使用高低位的迁移方式，就是解决这个问题.
大家可以看到，16 位的结果到 32 位的结果，正好增加了 16.
比如 20 & 15=4 、20 & 31=20 ； 4-20 =16
比如 60 & 15=12 、60 & 31=28； 12-28=16
所以对于高位，直接增加扩容的长度，当下次 hash   获取数组位置的时候，可以直接定位到对应的位置。  
这个地方又是一个很巧妙的设计，直接通过高低位分类以后，就使得不需要在每次扩容的时候来重新计算 hash，极大提升了效率。 

扩容结束以后的退出机制  
如果线程扩容结束，那么需要退出，就会执行 transfer 方法的如下代码:  

```
//i<0 说明已经遍历完旧的数组，也就是当前线程已经处理完所有负责的 bucket
if (i < 0 || i >= n || i + n >= nextn) {
int sc;
if (finishing) {//如果完成了扩容
nextTable = null;//删除成员变量
table = nextTab;//更新 table 数组
sizeCtl = (n << 1) - (n >>> 1);//更新阈值(32*0.75=24)
return;
}
// sizeCtl 在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2
// 然后，每增加一个线程参与迁移就会将 sizeCtl 加 1，
// 这里使用 CAS 操作对 sizeCtl 的低 16 位进行减 1，代表做完了属于自己的任
务
if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
第一个扩容的线程，执行 transfer 方法之前，会设置 sizeCtl =
(resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)
后续帮其扩容的线程，执行 transfer 方法之前，会设置 sizeCtl = sizeCtl+1
每一个退出 transfer 的方法的线程，退出之前，会设置 sizeCtl = sizeCtl-1
那么最后一个线程退出时：必然有
sc == (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)，即 (sc - 2)
== resizeStamp(n) << RESIZE_STAMP_SHIFT
// 如果 sc - 2 不等于标识符左移 16 位。如果他们相等了，说明没有线程在
帮助他们扩容了。也就是说，扩容结束了。
if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
return;
// 如果相等，扩容结束了，更新 finising 变量
finishing = advance = true;
// 再次循环检查一下整张表
i = n; // recheck before commit
}
}
```   


如果对应的节点存在，判断这个节点的 hash 是不是等于 MOVED(-1)，说明当前节点是
ForwardingNode 节点，
意味着有其他线程正在进行扩容，那么当前现在直接帮助它进行扩容，因此调用 helpTransfer
方法    


```
else if ((fh = f.hash) == MOVED)
tab = helpTransfer(tab, f);

helpTransfer
从名字上来看，代表当前是去协助扩容
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
Node<K,V>[] nextTab; int sc;
// 判断此时是否仍然在执行扩容,nextTab=null 的时候说明扩容已经结束了
if (tab != null && (f instanceof ForwardingNode) &&
(nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
int rs = resizeStamp(tab.length);//生成扩容戳
while (nextTab == nextTable && table == tab &&
(sc = sizeCtl) < 0) {//说明扩容还未完成的情况下不断循环来尝试将当前
线程加入到扩容操作中
//下面部分的整个代码表示扩容结束，直接退出循环
//transferIndex<=0 表示所有的 Node 都已经分配了线程
//sc=rs+MAX_RESIZERS 表示扩容线程数达到最大扩容线程数
//sc >>> RESIZE_STAMP_SHIFT !=rs， 如果在同一轮扩容中，那么 sc 无符号
右移比较高位和 rs 的值，那么应该是相等的。如果不相等，说明扩容结束了
//sc==rs+1 表示扩容结束
if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
sc == rs + MAX_RESIZERS || transferIndex <= 0)
break;//跳出循环
if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {//在低 16 位
上增加扩容线程数
transfer(tab, nextTab);//帮助扩容
break;
}
}
return nextTab;
}
return table;//返回新的数组
}
```