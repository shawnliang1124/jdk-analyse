##### 一、基本概念   
 1.为什么需要ReentrantLock ？   
 很简单，ReentrantLock作为Lock的一个实现类，也叫重入互斥锁。下面来解释一下这个名词。   
 - 重入：  
 重入能够避免死锁的可能，同一个线程抢到了同一把锁，只是重入次数+1。  
- 互斥：  
互斥的作用是保证只能有一个线程进行操作，别的线程进行等待。


Synchornized关键字的作用确实是有锁的功能，但是我们可以看到
Synchornized无法手动地释放锁，只能方法出栈，或者出现异常，由JVM底层去帮我们释放。  

##### 二、数据结构   
简单看下ReentrantLock的类结构图： 
![image](https://wx4.sinaimg.cn/mw1024/007R8l8Fly1g9lti6sbjaj30v10hcdgi.jpg)  

可以看到ReentrantLock实现了Lock,但是底层是Sync内部类，这个内部类底层又是继承了AQS(AbstractQueuedSynchronizer)抽象类。注意了，这个AQS实际上是J.U.C里面的基础，很多类底层都是通过AQS来实现的。    

##### 三. AQS分析   
1. AQS是什么？  
**在 Lock 中，用到了一个同步队列 AQS，全称 AbstractQueuedSynchronizer，它
是一个同步工具也是 Lock 用来实现线程同步的核心组件。**

2.AQS的功能是什么？  
**AQS功能主要分为两种，独占和共享**  

**独占锁：每次只能有一个线程持有锁，比如前面给大家演示的 ReentrantLock 就是
以独占方式实现的互斥锁**   

**共 享 锁： 允 许 多 个 线 程 同 时 获 取 锁 ， 并 发 访 问 共 享 资 源 ， 比 如
ReentrantReadWriteLock**   


3.AQS的数据结构是什么？   
AQS 队列内部维护的是一个 FIFO 的**双向链表**，这种结构的特点是每个数据结构
都有两个指针，分别指向直接的后继节点和直接前驱节点。所以双向链表可以从任
意一个节点开始很方便的访问前驱和后继。每个 Node 其实是由线程封装，当线
程争抢锁失败后会封装成 Node 加入到 ASQ 队列中去；当获取锁的线程释放锁以
后，会从队列中唤醒一个阻塞的节点(线程)。  

上代码：   
```
// 头结点，你直接把它当做 当前持有锁的线程 可能是最好理解的
private transient volatile Node head;

// 阻塞的尾节点，每个新的节点进来，都插入到最后，也就形成了一个链表
private transient volatile Node tail;

// 这个是最重要的，代表当前锁的状态，0代表没有被占用，大于 0 代表有线程持有当前锁
// 这个值可以大于 1，是因为锁可以重入，每次重入都加上 1
private volatile int state;

// 代表当前持有独占锁的线程，举个最重要的使用例子，因为锁可以重入
// reentrantLock.lock()可以嵌套调用多次，所以每次用这个来判断当前线程是否已经拥有了锁
// if (currentThread == getExclusiveOwnerThread()) {state++}
private transient Thread exclusiveOwnerThread; //继承自AbstractOwnableSynchronizer
```   

四、ReentrantLock lock方法时序图：   

![image](https://wxt.sinaimg.cn/mw1024/007R8l8Fly1g9lulfmztqj30ip08ewih.jpg?tags=%5B%5D)  


##### 四、源码分析   
下面直接进入源码分析阶段：   

```
 debug开始
  public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock();
        lock.lock();
        //业务代码
        lock.unlock();
    }
}
```  

调用了ReentrantLock方法:   
```
 public void lock() {
        sync.lock();
    }
```  

这个sync.lock方法，有两个实现，NonfairSync和FairSync。非公平锁和公平锁。默认实现是非公平锁NonfairSync。。

```
 /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }
```  

好了，进入NonfairSync的lock代码  
```
     final void lock() {
            if (compareAndSetState(0, 1))   //如果锁没有被任何线程锁定且加锁成功则设定当前线程为锁的拥有者
                setExclusiveOwnerThread(Thread.currentThread());    //将当前线程设置为获取锁的线程
            else
                acquire(1); //获取锁失败就走这个方法
        }

     
```  

CAS操作，假设当前锁还没有线程占领，ThreadA进入lock方法，将全局变量state改成1，并且把线程设置成ThreadA，这里很简单。。  

唯一的点就是这个compareAndSetState方法要进行讲解。  
```
  protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```  

这一段代码的意思是：通过CAS乐观锁的方式进做比较并替换。假设当前内存中的state的值和预期值expect相等，那么就将其替换为update，替换成功返回true，否则返回false。  

state是AQS的一个属性啊，之前提到过，在不同的实现类中，这个state代表的含义不相同，在可重入锁中，代表以下的含义。   
1.当state=0，表示无锁状态。   
2.当state>0，表示已经有线程获得了锁，也就是state=1,但是因为ReentrantLock允许重入，同一个线程在重复获得锁的时候，state会递增，假设state=6，证明这个线程重复获得了6次锁。   


好了，现在来了ThreadB，调用lock方法，肯定就是走lock方法的else逻辑了（假设A还没释放锁）， 调用的是AQS的acquire(int args)方法。   
```
   public final void acquire(int arg) {
            if (!tryAcquire(arg) &&
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                selfInterrupt();
        }
```   



拆开分析，首先是tryAcquire(arg)，意思是尝试下能不能获取到锁。看看代码：  

```
 protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
        
  final boolean nonfairTryAcquire(int acquires) { //返回true就代表获取到锁，两种可能 1没有线程在获取锁 2 重入锁，证明线程本身就是拿到锁，理所当然也能获取
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {   //c=0证明是没有线程用到这一把锁的
                if (compareAndSetState(0, acquires)) {  //cas成功的话就证明拿到了锁，否则证明刚刚瞬间被别的线程先抢到了锁
                    setExclusiveOwnerThread(current);   //cas成功的情况下，就标记一下，告诉大家，我这个线程拿到了锁
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {    //进入这个分支，代表是可重入
                int nextc = c + acquires;   //可重入的次数+1
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;  //并且返回true，代表是本线程获取到锁
            }
            return false;
        }
       
        
```   

返回true就是拿到了锁,false就是没拿到锁，
然后返回看看acquire(int arg)方法，调用了addWaiter(Node mode)方法。来看看，这个方法：  
```
  /**
     * @return the new node //这个方法的主要作用是将无法获取锁的线程包装成一个Node添加到队尾的意思
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;   //默认初始化的时候为空
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);  //所以第一次肯定是走的enq方法
        return node;
    }
```   


enq方法作用是通过自旋操作把当前节点加到队列中，我们来看看enq方法怎么初始化队列的？   
```
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { //第一次tail肯定为null
                if (compareAndSetHead(new Node()))  //通过cas初始化一个node给链表头节点，注意，这个node就一个简单无参构造
                    tail = head;  //cas成功后，tail=head= new Node()
            } else {
                node.prev = t;  //初始化head成功没跳出循环啊
                if (compareAndSetTail(t, node)) {   //cas把传入的node节点加到链表的队尾中去
                    t.next = node;  //设置前节点的next为传入的node节点
                    return t;   //返回node节点的前节点
                }
            }
        }
    }
```  

**addWaiter方法做了什么？**   
当 tryAcquire 方法获取锁失败以后，则会先调用 addWaiter 将当前线程封装成
Node.
入参 mode 表示当前节点的状态，传递的参数是 Node.EXCLUSIVE，表示独占状
态。意味着重入锁用到了 AQS 的独占锁功能
1. 将当前线程封装成 Node
2. 当前链表中的 tail 节点是否为空，如果不为空，则通过 cas 操作把当前线程的
node 添加到 AQS 队列
3. 如果为空或者 cas 失败，调用 enq 将节点添加到 AQS 队列   


好，分析完addWaiter方法以后，我们再来看看acquireQueued(final Node node, int arg)方法  
```
   final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {  //自旋操作，我们要关注的是跳出的条件
                final Node p = node.predecessor();    //获得新增的node的前节点
                if (p == head && tryAcquire(arg)) { //这里又tryAcquire，看看能不能CAS获取到锁
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true; //这个if里面是没有返回的，必须走上面的if才能跳出循环
            }       //我们思考一个问题，这样一直自旋下去，会不会很影响性能？AQS底层又是怎么解决的呢？
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```   

这个acquireQueued方法的作用是以下几点：  
通过 addWaiter 方法把线程添加到链表后，会接着把 Node 作为参数传递给
acquireQueued 方法，去竞争锁
1. 获取当前节点的 prev 节点
2. 如果 prev 节点为 head 节点，那么它就有资格去争抢锁，调用 tryAcquire 抢占
锁
3. 抢占锁成功以后，把获得锁的节点设置为 head，并且移除原来的初始化 head
节点
4. 如果获得锁失败，则根据 waitStatus 决定是否需要挂起线程
5. 最后，通过 cancelAcquire 取消获得锁的操作  


**并且我还留了一个问题，一直for循环自旋，岂不是会造成资源浪费？**   

**JDK底层是怎么做的呢，如果一直拿不到锁，直接调用LockSupport.park(obj)方法，将当前的线程挂起，直到抢占锁的资源释放锁，再来唤醒，这样就节约了cpu的资源**

我们看看shouldParkAfterFailedAcquire方法：  


首先，有个变量要和大家先科普，就是Node节点的waitStatus。来分析一下：  
 有 5 中状态，分别是：CANCELLED（1），SIGNAL（-1）、CONDITION（-
2）、PROPAGATE(-3)、默认状态(0)  

**CANCELLED**: 在同步队列中等待的线程等待超时或被中断，需要从同步队列中取
消该 Node 的结点, 其结点的 waitStatus 为 CANCELLED，即结束状态，进入该状
态后的结点将不会再变化  

**SIGNAL**: 只要前置节点PROPAGATE：共享模式下，PROPAGATE 状态的线程处于可运行状态释放锁，就会通知标识为 SIGNAL 状态的后续节点的线程    


**CONDITION**： 和 Condition 有关系，后续会讲解


**PROPAGATE**：共享模式下，PROPAGATE 状态的线程处于可运行状态   

0:初始状态
```
 */ //方法的意思理解，在抢夺锁失败的情况下是否应该挂起
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) { //pred为前节点 node为当前节点
        int ws = pred.waitStatus;   //获得前置节点的状态
        if (ws == Node.SIGNAL)      //为SIGNAL状态，前置节点安心挂起吧，直接返回true，抢夺锁失败了，应该挂起
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {   //CANCELED的状态
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);  //遍历双向链表，移除cancelled的节点
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL); //cas无锁机制将prev节点的状态设置成Singnal
        }
        return false;
    }

```    


```
  /**
     * Convenience method to park and then check if interrupted
     *
     * @return {@code true} if interrupted
     */
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this); //调用底层的park方法来挂起线程
        return Thread.interrupted();
    }
```   


使用 LockSupport.park 挂起当前线程编程 WATING 状态
Thread.interrupted，返回当前线程是否被其他线程触发过中断请求，也就是
thread.interrupt();   

这个LockSupport.park(this)方法，会一直让这个线程挂起，但是不会释放线程的资源。也就是当前线程一直hold住在这里。    


好，下面用一张图来解释一下Node节点的变化流程   
![image](https://wx3.sinaimg.cn/mw1024/007R8l8Fly1g9m1kywyzkj30ro0ep0tx.jpg)  



对了，释放锁的流程还没讲呢，释放锁无非就是调用的unlock方法了。不用看都知道，将head节点的Thread置为null，将state全局变量设置成0,然后将刚刚park的线程unpark，解除阻塞的状态 


下面我们看源码：    
```
//返回true释放成功，返回false为释放失败
 public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;  //释放锁成功的情况下
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h); //head不为空的情况下调用unparkSuccessor方法
            return true;
        }
        return false;
    }
```   

先看tryRelease方法   

```
    //返回true才释放锁成功，否则失败
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;  //因为ReenreantLock是可重入的，state是正整数
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {   //c=0的情况就释放锁
                free = true;    
                setExclusiveOwnerThread(null);  //将当前持有锁的线程设置为null
            }
            setState(c);    //设置state的值
            return free;   
        }
  
```   


这种代码so easy了，大家都能懂，我们回到release方法里面，再看看unparkSuccessor的方法，这个方法的作用是什么？  

到里面看看去：   

```
   private void unparkSuccessor(Node node) {   //这个node是原本链表的头节点
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);   //cas将头节点的waitStatus设置为0

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next; //获得头节点的下个节点，也就是等待的线程
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev) //很好奇吧，为什么这里如果是cancel的状态，是从后续遍历来删除节点？
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);   //调用Unpark方法，将线程的阻塞状态取消
    }
```  

这里来解释，为什么释放锁的时候，是从后序来遍历链表，而不是前序遍历：  

我们回到enq那个方法看看就知道啦。   

![image](https://wx3.sinaimg.cn/mw1024/007R8l8Fgy1g9m2e1v8rgj31ao0gd0uy.jpg)  

看看红色方框里面的代码，用图表示：  

![image](https://wx1.sinaimg.cn/mw1024/007R8l8Fgy1g9m2fmb09cj30ot05ymxf.jpg)   

next的关系是在加入到链表之后才生成的，并且3的操作没有保证原子性，因为没用cas操作，很可能某个线程做完第二部的cas(tail)后，链表关系还没建立完整呢，从前开始遍历会导致链表中断。。。  


继续下一步，我们刚刚在哪里停来着，park方法在哪里就在哪里停的。


```
 final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {  //自旋操作，我们要关注的是跳出的条件
                final Node p = node.predecessor();    //获得新增的node的前节点
                if (p == head && tryAcquire(arg)) { //这里又tryAcquire，看看能不能CAS获取到锁
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true; //这个if里面是没有返回的，必须走上面的if才能跳出循环
            }       //我们思考一个问题，这样一直自旋下去，会不会很影响性能？AQS底层又是怎么解决的呢？
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

```   

既然线程A已经把锁让了出来，那么假设线程线程B拿到了锁，就直接走的 if (p == head && tryAcquire(arg))里面的逻辑了。    

首先将自己node设置成链表的头，并且让自己的next(原来的head节点)指向null。至此，ReentrantLock获取整个流程分析结束。   
现在的节点状态是这样的：  
![image](https://wx4.sinaimg.cn/mw1024/007R8l8Fgy1g9m2r7u0svj30w406e74n.jpg)    


