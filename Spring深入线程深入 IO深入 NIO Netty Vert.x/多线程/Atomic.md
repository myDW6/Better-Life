# Atomic



juc层次:

CompletableFuture

线程池,Future,ForkJoinPool

并发容器(BlockingQueue, ConcurrentHashMap)

同步工具(信号量,CountDownLatch)

锁与条件(互斥锁,读写锁)

Atomic类



## AtomicInteger & AtomicLong

```java
class Example{
    int count = 0;
   	void synchronized increment(){count++} //线程A调用
    void synchronized decrement(){count--} //线程B调用
}
//事实上,synchronized可以使用Atomic包下的类替代,性能更好
class Example{
    AtomicInteger count = new AtomicInteger(0);
    void add(){count.getAndIncrement}
    long decr(){count.getAndDecrement}
}
```

### 悲观锁与乐观锁

认为数据发生并发冲突的概率比较小,所以读之前不上锁,等到写时,判断数据在此期间是否被其他线程修改,如果修改,重新读,重复该过程.如果没有修改,就写回去.

判断是否被修改,同时写回新值,两个操作要合成一个原子操作,也就是CAS(Compare And Sert,  AtmoicInteger典型的乐观锁,Mysql和Redis也有类似思路)



### Unsafe的CAS

```java
//jdk11
public final boolean compareAndSet(int expectedValue, int newValue) {
    return U.compareAndSetInt(this, VALUE, expectedValue, newValue);
}
//Atomic封装过的CompareAndSet有两个参数,
//第一个指变量旧值(读出来的值,写回去的时候,希望没有被其他线程修改,所以叫expectValue);
//第二个参数newValue,指的是变量的新值(修改过的,希望写入的值,jdk7叫update).
//当expectedValue等于变量当前的值时,说明在修改的期间,没有其他线程对此变量进行过修改,所以可以成功写入.变量被更新为newValue.
//Unsafe类是concurrent包基础,所有函数都是native,
public final native boolean compareAndSetInt(Object o, long offset, int expected,int x);


提一下:
CAS(比较和交换) 伪代码
    compare_and_swap (*p, oldval, newval):
      if (*p == oldval)
          *p = newval;
          success;
      else
          fail;

jdk8该方法在Unsafe中为compareAndSwapInt
Atomic类中:
	public final boolean compareAndSet(int expect, int update) {
	        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
   	 }
调用unsafe类的:
    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
四个参数:第一个是对象(Atomic对象),第二个是对象成员变量(AtomicInteger的value)
    第二个参数是个long,经常被称为xxxOffset,意思是某个成员变量在对应的类中的内存偏移量(该变量在内存中的位置),表示该成员变量本身,使用Unsafe类的函数 public native long objectFieldOffset(Field var1); 将成员变量转化成偏移量.
    valueOffset的赋值:
	static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
所有调用CAS的地方,都会先通过这个函数把成员变量转化成一个offset,无论是Unsafe还是Offset都是静态的,类级别,所有对象公用.
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;
转化的时候,通过反射获取成员对象对应的Field对象,通过objectFieldOffset函数转为valueOffset,valueOffset就代表了value对象本身,后面执行cas,不是直接操作value,而是操作valueOffset

cas在jdk9命名:
	public final native boolean compareAndSetInt(Object o, long offset, int expected,int x);

```

### 自旋和阻塞

当一个线程拿不到锁,有俩种基本的等待策略:

1. 放弃cpu,进入阻塞,等待后续被唤醒,再重新被OS调度
2. 不放弃cpu,空转,不断尝试,就是自旋

单核cpu只能用策略1,如果不放弃cpu,其他线程无法运行释放锁,对于多核cpu,策略2没有线程切换开销

AtomicInteger实现采用自旋.拿不到锁会一直尝试.

两种策略并不互斥:可以结合,拿不到锁,先自旋,还拿不到,阻塞.参考synchronized实现.



## AtomicBoolean & AtomicReference

### 为啥要atomicboolean

对于int和long,需要进行加减,所以加锁,对于boolean,只有赋值和取值,加个volatile就差不多,为啥还需要原子类.

```java
if (flag == false){
    flag = true;
    ...
}
要实现上述操作,也就是实现compare 和 set两个操作合在一起的原子性,正是cas提供的功能,代码变成:
if (comparerAndSet(false, true)){
    ...
}

public final boolean compareAndSet(boolean expect, boolean update) {
        int e = expect ? 1 : 0;
        int u = update ? 1 : 0;
        return unsafe.compareAndSwapInt(this, valueOffset, e, u);
    }

同样,AtomicReference也需要该功能.
public final boolean compareAndSet(V expect, V update) {
    return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
}
//expect 为旧的引用,update为新的引用
```

### 对boolean和double的支持

unsafe提供了三种类型的cas:int long Object

```java
public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);

public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
第一个参数是要修改的对象,第二个参数是对象的成员变量再内存中的位置,第三个参数为该变量旧值,第四个参数是该变量的新值.
    
    boolean参考AtomicBoolean的boolean转整数,整数转boolean,
double类型,依赖一对double和long互相转换的函数,DoubleAdder
     public static native long doubleToRawLongBits(double value);
     public static native double longBitsToDouble(long bits);
```



## AtomicStampedReference & AtomicMarkableReference

### ABA问题

cas基于'值'比较,如果另外一个线程把变量的值从A改为B，再从B改回到A，那么尽管修改过两次，可是在当前线程做CAS操作的时候，却会因为值没变而认为数据没有被其他线程修改过，这就是所谓的ABA问题。

ABA解决:比较值,比较版本号, AtomicStampedReference 对应的cas函数:

```java
public boolean compareAndSet(V   expectedReference,
                             V   newReference,
                             int expectedStamp,
                             int newStamp) {
    Pair<V> current = pair;
    return
        expectedReference == current.reference &&
        expectedStamp == current.stamp &&
        ((newReference == current.reference &&
          newStamp == current.stamp) ||
         casPair(current, Pair.of(newReference, newStamp)));
}
//后两个参数就是版本号的旧值和新值
要解决Integer和Long的ABA问题,只有AtomicStampedReference没有AtomicStampedInteger/AtomicStampedLong,因为这里需要同时比较值和版本号,Integer和Long的CAS没办法同时比较两个变量,于是只能把值和版本号封装成一个Pair内部类,通过对象引用的cas实现
    
    private static class Pair<T> {
        final T reference;
        final int stamp;
        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }
//构造时
 public AtomicStampedReference(V initialRef, int initialStamp) {
        pair = Pair.of(initialRef, initialStamp);
    }


AtomicMarkableReference原理基本一致,只是版本号是boolean,不是整型的累加变量.
    private static class Pair<T> {
        final T reference;
        final boolean mark;
        private Pair(T reference, boolean mark) {
            this.reference = reference;
            this.mark = mark;
        }
        static <T> Pair<T> of(T reference, boolean mark) {
            return new Pair<T>(reference, mark);
        }
    }
不能完全避免ABA,降低发生概率
```





## AtomicIntegerFieldUpdater, AtomicLongFieldUpdater, AtomicReferenceFieldUpdater

### 为什么需要AtomicXXXFieldUpdater

如果自己编写一个类,可以在编写时,把成员变量定义为Atomic类型.

但是是一个已经存在的类,在不改变其源码的情况下,想实现对其成员变量的原子操作,就需要这些类.

构造函数是protected,只能使用:

```java
public static <U> AtomicIntegerFieldUpdater<U> newUpdater(
    Class<U> tclass,String fieldName)
    传入要修改的类和对应的成员变量的名字,通过反射拿到这个类的成员变量,包装成一个AtomicIntegerFieldUpdater对象,所以这个对象表示的是类的某个成员,而不是对象的成员变量..
    
    修改某个对象的成员变量的值,再传入相应的对象:
public int getAndIncrement(T obj) {
        int prev, next;
        do {
            prev = get(obj);
            next = prev + 1;
        } while (!compareAndSet(obj, prev, next));
        return prev;
    }

//AtomicIntegerFieldUpdaterImpl
public final boolean compareAndSet(T obj, int expect, int update) {
            accessCheck(obj);//检查类型是不是tClass,不是拒绝修改抛异常
            return U.compareAndSwapInt(obj, offset, expect, update);
        }c
cas原理和AtomicInteger一样,调用compareAndSwapInt()
    
    限制条件:使用AtomicIntegerFieldUpdater修改成员变量,成员变量必须是
        volatile int (不可以是Integer)可从构造函数中看出
        
         if (field.getType() != int.class)
                throw new IllegalArgumentException("Must be integer type");

            if (!Modifier.isVolatile(modifiers))
                throw new IllegalArgumentException("Must be volatile type");

AtomicLongFieldUpdater、AtomicReferenceFieldUpdater，也有类似的限制条件。其底层的CAS原理，也和AtomicLong、AtomicReference一样

```



## AtomicIntegerArray AtomicLongArray AtomicReferenceArray

数组元素的原子操作,不是说对整个数组的操作是原子的,而是针对数组中的每一个元素的原子操作.

使用很简单:

```java
public final int getAndIncrement(int i) {
    return getAndAdd(i, 1);
}
public final int getAndDecrement(int i) {
        return getAndAdd(i, -1);
    }
public final int getAndAdd(int i, int delta) {
        return unsafe.getAndAddInt(array, checkedByteOffset(i), delta);
    }
//传入数组下标
```

### 实现原理

cas还是用的compareAndSwapInt,把数组下标转为对应的内存偏移量

```java
//base表示数组的首地址位置,
//sacle表示一个数组元素的大小,
//i的偏移就是: i * scale + base

private static final int base = unsafe.arrayBaseOffset(int[].class);
//arrayBaseOffset方法用于返回数组中第一个元素实际地址相对整个数组对象的地址的偏移量。
private static final int shift;

 static {
        int scale = unsafe.arrayIndexScale(int[].class);
     //arrayIndexScale方法用于计算数组中第一个元素所占用的内存空间。
        if ((scale & (scale - 1)) != 0)
            throw new Error("data type scale not a power of two");
        shift = 31 - Integer.numberOfLeadingZeros(scale);
 }

//转换数组下标到内存偏移量
private static long byteOffset(int i) {
    return ((long) i << shift) + base;
}


scale表示2的倍数了 31 - scale 表示什么呢? shift表示scale中1的位置
 
    //scale为负数 i << -1  == i << 31
  
   
private boolean compareAndSetRaw(long offset, int expect, int update) {
        return unsafe.compareAndSwapInt(array, offset, expect, update);
    }
//cas第一个参数为int[] 第二个是下标i对应的内存偏移,第三个和第四个是旧值和新值
```



## Striped64 LongAdder

jdk8开始,对Long的原子操作,提供了LongAdder, LongAccumulator.

针对Double,提供了DoubleAdder, DoubleAccumulator,

![image-20200913202319128](https://gitee.com/shao_dw/pic/raw/master/upload/image-20200913202319128.png)

有了AtomicLong为啥还要LongAdder?

AtomicLong内部是一个volatile long型变量，由多个线程对这个变量进行CAS操作。多个线程同时对一个变量进行CAS操作，在高并发的场景下仍不够快

### LongAdder原理

把一个变量拆成多份，变为多个变量，有些类似于ConcurrentHashMap 的分段锁的例子

把一个Long型拆成一个base变量外加多个Cell，每个Cell包装了一个Long型变量。当多个线程并发累加的时候，如果并发度低，就直接加到base变量上；如果并发度高，冲突大，平摊到这些Cell上。在最后取值的时候，再把base和这些Cell求sum运算。

![image-20200913224131190](https://gitee.com/shao_dw/pic/raw/master/upload/image-20200913224131190.png)

```java
public long sum() {
        Cell[] as = cells; Cell a;
        long sum = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null) //学习下这种写法
                    sum += a.value;
            }
        }
        return sum;
    }
由于无论是long，还是double，都是64位的。但因为没有double型的CAS操作，所以是通过把double型转化成long型来实现的。所以，上面的base和cell[]变量，是位于基类Striped64当中的。英文Striped意为“条带”，也就是分片
    
    
```

### 最终一致性

在sum求和函数中，并没有对cells[]数组加锁。也就是说，一边有线程对其执行求和操作，一边还有线程修改数组里的值，也就是最终一致性，而不是强一致性

类似于ConcurrentHashMap 中的clear（）函数，一边执行清空操作，一边还有线程放入数据，clear（）函数调用完毕后再读取，hash map里面可能还有元素

因此，在LongAdder开篇的注释中，把它和AtomicLong 的使用场景做了比较。它适合高并发的统计场景，而不适合要对某个Long 型变量进行严格同步的场景。



### 伪共享与缓存行填充

```java
@sun.misc.Contended static final class Cell {}
jdk8之后的注解
    缓存与主内存进行数据交换的基本单位叫Cache Line（缓存行）
    64位架构,缓存行是64字节，也就是8个Long型的大小。这也意味着当缓存失效，要刷新到主内存的时候，最少要刷新64字节。
  
```

![image-20200913224731714](https://gitee.com/shao_dw/pic/raw/master/upload/image-20200913224731714.png)

伪共享:主存中有变量X、Y、Z（假设每个变量都是一个Long型），被CPU1和CPU2分别读入自己的缓存，放在了同一行Cache Line里面。当CPU1修改了X变量，它要失效整行Cache Line，也就是往总线上发消息，通知CPU 2对应的Cache Line失效。由于Cache Line是数据交换的基本单位，无法只失效X，要失效就会失效整行的Cache Line，这会导致Y、Z变量的缓存也失效

虽然只修改了X变量，本应该只失效X变量的缓存，但Y、Z变量也随之失效。Y、Z变量的数据没有修改，本应该很好地被CPU1和CPU2共享，却没做到，这就是所谓的“伪共享问题”。

题的原因是，Y、Z和X变量处在了同一行Cache Line里面。要解决这个问题，需要用到所谓的“缓存行填充”，分别在X、Y、Z后面加上7个无用的Long型，填充整个缓存行，让X、Y、Z处在三行不同的缓存行中

![image-20200913224828496](https://gitee.com/shao_dw/pic/raw/master/upload/image-20200913224828496.png)

以前是声明多个long型变量来实现缓存行填充的, 但是晦涩,jdk8使用该注解即可

这个地方给Node修饰,是为了不让cell[]数组中相邻元素落在同一缓存行



### LongAdder实现

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    if ((as = cells) != null || !casBase(b = base, b + x)) {//第一次尝试
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))//第二次尝试
            longAccumulate(x, null, uncontended);
    }
}

当一个线程调用add（x）的时候，首先会尝试使用casBase把x加到base变量上。如果不成功，则再用a.cas（..）函数尝试把x 加到Cell数组的某个元素上。如果还不成功，最后再调用longAccumulate（..）函数。
//casBase(b = base, b + x) 使用cas将x加到base变量上,
base赋值给了b, 使用cas(b,b+x),如果成功,b变为b+x

public void increment() {add(1L);}
public void decrement() {add(-1L);}

final boolean casBase(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
    }


final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            if ((as = cells) != null && (n = as.length) > 0) {
                if ((a = as[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = advanceProbe(h);
            }
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```