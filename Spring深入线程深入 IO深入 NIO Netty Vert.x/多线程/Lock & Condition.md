# Lock & Condition



## 互斥锁

###  锁 可重入性

concurrent包中的锁都是"可重入锁",一般都命名为ReentrantX.可重入锁是当一个线程调用obj.lock()拿到锁,进入互斥区,再次调用obj.lock(),仍然可以拿到锁.

![image-20200914201429801](https://gitee.com/shao_dw/pic/raw/master/upload/image-20200914201429801.png)

I:接口,A:abstract class, C 类 $内部类 实线继承, 虚线引用

```java
public interface Lock {
  	void lock(); //不能被中断
    void lockInterruptibly() throws InterruptedException;//可以被中断
    boolean tryLock();	
    void unlock();
    Condition newCondition();
}

ReentrantLock实现都在内部类Sync上
public class ReentrantLock implements Lock{
    private final Sync sync;
    public void lock() {
        sync.lock();
    }
    public void unlock() {
        sync.release(1);
    }
}
```

### 锁的公平性 与非公平性

```java
abstract static class Sync extends AbstractQueuedSynchronizer
    Sync是抽象类,两个子类FairSync 和 NonfairSync, 
默认非公平锁 为了提高效率,减少线程切换
```

### 锁实现的基本原理

Sync的父类AbstractQueuedSynchronizer队列同步器(AQS)

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
```

对于Atomic类.都是自旋性质的锁,而这里的锁具备synchronized功能,可以阻塞一个线程.

```
为了实现一把具有阻塞和唤醒功能的锁,需要以下:
1. 需要一个state,标记该锁的状态,state至少两个值:0,1 对state变量的操作,要确保线程安全,也就是会用CAS
2. 需要记录当前是哪个线程持有锁
3. 需要底层支持对一个线程进行阻塞或唤醒操作
4. 需要有一个队列维护所有阻塞的线程.这个队列必须是线程安全的无锁队列,也需要用到CAS.
```

对于1.2 在两个类中有体现:

```java
public abstract class AbstractOwnableSynchronizer{
	private transient Thread exclusiveOwnerThread;
    //记录锁被哪个线程持有
}
public abstract class AbstractQueuedSynchronizer{
    private volatile int state;//记录锁的状态,通过cas修改state
}

state取值不仅可以为0,1还可以大于1  是为了支持可重入性
    同一个线程,调用5次lock,state变成5,调用5次unlock,state减为0
    state=0时,没有线程持有锁,exclusiveOwnerThread=null
    state=1时,有一个线程持有锁,exclusiveOwnerThread=该线程
    state>1,说明该线程重入了该锁
```

对于3: 需要底层支持对一个线程进行阻塞或唤醒操作,在Unsafe类中提供了阻塞和唤醒线程的一对操作原语,也就是park和unpark

```java
public native void unpark(Object var1);

public native void park(boolean var1, long var2);
```

LockSupport工具类对这对原语做了简单封装:

```java
public class LockSupport{
    public static void park() {
        UNSAFE.park(false, 0L);
    }
    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
}
当前线程中调用park(),该线程就会被阻塞;
在另一个线程中,调用unpark(Thread t),传入一个被阻塞的线程,就可以唤醒阻塞在park()地方的线程.
    unpark(Thread t),实现了一个线程对另外一个线程的"精准唤醒".wait和notify无法指定唤醒哪个线程.
```

对于4,在AQS中利用双向链表和CAS实现了一个阻塞队列.

```java
public abstract class AbstractQueuedSynchronizer{
    static final class Node{
        volatile Node prev;
        volatile Node next;
        volatile Thread thread; //每个node对应一个被阻塞的线程
    }
    
    private transient volatile Node head;
    private transient volatile Node tail;
}
```

![image-20200914212822800](https://gitee.com/shao_dw/pic/raw/master/upload/image-20200914212822800.png)head指向双向链表头部,tail指向尾部

入队就是把新的Node加到tail后面,然后对tail进行cas;

出队就是对head进行cas操作,把head向后移一个位置

初始化时:head = tail = null;然后,在往队列中加入阻塞的线程时,会创建一个空Node,让head和tail指向这个空Node,之后,在后面加入被阻塞的线程对象.所以当head = tail时,说明队列为空.

## 公平 非公平的lock实现

```java
static final class FairSync extends Sync {
    final void lock() {
        acquire(1); //没有一上来就抢锁,在这个函数内部排队,是公平的.
    }
}

static final class NonfairSync extends Sync {
     final void lock() {
        if (compareAndSetState(0, 1))
           etExclusiveOwnerThread(Thread.currentThread());
         //一上来就尝试修改state值,也就是抢锁,不考虑队列中有没有其他线程在排队,是非公平的.
        else
           acquire(1);
        }
}


acquire是AQS的模板方法:
public abstract class AbstractQueuedSynchronizer{
  public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
}

tryAcquire是一个虚函数,也就是再次尝试拿锁,被NonFairSync和FairSync分别实现, acquireQueued()函数目的是把线程放入阻塞队列,然后阻塞该线程.acquireQueue函数见下一节
    

static final class FairSync extends Sync{
       protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //两段代码这里多了一个if
                //只有当c==0时候,没有线程持有锁,并且排在队列的第一个时,才去抢锁,否则,继续排队,称之公平
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
}
 

abstract static class Sync extends AbstractQueuedSynchronizer{
    final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //无人持有锁,开始抢锁
                if (compareAndSetState(0, acquires)) {
                    //拿锁成功,设置ExclusiveOwnerThread为当前线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
}
```



### 阻塞队列和唤醒机制

```java
public abstract class AbstractQueuedSynchronizer{
    
    
    //addWaiter为当前线程生成一个Node放入双向链表的尾部,只是把Thread放入了一个队列,线程本身并未被阻塞
     private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {//先尝试加到队列尾部,不成功则执行enq
                pred.next = node;
                return node;
            }
        }
        enq(node); //enq内部会进行队列的初始化,新建一个新的空Node,然后不断尝试自旋,直至成功把该Node加入队列尾部为止.
        return node;
    }
    
    
    //acquireQueued(addWaiter(Node.EXCLUSIVE), arg)
    在addWaiter把Thread对象加入阻塞队列了之后的工作要靠acquireQueued完成.
        线程一旦进入acquireQueued就会被无限期阻塞,即使其他线程调用interrupt()函数也不能将其唤醒,除非有其他线程释放了锁,并且该线程拿到了锁,才会从
        acquireQueued()返回
    进入acquireQueued(),该线程被阻塞,在该函数返回的一刻,就是拿到锁的那一刻,也就是被唤醒的那一刻此时会删除队列的第一个元素,(head指针前移一个节点)
        
    final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) { //被唤醒,如果自己在队列头部,(自己的前一个结点是head指向的空节点),则尝试拿锁;
                
                setHead(node);//拿锁成功,出队列(head前移一个节点),同时会把node的thread置为null,所以head还是指向了一个空节点
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())//阻塞发生在这个函数中,线程调用park(),自己把自己阻塞,直到被其他线程唤醒,该函数返回.
                //park()函数返回有两种情况:
                //1:其他线程调用unpark(Thread t)
                //2:其他线程调用t.interrupt().这里要注意的是:lock不能响应中断,但是LockSupport.park()会响应中断.也正因为LockSupport.park（）可能被中断唤醒，acquireQueued（..）函数才写了一个for死循环。唤醒之后，如果发现自己排在队列头部，就去拿锁；如果拿不到锁，则再次自己阻塞自己。不断重复此过程，直到拿到锁。被唤醒之后，通过Thread.interrupted（）来判断是否被中断唤醒。如果是情况1，会返回false；如果是情况2，则返回true。
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
	}   
}

   acquireQueued的返回值:虽然该函数不会响应中断,但它会记录阻塞期间有没有其他线程向他发送过中断信号,如果有,返回true   
       基于该返回值,才有:
 public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
//返回true时,调用selfInterrupt().自己给自己发送中断信号,也就是自己把自己的中断标志位设为true.之所以要这么做，是因为自己在阻塞期间，收到其他线程中断信号没有及时响应，现在要进行补偿。这样一来，如果该线程在lock代码块内部有调用sleep（）之类的阻塞方法，就可以抛出异常，响应该中断信号。
```

```java
volatile int state = 0;//没有锁
Queue parkQueue;//假设为同步队列

void lock(){
    while (!compareAndSet(0,1)){
        park();//第一个线程进来是0,cas改成1 不会进while往下执行
        //此时第二个线程进来,cas更改state state为1 更改失败,进入while 但是不能一直让它占着cpu 那就给我阻塞掉  
        //使用volatile 要知道 volatile写的结果对任意后续volatile读可见,别的线程可以马上知道,statue被修改为0,可以尝试获取锁
    }
    .....
    unlock();
}

void unlock(){
    status = 0;
    lock_notify();
}

void park(){
    //当前线程不能让它一直在那里转浪费cpu 拿不到锁就给我阻塞
    parkQueue.add(currentThread);
    releaseCpu();
}

void lock_notify(){
    //得到要唤醒的队列头部线程
    Thread t = parkQueue.header();
    unpark(t);
}

//同步就是让我们的lock方法正常返回

//线程交替执行 并发执行
并发编程不一定总是有竞争
```

