## JDK9响应式编程

### java.util.concurrent.Flow

| 接口         | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| Publisher    | 通过该接口发布一个元素序列给有需要的消费者                   |
| Subscriber   | 每一个订阅者(消费者)从Publisher获取所需的元素来进行消费      |
| Subscription | Subscription属于连接Subscriber和Publisher的中间人,在订阅者请求的时候需要用到它(在Subscriber请求元素消费时和Subscriber不再需要元素时) |
| Processor    | 可能需要一个既是生产者又是消费者的角色,Processor扮演了同时属于Subscriber和Publisher的角色 |



![image-20200912150232569](https://gitee.com/shao_dw/pic/raw/master/upload/image-20200912150232569.png)

Publisher用于发布元素，并将元素推送给Processor。Processor通过调用Subscription：：request方法来从Publisher请求元素,而这个动作是在Processor中进行的（此时Processor是作为Subscriber角色存在的），所以箭头指向左；Subscriber：：onNext接收并消费元素的动作是在Subscription中进行的，所以箭头指向右。

Processor再将元素推送给Subscriber，Subscriber通过使用Subscriber：：onNext方法来接收元素

![image-20200912151025729](https://gitee.com/shao_dw/pic/raw/master/upload/image-20200912151025729.png)



Flow.Publisher<T>:

+ subscribe(Subscriber<? super T> subscriber) 如果可能的话，这个方法需要添加一个给定的Flow.Subscriber，如果尝试订阅失败，那么会调用Flow.Subscriber的onError方法来发出一个IllegalStateException类型异常。否则，Flow.Subscriber<T>会调用onSubscribe方法，同时传入一个Flow.Subscription，Subscriber通过调用其所属的Flow.Subscription的request方法来获取元素，也可以调用它的cancel方法来解除订阅。

Flow.Subscriber<T>:

+ void onSubscribe(Subscription subscription) ：在给定的Subscription想要使用Subscriber其他方法的前提下，必须先调用这个方法。
+ void onError(Throwable throwable)：当Publisher或者Subscription遇到了不可恢复的错误时，调用此方法，然后Subscription就不能再调用Subscriber的其他方法了。
+ void onNext(T item)：获取Subscription的下一个元素。
+ void onComplete：在调用这个方法后，Subscription就不能再调用Subscriber的其他方法了。

Flow.Subscription<T>:

+ void cancel：调用这个方法造成的直接后果是Subscription会停止接收信息。
+ void request(long n)：Subscription调用这个方法添加 n个元素。如果 n小于0，Subscriber将收到一个onError信号。如果n等于0，那么调用complete方法，否则调用onNext(T item)方法。

Flow.Processor<T，R>是Subscriber<T>、Publisher<R>的集合体 

```java
Processor<T,R> extends Subscriber<T>, Publisher<R>
```