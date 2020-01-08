+++
title = "RxJava magic (finally) goes away"
draft = false
date = "2015-12-08"
tags = ["Java", "Android"]
+++

There is too much "magic" inside of RxJava.

Docs and tutorials are either too shallow or too complicated.

Here is the middle ground that is *essential* to know before playing with RxJava.

In these tests we will take each RxJava part... apart and explore how exactly it works.

<!--more-->

Inspired by:

* [Make the Magic go away.](https://blog.8thlight.com/uncle-bob/2015/08/06/let-the-magic-die.html)
* [Reactive Programming](https://www.youtube.com/watch?v=3bAQXTVsEiQ)

### Basic usage pattern

Let's start from a familiar example. We will take it apart later.

``` java
AtomicInteger received = new AtomicInteger();

Observable.just(1)
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer integer) {
            received.set(integer);
        }
    });

assertEquals(1, received.get());
```

### Implementing own `Observable` and `Subscriber`

We will now implement our own `just(1)` `Observable` and take a brief look at `Observable.subscribe`.

``` java
AtomicBoolean emitted = new AtomicBoolean();

// Creating Observable the usual way:
Observable<Integer> observable = Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        subscriber.onNext(1);
        subscriber.onCompleted();
        emitted.set(true);
    }
});

assertFalse(emitted.get());
// Nothing happens at this point! Observable does not emit values by itself,
// we need to subscribe first.

AtomicInteger received = new AtomicInteger();

// First, Observable.unsafeSubscribe is a simplified version of Observable.subscribe,
// so we will use it instead.
// Second, Observable.subscribe(Action1) is just a shortcut for
// Observable.subscribe(Subscriber), here we will use Subscriber directly.

observable
    .unsafeSubscribe(new Subscriber<Integer>() {
        @Override
        public void onCompleted() {}

        @Override
        public void onError(Throwable e) {
            // We will rethrow the exception to not ignore asserts.
            throw new RuntimeException(e);
        }

        @Override
        public void onNext(Integer it) {
            received.set(it);
        }
    });

assertTrue(emitted.get());
assertEquals(1, received.get());
// The value has been emitted and received after the subscription.
```

### `Observable.create` and `Observable.unsafeSubscribe`

We will now take apart `Observable.create` and `Observable.unsafeSubscribe`.

``` java
Observable.OnSubscribe<Integer> onSubscribe = new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        subscriber.onNext(1);
        subscriber.onCompleted();
    }
};

// This is how Observable.create(OnSubscribe<T> f) looks like (simplified),
// so we don't need to "implement" it:
//  protected Observable(OnSubscribe < T > f) {
//      this.onSubscribe = f;
//  }

AtomicInteger received = new AtomicInteger();

Subscriber<Integer> subscriber = new Subscriber<Integer>() {
    @Override
    public void onCompleted() {}

    @Override
    public void onError(Throwable e) {
        throw new RuntimeException(e);
    }

    @Override
    public void onNext(Integer it) {
        received.set(it);
    }
};

// This is how Observable.unsafeSubscribe looks like (simplified):
try {
    onSubscribe.call(subscriber);
}
catch (Throwable e) {
    subscriber.onError(e);
}

assertEquals(1, received.get());
// onNext still works!
```

A little resume: `OnSubscribe` function sends data directly to a given `Subscriber` by
calling it's methods.

Note: `Observable` was not used in this test at all.

### What `Subscription` is

`Subscription` is a simple interface with just two methods:

``` java
void unsubscribe();
boolean isUnsubscribed();
```

``` java
AtomicInteger received = new AtomicInteger();

Subscriber<Integer> subscriber = new Subscriber<Integer>() {
    @Override
    public void onCompleted() {}

    @Override
    public void onError(Throwable e) {
        throw new RuntimeException(e);
    }

    @Override
    public void onNext(Integer integer) {
        received.set(integer);
    }
};

Observable<Integer> observable = Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> targetSubscriber) {

        assertEquals(subscriber, targetSubscriber); 
        // subscriber and targetSubscriber are the same object.

        assertFalse(subscriber.isUnsubscribed());
        // subscriber is subscribed, this is OK.
        // All subscriptions are created in the subscribed state.

        targetSubscriber.onNext(1);
        targetSubscriber.onCompleted();
    }
});

Subscription subscription = observable.unsafeSubscribe(subscriber);

assertEquals(subscription, subscriber);
// Surprise! Subscription and Subscriber are the same object.
// Subscriber implements Subscription interface.

assertFalse(subscriber.isUnsubscribed());
// ...and it is still subscribed even after the onCompleted() call... wait, WHY???

assertEquals(1, received.get());
// onNext still works!
```

### Regular `Observable.subscribe` and `SafeSubscriber`

The regular version of `Observable.subscribe(...)` wraps `Subscriber`
into `SafeSubscriber` and automatically enforces the lifecycle event order.

For example, `SafeSubscriber` automatically calls `unsubscribe()` after `onComplete()` has been received.

``` java
AtomicInteger received = new AtomicInteger();

Subscriber<Integer> subscriber = new Subscriber<Integer>() {
    @Override
    public void onCompleted() {}

    @Override
    public void onError(Throwable e) {
        throw new RuntimeException(e);
    }

    @Override
    public void onNext(Integer integer) {
        received.set(integer);
    }
};

Observable<Integer> observable = Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> targetSubscriber) {

        assertTrue(targetSubscriber instanceof SafeSubscriber);
        // The regular subscribe() gave us a SafeSubscriber instance.

        targetSubscriber.onNext(1);
        targetSubscriber.onCompleted();
    }
});

// This time we will use the regular subscribe():
Subscription subscription = observable.subscribe(subscriber);

assertNotEquals(subscription, subscriber);
// Subscription and Subscriber are NOT the same object now.

assertTrue(subscriber.isUnsubscribed());
// ...SafeSubscriber in action.

assertEquals(1, received.get());
// onNext still works! :)
```

### Chaining operators

This is a simple exploration of chaining operators.

``` java
Observable<Integer> observable = Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {

        // the received subscriber is some internal class inside of OperatorTake.
        assertTrue(subscriber.getClass().getName().contains("OperatorTake"));

        subscriber.onNext(1);
        subscriber.onNext(2);
        subscriber.onNext(3);
        subscriber.onCompleted();
    }
});

List<Integer> received = Collections.synchronizedList(new ArrayList<>());

Subscriber<Integer> subscriber = new Subscriber<Integer>() {
    @Override
    public void onCompleted() {

    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(Integer it) {
        received.add(it);
    }
};

Subscription subscription = observable
    .take(2)
    .skip(1)
    .mergeWith(Observable.just(3, 4))
    .unsafeSubscribe(subscriber);

assertEquals(subscription, subscriber);
// subscription is still subscriber - it is just returned by unsafeSubscriber as-is!

assertArrayEquals(new Integer[]{2, 3, 4}, received.toArray());
// It works, surprisingly!
```

### But... *how* does operator chaining work?

We will now look inside `take()` function and see how one observable gets transformed into another after applying the operator.

``` java
Observable<Integer> observable = Observable.just(1, 2, 3);
OperatorTake<Integer> operatorTake = new OperatorTake<>(2);

List<Integer> received1 = Collections.synchronizedList(new ArrayList<>());

// Operator is a function that takes one subscriber and returns another:
// public interface Operator<R, T> extends Func1<Subscriber<R>, Subscriber<T>> {
observable
    .lift(operatorTake) // this is what happens inside of .take(2)
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer integer) {
            received1.add(integer);
        }
    });

assertArrayEquals(new Integer[]{1, 2}, received1.toArray());
// Works as expected.

List<Integer> received2 = Collections.synchronizedList(new ArrayList<>());

// We will now decompose the lift(Operator) function.
// It just returns a new Observable like this:
new Observable<Integer>(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> childSubscriber) {
        try {
            // This is where operator is called to create a new subscriber.
            Subscriber<? super Integer> operatorSubscriber =
                operatorTake.call(childSubscriber);
            try {
                operatorSubscriber.onStart();
                observable.subscribe(operatorSubscriber);
            }
            catch (Throwable e) {
                operatorSubscriber.onError(e);
            }
        }
        catch (Throwable e) {
            childSubscriber.onError(e);
        }
    }
}) {} // Observable`s constructor is protected, so we call
      // it here by creating an anonymous class
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer integer) {
            received2.add(integer);
        }
    });
// The chain of observables is constructed first.
// When subscribe() is called, a chain of subscribers gets created from operators.

assertArrayEquals(new Integer[]{1, 2}, received2.toArray());
// Works!
```

### Asynchronous subscription

Here we will see how `Observable.subscribeOn` works.

``` java
AtomicReference<String> onSubscribeThreadName = new AtomicReference<>();

Observable<Integer> observable = Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        onSubscribeThreadName.set(Thread.currentThread().getName());
        subscriber.onNext(1);
        subscriber.onCompleted();
    }
});

AtomicInteger received = new AtomicInteger();
AtomicReference<String> subscriberThreadName = new AtomicReference<>();

Subscription subscription = observable
    .subscribeOn(Schedulers.io())
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer integer) {
            subscriberThreadName.set(Thread.currentThread().getName());
            received.set(integer);
        }
    });

while (!subscription.isUnsubscribed()) {
    Thread.sleep(1);
}

assertTrue(onSubscribeThreadName.get().contains("RxCachedThreadScheduler"));
// OnSubscribe is called on some internal Rx thread, as expected.

assertEquals(onSubscribeThreadName.get(), subscriberThreadName.get());
// But values has been received on the same thread where subscription happened!
// This is because we control only subscription with subscribeOn.
// If we want to receive values on a specific thread we need to use observeOn.

assertEquals("main", Thread.currentThread().getName());
// We're still safe here.
```

### Observing values on a custom thread

This time we will use `Observable.observerOn` operator to receive values on a given thread.
We will also create our own simple `Scheduler` to understand how it works line.

``` java
AtomicReference<String> onSubscribeThreadName = new AtomicReference<>();

AtomicInteger received = new AtomicInteger();
AtomicReference<String> subscriberThreadName = new AtomicReference<>();

// We will add scheduled actions from the background thread here
// and run them on the current thread.
CopyOnWriteArrayList<Runnable> commandQueue = new CopyOnWriteArrayList<>();

Subscription subscription = Observable
    .just(1)
    .doOnSubscribe(() -> onSubscribeThreadName.set(Thread.currentThread().getName()))
    .subscribeOn(Schedulers.io())
    .observeOn(Schedulers.from(new Executor() { // Schedulers.from just wraps Executor.
        @Override
        public void execute(Runnable command) {
            commandQueue.add(command);
        }
    }))
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer integer) {
            subscriberThreadName.set(Thread.currentThread().getName());
            received.set(integer);
        }
    });

while (!subscription.isUnsubscribed() || commandQueue.size() > 0) {
    while (commandQueue.size() > 0) {
        commandQueue.remove(0).run();
    }
    Thread.sleep(1);
}

assertTrue(onSubscribeThreadName.get().contains("RxCachedThreadScheduler"));
// OnSubscribe is still called on the given io() thread.

assertEquals("main", subscriberThreadName.get());
// But we observe values on the given observeOn thread (which is "main").
```

### Backpressure

Backpressure is the ability of `Observable` to *not* emit more values than it is requested by `Subscriber`.

Let's see how it works.

```java
List<Integer> received = Collections.synchronizedList(new ArrayList<>());

Subscriber<Integer> subscriber = new Subscriber<Integer>() { // #1
    @Override
    public void onStart() {
        super.onStart();
        // Start from requesting one value.
        request(1); 

        // request() is Subscriber's method that just calls Producer`s request().
        // (more about Producer a few lines below)
    }

    @Override
    public void onCompleted() {}

    @Override
    public void onError(Throwable e) {
        throw new RuntimeException(e);
    }

    @Override
    public void onNext(Integer integer) {
        received.add(integer);
        if (integer < 5) // We want to get only 5 items.
            request(1); // #3
    }
};

Observable
    .create(new Observable.OnSubscribe<Integer>() {
        @Override
        public void call(Subscriber<? super Integer> targetSubscriber) {

            AtomicInteger lastSent = new AtomicInteger();

            targetSubscriber.setProducer(new Producer() {
                @Override
                public void request(long n) {
                    targetSubscriber.onNext(lastSent.incrementAndGet()); // #2
                }
            });
        }
    })
    .unsafeSubscribe(subscriber);

assertArrayEquals(new Integer[]{1, 2, 3, 4, 5}, received.toArray());
```

A careful reader could notice that `Subscriber` and `OnSubscribe` are calling each other recursively.
What will happen if we will change `(integer < 5)` to `(integer < 3000)`?

A-ha!

<pre>
 java.lang.StackOverflowError
 at ...MakeTheMagicGoAway$7.onNext(MakeTheMagicGoAway.java:201) // #1
 at ...MakeTheMagicGoAway$8$1.request(MakeTheMagicGoAway.java:232) // #2
 at rx.Subscriber.request(Subscriber.java:157)
 at ...MakeTheMagicGoAway$7.onNext(MakeTheMagicGoAway.java:218) // #3
 at ...MakeTheMagicGoAway$7.onNext(MakeTheMagicGoAway.java:201) // #1
 etc.
</pre>

In the real life we're not going to use backpressure in a single thread chain, so don't be scared by this exception.

### Asynchronous backpressure

Sending requests and values, back and forth, through different threads. Amazing!

``` java
List<Integer> received = Collections.synchronizedList(new ArrayList<>());

Subscriber<Integer> subscriber = new Subscriber<Integer>() {
    @Override
    public void onStart() {
        super.onStart();
        request(1);
    }

    @Override
    public void onCompleted() {}

    @Override
    public void onError(Throwable e) {
        throw new RuntimeException(e);
    }

    @Override
    public void onNext(Integer integer) {
        received.add(integer);

        // We can use large backpressure numbers now because
        // each scheduled action's stack starts from the scheduler's thread root.
        if (integer < 5000)
            request(1);
        else
            unsubscribe(); // We need to unsubscribe to exit background thread loop later.
    }
};

Subscription subscription = Observable
    .create(new Observable.OnSubscribe<Integer>() {
        @Override
        public void call(Subscriber<? super Integer> targetSubscriber) {

            AtomicInteger lastSent = new AtomicInteger();

            targetSubscriber.setProducer(new Producer() {
                @Override
                public void request(long n) {
                    targetSubscriber.onNext(lastSent.incrementAndGet());
                }
            });
        }
    })
    .observeOn(Schedulers.io())
    .subscribe(subscriber);

while (!subscription.isUnsubscribed()) {
    Thread.sleep(1);
}

assertEquals(5000, received.size());
// We can now receive any amount of requested items.
```