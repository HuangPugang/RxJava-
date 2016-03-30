#####扔物线大大的文章确实写的牛 [扔物线](http://gank.io/post/560e15be2dca930e00da1083)，看了他的文章受益匪浅，文中很多会引用到他的一些分析，没有看过他的文章的建议先看一下。
#一.概述
###先简单介绍一下RxJava的思想
RxJava 有四个基本概念：Observable (可观察者，即被观察者)、 Observer (观察者)、 subscribe (订阅)、事件。Observable 和 Observer 通过 subscribe() 方法实现订阅关系，从而 Observable 可以在需要的时候发出事件来通知 Observer。

###概念介绍
####1.Observable（可观察者，即被观察者）
事件的触发者。
####2.Observer/Subscriber（观察者）
事件的产生者
####3.subscribe(订阅)
可被观察者和观察者之间的桥梁
####4.事件
产生的事件
####5.总结
举个例子：我是一名读者杂志会员，我想订阅读者期刊，当我订阅之后，读者工作室就会每个月给我发一本读者杂志。在这个事件中，我就是一名被观察者，读者工作室就是观察者，因为我一旦产生订阅这件事，就会触发读者工作室的一系列动作。

#二.实例解析
###1.最简单的例子
observable(可被观察者)

```
        Observable observable = Observable.create(new Observable.OnSubscribe<String>() {

            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("test1");
                subscriber.onNext("test2");
                subscriber.onCompleted();
            }
        });
```
subscriber/observer(观察者)

```
    private Subscriber<String> subscribe = new Subscriber<String>() {
        @Override
        public void onCompleted() {
			Log.e("HP", "onCompleted");
        }

        @Override
        public void onError(Throwable e) {

        }

        @Override
        public void onNext(String s) {
            Log.e("HP", s);
        }
    };
```
subscribe(订阅)

```
observable.subscribe(subscribe);
```

执行以上代码会打印出如下结果

```
test1
test2
onCompleted
```

这样一个最简单的RxJava代码就完成了。

为什么在call方法中调用onNext(),onCompleted()会触发subscriber/observer中对应的方法呢？接下来一起看一下源码，看看是如何订阅成功的。

进入到Observable.create()方法

```
 public final static <T> Observable<T> create(OnSubscribe<T> f) {
        return new Observable<T>(hook.onCreate(f));
    }
```

```
    protected Observable(OnSubscribe<T> f) {
        this.onSubscribe = f;
    }
```

这里创建了一个Observable<T>对象，同时通过构造函数将 f 赋值给Observalbe类中的onSubscribe

进入到subscribe方法 

```
    public final Subscription subscribe(Subscriber<? super T> subscriber) {
        return Observable.subscribe(subscriber, this);
    }
    
    private static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
     // validate and proceed
        if (subscriber == null) {
            throw new IllegalArgumentException("observer can not be null");
        }
        if (observable.onSubscribe == null) {
            throw new IllegalStateException("onSubscribe function can not be null.");
            /*
             * the subscribe function can also be overridden but generally that's not the appropriate approach
             * so I won't mention that in the exception
             */
        }
        
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

        // The code below is exactly the same an unsafeSubscribe but not used because it would 
        // add a significant depth to already huge call stacks.
        try {
            // allow the hook to intercept and/or decorate
            hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
            return hook.onSubscribeReturn(subscriber);
        } catch (Throwable e) {
            // special handling for certain Throwable/Error/Exception types
            Exceptions.throwIfFatal(e);
            // if an unhandled error occurs executing the onSubscribe we will propagate it
            try {
                subscriber.onError(hook.onSubscribeError(e));
            } catch (Throwable e2) {
                Exceptions.throwIfFatal(e2);
                // if this happens it means the onError itself failed (perhaps an invalid function implementation)
                // so we are unable to propagate the error correctly and will just throw
                RuntimeException r = new RuntimeException("Error occurred attempting to subscribe [" + e.getMessage() + "] and then again while trying to pass to onError.", e2);
                // TODO could the hook be the cause of the error in the on error handling.
                hook.onSubscribeError(r);
                // TODO why aren't we throwing the hook's return value.
                throw r;
            }
            return Subscriptions.unsubscribed();
        }
    }

```
我们看到

```
hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
```
其中hook.onSubscribeStart返回的是create()方法中创建的OnSubscribe<T>对象，即上文中提到的 onSubscribe ,在这里调用OnSubscribe 中的call方法，将subscriber/observer传递到call方法中，所以我们在call中调用onNext(),onError(),onCompleted()会触发observer中对应的方法，从而达到了事件通知的效果。


###2.订阅Action
观察者除了我们的subscriber/observer之外，还可以是Action

再看一个例子
定义一个action

```
    private Action1<String> action1 = new Action1<String>() {
        @Override
        public void call(String s) {
            Log.e("HP",s);
        }
    };
```

```
    observable.subscribe(action1);

```

再看输出结果

```
test1
test2
```
不出所料，输出结果与预期的相同。
进入subscribe方法看以看究竟

```
public final Subscription subscribe(final Action1<? super T> onNext) {
        if (onNext == null) {
            throw new IllegalArgumentException("onNext can not be null");
        }

        return subscribe(new Subscriber<T>() {

            @Override
            public final void onCompleted() {
                // do nothing
            }

            @Override
            public final void onError(Throwable e) {
                throw new OnErrorNotImplementedException(e);
            }

            @Override
            public final void onNext(T args) {
                onNext.call(args);
            }

        });
```

原来在subscribe方法中将我们的action转换成了subscribe。
所以通过action你可以自定义一些实现onNext(),onError(),onComplete()方法的action，这些action通常是可以作为公用的操作。

这些仅仅是RxJava最基础用法的一个解析。









