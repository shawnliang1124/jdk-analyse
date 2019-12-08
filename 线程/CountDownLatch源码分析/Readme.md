##### 一、应用场景   
Condition里面主要是有两个方法，await()和signal，注意的是Condition是一个接口，上述提到的两个方法，主要是用于线程之间的通讯，类似于synchornized的wait和notify方法，常用于生产者消费者模式中。。。 注意的是, Condition必须和lock锁一起使用

##### 二、Condition的demo   

关于Condition的简单demo   

生产者生产完东西之后，调用await方法进行挂起。看下代码：  
```
public class ConditionAwait implements Runnable {
    private Lock lock;

    private Condition condition;

    public ConditionAwait(Lock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName()+"begin await");
        try {
            lock.lock();
            //生产者业务代码   produceSth()
            condition.await();
            System.out.println(Thread.currentThread().getName()+"end await");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        finally {
          lock.unlock();
        }
    }
}

```     

消费者马上进行消费。。   
```
public class ConditionNotify implements Runnable {

    private Lock lock;

    private Condition condition;

    public ConditionNotify(Lock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }


    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName()+"begin notify");
        try {
            lock.lock();
            //消费者业务代码 consumeSth()
            condition.signal();
            System.out.println(Thread.currentThread().getName()+"end notify");
        } finally {
          lock.unlock();
        }
    }
}

```    

最后是一个测试类：   

```
public class ConditionDemo {
   static Lock lock = new ReentrantLock();
   static Condition condition = lock.newCondition();

    public static void main(String[] args) {
        ConditionDemo conditionDemo = new ConditionDemo();
        ConditionAwait conditionAwait = new ConditionAwait(lock, condition);
        ConditionNotify conditionNotify = new ConditionNotify(lock, condition);
        new Thread(conditionAwait).start();
        new Thread(conditionNotify).start();
    }

}

```   

输出的结果是  
```
Thread-0 begin await
Thread-1 begin notify
Thread-1 end notify
Thread-0 end await
```   

##### 三、源码分析   
来，在了解完DEMO之后，直接开始分析源码，首先，分析Condition的源码之前肯定是要对ReentrantLock的源码有一定的了解，不然是没法了解底层的原理的。。   

**首先要知道await方法是干嘛？**  
await方法是首先将抢占到锁的线程资源进行释放，之后并且调用底层的LockSupport.park方法进行线程的阻塞。  


**那么,signal方法是干嘛的？**   
signal方法就是唤醒之前由于await阻塞的线程  

先科普一个整体的概念：   

实现await和notify需要用到两个底层的数据结构。  
一个是**AQS底层的双向链表**：里面存放的是获取不到锁的竞争线程节点（头节点存放的是抢到线程的节点）。   
一个是**ConditionObject的单向链表**：里面存放的是因为await进行阻塞的线程节点。  

先不分析代码，一个wait和signal的逻辑无非是：  
先将获得锁的线程放到**ConditionObject的单向链表**中，然后调用park方法挂起，并且在**AQS底层的双向链表**中删除头节点，将锁资源让给别的线程；；之后，别的线程消费完了东西，调用lock的signal方法，将**ConditionObject的单向链表**的节点删除，并且把删除的节点添加到**AQS底层的双向链表**后面去，并且删除双向链表的头节点(当前获得锁的线程)，这样，唤醒被阻塞的线程，并且让它重新拿到锁。大概就是这么个流程了。具体的实现，看源码分析吧。


首先看await方法：   

```
public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();       //无非就是ConditionObject初始化node节点
            int savedState = fullyRelease(node);    //调用fullyRelease的作用是要释放线程的锁，别问为什么，因为await方法作用就
            int interruptMode = 0;      //是让当前线程进入等待队列并且释放锁，同时让线程变为等待的状态
            while (!isOnSyncQueue(node)) {  //如果当前的Node不在Sync队列里面的话，
                LockSupport.park(this); //挂起当前的线程
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```   

这个方法的作用之前讲过，  
1.将抢占到锁的线程资源进行释放   
2.之后并且调用底层的LockSupport.park方法进行线程的阻塞。 

 这里减少枯燥的文字，笔者时间少。实际上代码实现的基本流程就是我下图画的那样。。。
 
 
 
 ![image](https://wx3.sinaimg.cn/mw1024/007R8l8Fly1g9n05evqp1j30u0129jtq.jpg)

掌握了AQS之后，这个CONDITION的逻辑真心不复杂  