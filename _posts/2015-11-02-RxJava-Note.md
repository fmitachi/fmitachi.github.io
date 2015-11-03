---
layout: post
title: RxJava源码浅析
---

最近项目中准备引入RxJava库，研究了一下源码，目前还没有正式投入使用，文中如有纰漏的地方，欢迎各位大神指正。

##1.概念##
本文中会涉及到一些自定义的概念，先列在前面。

Observable: 可被订阅者

Subscriber: 订阅者

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


##4.异步##

##5.结语##
