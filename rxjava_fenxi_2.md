#一.快速创建Observable
###1.from创建
看一个from的例子

```
        String[] name = new String[]{"huang", "pu", "hpdroid@yahoo.com"};
        Observable.from(name).subscribe(new Subscriber<String>() {
            @Override
            public void onCompleted() {
                Log.e("HP","onCompleted");
            }

            @Override
            public void onError(Throwable e) {
                e.printStackTrace();
            }

            @Override
            public void onNext(String s) {
                Log.e("HP",s);
            }
        });
```
打印结果

```
huang
pu
hpdroid@yahoo.com
onCompleted
```
from方法

```
  public final static <T> Observable<T> from(T[] array) {
        return from(Arrays.asList(array));
    }

    public final static <T> Observable<T> from(Iterable<? extends T> iterable) {
        return create(new OnSubscribeFromIterable<T>(iterable));
    }

 	public final static <T> Observable<T> create(OnSubscribe<T> f) {
        return new Observable<T>(hook.onCreate(f));
    }


```

一步一步跟进，又回到了最原始的创建模式

OnSubscribeFromIterable 实现了 OnSubscribe<T>，是OnSubscribe中的一种。

再次进入subscribe()方法，依旧调用了call()方法，而这里call()是OnSubscribeFromIterable类中的方法。

接下来看一下from是如何实现逐个打印的。

进入OnSubscribeFromIterable.call()

```
    @Override
    public void call(final Subscriber<? super T> o) {
        final Iterator<? extends T> it = is.iterator();
        if (!it.hasNext() && !o.isUnsubscribed())//没有下一个就执行onCompleted方法
            o.onCompleted();
        else 
            o.setProducer(new IterableProducer<T>(o, it));
    }
```
接着跟进 Subscriber 的 setProducer

```
    public void setProducer(Producer p) {
        long toRequest;
        boolean passToSubscriber = false;
        synchronized (this) {
            toRequest = requested;
            producer = p;
            if (subscriber != null) {
                // middle operator ... we pass thru unless a request has been made
                if (toRequest == NOT_SET) {
                    // we pass-thru to the next producer as nothing has been requested
                    passToSubscriber = true;
                }
            }
        }
        // do after releasing lock
        if (passToSubscriber) {
            subscriber.setProducer(producer);
        } else {
            // we execute the request with whatever has been requested (or Long.MAX_VALUE)
            if (toRequest == NOT_SET) {
                producer.request(Long.MAX_VALUE);
            } else {
                producer.request(toRequest);
            }
        }
    }

```

接着跟进 IterableProducer 的 request方法

```
        public void request(long n) {
            if (get() == Long.MAX_VALUE) {
                // already started with fast-path
                return;
            }
            if (n == Long.MAX_VALUE && compareAndSet(0, Long.MAX_VALUE)) {
                fastpath();
            } else 
            if (n > 0 && BackpressureUtils.getAndAddRequest(this, n) == 0L) {
                slowpath(n);
            }

        }
```

紧接着计入到fastpath()和slowpath()

```
     void slowpath(long n) {
            // backpressure is requested
            final Subscriber<? super T> o = this.o;
            final Iterator<? extends T> it = this.it;

            long r = n;
            while (true) {
                /*
                 * This complicated logic is done to avoid touching the
                 * volatile `requested` value during the loop itself. If
                 * it is touched during the loop the performance is
                 * impacted significantly.
                 */
                long numToEmit = r;
                while (true) {
                    if (o.isUnsubscribed()) {
                        return;
                    } else if (it.hasNext()) {
                        if (--numToEmit >= 0) {
                            o.onNext(it.next());
                        } else
                            break;
                    } else if (!o.isUnsubscribed()) {
                        o.onCompleted();
                        return;
                    } else {
                        // is unsubscribed
                        return;
                    }
                }
                r = addAndGet(-r);
                if (r == 0L) {
                    // we're done emitting the number requested so
                    // return
                    return;
                }

            }
        }

        void fastpath() {
            // fast-path without backpressure
            final Subscriber<? super T> o = this.o;
            final Iterator<? extends T> it = this.it;

            while (true) {
                if (o.isUnsubscribed()) {
                    return;
                } else if (it.hasNext()) {
                    o.onNext(it.next());
                } else if (!o.isUnsubscribed()) {
                    o.onCompleted();
                    return;
                } else {
                    // is unsubscribed
                    return;
                }
            }
        }
```

终于看到了咱们的onNext()方法了。到这里，from的调用也算走通了。




#二.线程之间自由切换