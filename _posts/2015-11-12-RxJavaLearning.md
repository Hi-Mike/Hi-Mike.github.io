---
layout: post
title: "我看到的RxJava"
date: 2015-11-12
categories:
---

>从来没有努力到拼天赋的程度

接触RxJava才短短的几个月，甚是喜欢。虽然平时写点简单的还行，稍微有点挑战便力不从心，用的不好，出了bug也是常有的事。

后来看到 **扔物线** 写的 [给Android开发者的RxJava详解](http://gank.io/post/560e15be2dca930e00da1083) ,受益颇多！心想如果不自己捋一捋，难免心有芥蒂，何况其中的来龙去脉确实值得推敲一番。另外 [深入浅出RxJava(译)](http://blog.csdn.net/lzyzsd/article/details/41833541) 也是很好的资料，原文出处 [Dan Lew Codes](http://blog.danlew.net/) 更值得推荐，不得不感叹学习英文的重要性。

### 基本原理

话不多说，第一视角呈现我看到的RxJava，做个备忘。所用例子没有实际意义，重在了解内在结构。先来一个最简单的。

```java
	Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("1");
            }
        }).subscribe(new Observer<String>() {
            @Override
            public void onCompleted() {
            }

            @Override
            public void onError(Throwable e) {
            }

            @Override
            public void onNext(String s) {
				Log.d(TAG,s);//1
            }
        });
```

这个样子，一个订阅便好了。顺便一说，RxJava，简洁的异步流也。

直接来看调用过程。

```java
	public final static <T> Observable<T> create(OnSubscribe<T> f) {
        return new Observable<T>(hook.onCreate(f));
    }

	protected Observable(OnSubscribe<T> f) {
        this.onSubscribe = f;
    }

    private static final RxJavaObservableExecutionHook hook = RxJavaPlugins.getInstance().getObservableExecutionHook();
```

当 **Observable.create** 的时候创建了 **Observable** 对象，然后返回的 **Observable** 对象订阅了 **观察者** ，看 **subscribe** 方法：

```java
	public final Subscription subscribe(final Observer<? super T> observer) {
        if (observer instanceof Subscriber) {
            return subscribe((Subscriber<? super T>)observer);
        }
        return subscribe(new Subscriber<T>() {

            @Override
            public void onCompleted() {
                observer.onCompleted();
            }

            @Override
            public void onError(Throwable e) {
                observer.onError(e);
            }

            @Override
            public void onNext(T t) {
                observer.onNext(t);
            }

        });
    }
```

发现 **Observer** 对象都会被包装为 **Subscriber** 对象， **Subscriber** 实现了 **Observer** ，并增加了 **onStart()** 回调。最后调用：

```java
	private static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
     	.........
        .....
        // new Subscriber so onStart it
        subscriber.onStart();
        
        /*
         * See https://github.com/ReactiveX/RxJava/issues/216 for discussion on "Guideline 6.4: Protect calls
         * to user code from within an Observer"
         */
        // if not already wrapped
        if (!(subscriber instanceof SafeSubscriber)) {
            // assign to `observer` so we return the protected version
            subscriber = new SafeSubscriber<T>(subscriber);
        }

        try {
            hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
            return hook.onSubscribeReturn(subscriber);
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            try {
                subscriber.onError(hook.onSubscribeError(e));
            } catch (OnErrorNotImplementedException e2) {
                throw e2;
            } catch (Throwable e2) {
                RuntimeException r = new RuntimeException("Error occurred attempting to subscribe [" + e.getMessage() + "] and then again while trying to pass to onError.", e2);
                hook.onSubscribeError(r);
                throw r;
            }
            return Subscriptions.unsubscribed();
        }
    }
```

查看源码可知： **hook.onSubscribeStart** 默认实现返回了 **observable.onSubscribe** 调用 **call** 方法，而这个 **onSubscribe** 正是我们创建 **Observable** 对象时的那个 **OnSubscribe** 对象。整个调用的过程其实就完了。从这里也可以知道RxJava的调用是从 **subscribe** 方法订阅时触发的。

下面来看一下当增加了操作符的情况：

```java
	Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("1");
            }
        }).map(new Func1<String, Integer>() {
            @Override
            public Integer call(String s) {
                return Integer.valueOf(s);
            }
        }).subscribe();
```

 **创建** 和 **订阅** 我们已经清楚了，直接看 **map** 操作符的实现：

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
                        Exceptions.throwIfFatal(e);
                        st.onError(e);
                    }
                } catch (Throwable e) {
                    Exceptions.throwIfFatal(e);
                    o.onError(e);
                }
            }
        });
    }
```

 **map** 操作符中又用 **OperatorMap** 对我们传入的实现对象做了包装，包装的结果是：当调用 **OperatorMap** 对象的 **call** 方法时，其实就是调用了传入对象的 **call** 方法。

再看 **lift** 方法，里面重新创建了一个 **Observable** 对象，看到这里，先去 **扔物线** 那里去盗图，貌似理解了。

![RxJava lift](http://7xnzl2.com1.z0.glb.clouddn.com/rxjava_lift.png)

也就是说，当 **subscribe** 订阅时，会最先调用最后 **lift** 返回的 **Observable** 对象，然而这个对象又会在 **call** 方法中先调用上一个 **Observable** 对象所包含的 **call** 方法。最终的效果就是，先 **lift** 先调用，水往低处流。

这里需要注意的是 **onSubscribe.call(st);** 中的 **onSubscribe** 与 **st** 所属的域， **lift** 中是新创建了 **Observable** 对象， **onSubscribe** 应该是属于新的 **Observable** 对象的。 **st** 是从旧的 **Observable** 对象中的 **Operator** 调用 **call** 方法返回的（这里就是 **OperatorMap** 中的 **call** 方法的返回），返回的 **Subscriber** 对象 **st** 会被继续往下传递，形成了一条不断的 **流** 。

RxJava的基本原理大概就是这样，至于其中操作符的具体实现，那又是另外一个问题了，还需要不断的学习。。。。

### RxJava的核心：异步——线程控制

头都整昏了

### subjects

这个有时间得看看