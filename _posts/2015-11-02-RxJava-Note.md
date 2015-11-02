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

那么RxJava中如何创建Observable和Subscriber呢？

```java
    Observable.create(new Observable.OnSubscribe<String>() {
        @Override
        public void call(Subscriber<? super String> subscriber) {
            subscriber.onNext("HelloWorld");
            subscriber.onCompleted();
        }
    }).
```

##3.操作##


##4.异步##

##5.结语##
