---
layout: post
title: RxJava源码浅析
---

最近项目中准备引入RxJava库，研究了一下源码，目前还没有正式投入使用，文中如有纰漏的地方，欢迎各位大神指正。

##1.概念##
本文中会涉及到一些自定义的概念，先列在前面。

Observable: 可被订阅者（缩写OB）

Subscriber: 订阅者（缩写SUB）

订阅: Observable.subscribe函数或者类似于该函数的行为。

Chain:  Observable被触发之后的代码逻辑执行路径。

操作: Observable被触发之后到订阅者处理之前所经历的变换（lift）。

##2.订阅者VS被订阅者##

订阅者和被订阅者属于观察者模式中的两大核心概念，被订阅者产生事件，订阅者处理事件，事件的触发往往由第三方完成。在RxJava中两者不在局限于狭义的事件，其处理的可以使任何一段代码逻辑，触发"事件"的行为往往由**订阅**完成。

那么RxJava中如何创建Observable和Subscriber呢？先上代码:

```java
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("HelloWorld");
        subscriber.onCompleted();
    }
});
```
在上面的代码中，创建了一个**可被订阅者**，其执行的逻辑是像其订阅者传递一个"HelloWorld"字符串。那继续看Observable.create的源码:

```java
public final static <T> Observable<T> create(OnSubscribe<T> f) {
    return new Observable<T>(hook.onCreate(f));
}
```

create方法接受一个OnSubscribe类型的参数，顾名思义该参数表示当**订阅**行为发生时执行的操作，返回一个Observable对象。

*hook:用于对**被订阅者**的生命周期进行拦截处理，默认hook不进行任何处理，代码详见RxJavaObservableExecutionHookDefault.java*

继续看Observable的构造函数

```java
protected Observable(OnSubscribe<T> f) {
    this.onSubscribe = f
}
```

在Observable的构造函数中，只是简单地存储了onSubscribe对象。至此一个**可被订阅者**就创建完成了。有了Observable对象之后就可以开始**订阅**行为了

```java
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("HelloWorld");
        subscriber.onCompleted();
    }}).subscribe(new Subscriber<Object>() {
    @Override
    public void onCompleted() {}
    @Override
    public void onError(Throwable throwable) {}
    @Override
    public void onNext(Object o) {
        System.out.println(o.toString());
    }});
```

当执行**订阅**操作时，需要传递一个Subscriber对象，对于接收和处理Observable产生的"事件"。

```java
private static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
    ...省略代码...
    subscriber.onStart();
    if (!(subscriber instanceof SafeSubscriber)) {
        subscriber = new SafeSubscriber<T>(subscriber);
    }
    try {
        hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
        return hook.onSubscribeReturn(subscriber);
    } catch (Throwable e) {
        ...省略代码...
        return Subscriptions.unsubscribed();
    }
}
```

Subscribe函数中会先调用onStart，然后转换为SafeSubscriber,并作为参数传递给onSubscribe的call函数，并传递Subscriber,对象，此处的onSubscribe即是create函数中new的onSubscribe，所以当其执行call函数时，自然调用到订阅者的onNext。

至此为止RxJava的一次简单使用已经完成，但是然并卵，这种特性和直接使用Callback并没有多大差别，那么RxJava的NB之处怎么体现呢？这就需要进入下一个主题，**操作**

##3.操作##
在上面的例子中Observer产生了一个Message（“HelloWorld”）,Subscriber对Message的处理方式就是把其打印出来，如果我们需要对Message进行加工怎么处理呢？这就需要用到神器**map**。

```java
	Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("HelloWorld");
                subscriber.onCompleted();
            }
        }).map(new Func1<String, Object>() {
            @Override
            public Object call(String s) {
                return new StringBuilder(s).reverse();
            }
        }).subscribe(new Subscriber<Object>() {
            ...省略代码...
        });
```

那map函数做了哪些处理呢？

```java
    public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
        return lift(new OperatorMap<T, R>(func));
    }

    public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
        return new Observable<R>(new OnSubscribe<R>() {
            @Override
            public void call(Subscriber<? super R> o) {
                try {
                    Subscriber<? super T> st = hook.onLift(operator).call(o);
                    try {
                        st.onStart();
                        onSubscribe.call(st);
                    } catch (Throwable e) {
                        if (e instanceof OnErrorNotImplementedException) {
                            throw (OnErrorNotImplementedException) e;
                        }
                        st.onError(e);
                    }
                } catch (Throwable e) {
                    if (e instanceof OnErrorNotImplementedException) {
                        throw (OnErrorNotImplementedException) e;
                    }
                    o.onError(e);
                }
            }
        });
    }
```

此处出现了RxJava中比较核心的概念，lift, 其返回了一个新的Observable对象，为方便区分, 新的Observable对象称之为OB’， create接口返回的Observable对象命名为OB，也就意味着，我们最终的订阅者是订阅OB‘的，按之前的理解，当subscribe行为发生时，会触发执行Observable.onSubscribe的call函数，即上面代码中的call函数。***注意此处的call中的参数o，使我们在subscribe函数中创建的Subscriber对象(命名为SUB)***上面的代码中先调用operator的call函数，传递SUB获取一个Subscriber对象SUB’，那SUB‘和SUB是啥关系呢？我们先看Operator的类型，在map函数中，先用我们创建的转换函数Func1构建了OperatorMap,然后调用lift，此处的operator的实际类型为OperatorMap,所以我们的目标转移到OperatorMap的call函数。

```java
public final class OperatorMap<T, R> implements Operator<R, T> {

    private final Func1<? super T, ? extends R> transformer;

    public OperatorMap(Func1<? super T, ? extends R> transformer) {
        this.transformer = transformer;
    }

    @Override
    public Subscriber<? super T> call(final Subscriber<? super R> o) {
        return new Subscriber<T>(o) {

            @Override
            public void onCompleted() {
                o.onCompleted();
            }

            @Override
            public void onError(Throwable e) {
                o.onError(e);
            }

            @Override
            public void onNext(T t) {
                try {
                    o.onNext(transformer.call(t));
                } catch (Throwable e) {
                    Exceptions.throwIfFatal(e);
                    onError(OnErrorThrowable.addValueAsLastCause(e, t));
                }
            }

        };
    }

}
```
***注意此处创建了SUB’，当SUB‘被通知时（onNext被调用），先调用转换函数Func1进行处理，然后将处理的结果通知给SUB。***

回到lift的代码中，获取SUB’之后，调用OB的onSubscribe的call函数，并传递了SUB‘。对比subscribe函数，此处即触发了对OB的一次“订阅行为”，即用SUB’订阅OB。综上，最终的执行路线如下:


![流程1](http://fmitachi.github.io/images/1.png “流程1”)

不带操作的订阅流程

![流程2](http://fmitachi.github.io/images/2.png “流程2”)

使用map之后的执行流程

![流程3](http://fmitachi.github.io/images/3.png “流程3”)

即map操作生成了一对代理OBProxy/SUBProxy,OBProxy用于接受真正的订阅，SUBProxy用于监听原本被观察者的事件。
下面我们扩展到两个map的情况，每一次map操作会产生一个新的OB和新的Sub。

```java
	Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("HelloWorld");
                subscriber.onCompleted();
            }
        }).map(new Func1<String, Object>() {
            @Override
            public Object call(String s) {
                return new StringBuilder(s).reverse();
            }
        }).map(new Func1<Object, Object>() {
            @Override
            public Object call(Object s) {
                return new StringBuilder(s.toString()).reverse();
            }
        }).subscribe(new Subscriber<Object>() {
            ...省略代码...
        });

```
其执行流程如下：

![流程4](http://fmitachi.github.io/images/4.png “流程4”)


##4.异步##
搞定了operator和lift之后，再来看线程调度就比较简单了，RxJava中的线程调度主要依赖于Scheduler完成。那如何将指定的operator放到特定的线程池中执行呢？RxJava提供两种方式：**observerOn**和**subscribeOn**,先看observerOn。

```java
    public final Observable<T> observeOn(Scheduler scheduler) {
        if (this instanceof ScalarSynchronousObservable) {
            return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
        }
        return lift(new OperatorObserveOn<T>(scheduler));
    }
```
其逻辑和普通的map操作一致，由此可知，线程调度相关的工作应由OperatorObserveOn.call返回的SUBProxy控制

```java
    public Subscriber<? super T> call(Subscriber<? super T> child) {
        if (scheduler instanceof ImmediateScheduler) {
            // avoid overhead, execute directly
            return child;
        } else if (scheduler instanceof TrampolineScheduler) {
            // avoid overhead, execute directly
            return child;
        } else {
            ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child);
            parent.init();
            return parent;
        }
    }
```
此处返回的SUBProxy是ObserveOnSubscriber

```java
	 @Override
        public void onNext(final T t) {
            if (isUnsubscribed()) {
                return;
            }
            if (!queue.offer(on.next(t))) {
                onError(new MissingBackpressureException());
                return;
            }
            schedule();
        }
```

在其onNext中果然发现了目标schedule函数。

```java
	protected void schedule() {
            if (COUNTER_UPDATER.getAndIncrement(this) == 0) {
                recursiveScheduler.schedule(action);
            }
        }
```

schedule的任务又传递给了recursiveScheduler,这个是怎么乱入的，action又是干啥的？继续查看构造函数

```java
	public ObserveOnSubscriber(Scheduler scheduler, Subscriber<? super T> child) {
            this.child = child;
            this.recursiveScheduler = scheduler.createWorker();
            if (UnsafeAccess.isUnsafeAvailable()) {
                queue = new SpscArrayQueue<Object>(RxRingBuffer.SIZE);
            } else {
                queue = new SynchronizedQueue<Object>(RxRingBuffer.SIZE);
            }
            this.scheduledUnsubscribe = new ScheduledUnsubscribe(recursiveScheduler);
        }
```

***注意此处的child即为SUB***

recursiveScheduler是通过最外层选择的Scheduler创建出来的，so...


```java
public final class NewThreadScheduler extends Scheduler {

    private static final String THREAD_NAME_PREFIX = "RxNewThreadScheduler-";
    private static final RxThreadFactory THREAD_FACTORY = new RxThreadFactory(THREAD_NAME_PREFIX);
    private static final NewThreadScheduler INSTANCE = new NewThreadScheduler();

    /* package */static NewThreadScheduler instance() {
        return INSTANCE;
    }

    private NewThreadScheduler() {

    }

    @Override
    public Worker createWorker() {
        return new NewThreadWorker(THREAD_FACTORY);
    }
}
```

继续找NewThreadWorker.schedule最终会调用到scheduleActual

```java
    public ScheduledAction scheduleActual(final Action0 action, long delayTime, TimeUnit unit) {
        Action0 decoratedAction = schedulersHook.onSchedule(action);
        ScheduledAction run = new ScheduledAction(decoratedAction);
        Future<?> f;
        if (delayTime <= 0) {
            f = executor.submit(run);
        } else {
            f = executor.schedule(run, delayTime, unit);
        }
        run.add(f);

        return run;
    }
```
外层传递的action会被包装为ScheduledAction，提交Java的线程池给executor执行，在其run方法内会执行action的call函数。

```java
    final Action0 action = new Action0() {

        @Override
        public void call() {
            pollQueue();
        }

    };

    void pollQueue() {
        int emitted = 0;
        do {
            
            for (;;) {
                ...省略代码...
                if (r > 0) {
                    Object o = queue.poll();
                    if (o != null) {
                        child.onNext(on.getValue(o));
                        r--;
                        emitted++;
                        produced++;
                    } else {
                        break;
                    }
                } else {
                    break;
                }
            }
            ...省略代码...
        } while (COUNTER_UPDATER.decrementAndGet(this) > 0);
        if (emitted > 0) {
            request(emitted);
        }
    }
```
至此，又回到了SUB上，唯一的区别就是SUB.onNext是在线程池中执行，而非创建SUB的线程中执行。值得注意的是，一旦执行一次observerOn之后，后续的逻辑都是在Scheduler指定的线程上运行的，直到再次调用observerOn或则流程运行结束。

![流程5](http://fmitachi.github.io/images/5.png “流程5”)

另外一种线程调度的方式是subscribeOn，那subscribeOn是 怎么执行的呢？它和observerOn有什么区别呢？
继续上源码

```java
    public final Observable<T> subscribeOn(Scheduler scheduler) {
        if (this instanceof ScalarSynchronousObservable) {
            return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
        }
        return nest().lift(new OperatorSubscribeOn<T>(scheduler));
    }
```

和observerOn的代码很神似，但是注意有两处区别：**nest**和**OperatorSubscribeOn**

```java
    public final Observable<Observable<T>> nest() {
        return just(this);
    }

    public final static <T> Observable<T> just(final T value) {
        return ScalarSynchronousObservable.create(value);
    }

```

同样nest创建了一个OB‘,只是其类型是ScalarSynchronousObservable，并且把OB作为参数传递给构造函数

```java
    protected ScalarSynchronousObservable(final T t) {
        super(new OnSubscribe<T>() {

            @Override
            public void call(Subscriber<? super T> s) {
                s.onNext(t);
                s.onCompleted();
            }

        });
        this.t = t;
    }
```

在ScalarSynchronousObservable中创建了OB’,当OB‘**被订阅**的时候，把OB作为参数传递给了OB’的订阅者，那OB‘的订阅者是谁呢？会是外层创建的SUB么？答案是否定的，因为SUB的onNext不能接受Observable类型的参数。回顾之前lift函数中会产生一次订阅操作，那么此处的订阅者应该是lift中从operator获取的SUB’，那OperatorSubscribeOn会生成一个怎样的订阅者呢？

```java
public Subscriber<? super Observable<T>> call(final Subscriber<? super T> subscriber) {
        final Worker inner = scheduler.createWorker();
        subscriber.add(inner);
        return new Subscriber<Observable<T>>(subscriber) {

            @Override
            public void onCompleted() {
                // ignore because this is a nested Observable and we expect only 1 Observable<T> emitted to onNext
            }

            @Override
            public void onError(Throwable e) {
                subscriber.onError(e);
            }

            @Override
            public void onNext(final Observable<T> o) {
                inner.schedule(new Action0() {

                    @Override
                    public void call() {
                        final Thread t = Thread.currentThread();
                        o.unsafeSubscribe(new Subscriber<T>(subscriber) {

                            @Override
                            public void onCompleted() {
                                subscriber.onCompleted();
                            }

                            @Override
                            public void onError(Throwable e) {
                                subscriber.onError(e);
                            }

                            @Override
                            public void onNext(T t) {
                                subscriber.onNext(t);
                            }

                            @Override
                            public void setProducer(final Producer producer) {
                                subscriber.setProducer(new Producer() {

                                    @Override
                                    public void request(final long n) {
                                        if (Thread.currentThread() == t) {
                                            // don't schedule if we're already on the thread (primarily for first setProducer call)
                                            // see unit test 'testSetProducerSynchronousRequest' for more context on this
                                            producer.request(n);
                                        } else {
                                            inner.schedule(new Action0() {

                                                @Override
                                                public void call() {
                                                    producer.request(n);
                                                }
                                            });
                                        }
                                    }

                                });
                            }

                        });
                    }
                });
            }

        };
    }
```

注意SUB‘的onNext函数，会先执行线程切换，然后对OB进行订阅操作，并在其订阅者的onNext中把结果传递给SUB，从而回到**Chain**上。综上，执行流程为：

![流程6](http://fmitachi.github.io/images/6.png “流程6”)

对比两种方式的执行流程，observerOn在切换线程之前所有的订阅行为已经发生，在执行**Chain**的过程中切换线程，subscribeOn则是切换线程后发生对OB的订阅从而进入**Chain**。所以对于observerOn每执行一次，其后续的Chain切换到另一条线程上执行，但是由于订阅行为已经发生，故其无法指定OB的执行线程;而对于后者，由于其线程切换发生在OB的订阅执行之前，所以其可以指定给OB指定线程，但是无论调用多少次，只有第一次会生效。

##5.结语##
本文知识对RxJava源码的匆匆一瞥，在实际的项目应用中，可以根据自己的需求选择一些封装库，RxBinding等,另外还有诸如flatMap、contactMap以及剩余几种Scheduler的原理，大家可以自行分析源码。