+++
title = "When to NOT use RxJava"
draft = false
date = "2017-01-10"
tags = ["Java", "Android"]
+++

RxJava is very powerful.
It solves a great amount of tasks easily.
But using it in wrong places can turn any codebase into an unmaintainable and buggy mess.
Here I will try to explain where RxJava can help and in which cases it is better to avoid using it.

<!--more-->

## Basic example

Let's take a very simple and obvious example of a wrong RxJava
usage and go into theoretical details later.

Often, people inspired with RxJava
start turning everything into a stream.
We even got a new mantra - "Everything is a stream".

Say, you need to implement `int max(int, int)` function (let's pretend it does not exist yet).
What will you think if you discover an implementation like this
that was proudly created for you by a newbie who can't wait to apply the new magical hammer?

#### Figure A.

```java
Observable<Integer> max(Observable<Integer> left, Observable<Integer> right) {
    return Observable.zip(left, right, (a, b) -> a < b ? b : a);
}
```

Interesting, yeah?

> "When you have a hammer everything looks like a nail."

A sane developer will probably say "WTF??!" and implement something
less shiny and inspiring, but more practical:

#### Figure B.

```java
int max(int a, int b) {
    return a < b ? b : a;
}
```

There is even an option to refactor the old code like this:

#### Figure C.

```java
Observable<Integer> max(Observable<Integer> left, Observable<Integer> right) {
    return Observable.zip(left, right, MyMath::max);
}
```

But in practice such a weird `max` implementation is just a consequence of other
weird functions that for some reason will look better if `max` works with observables.

This example can be easily scaled into any
size -- people turn into stream *everything* literally.

You can easily discover code like this:

```java
Observable<Integer> getId() {
    return Observable.just(this.id);
}
```

In a large codebase it happens all the time.

Or you can find a 20-line function of operators
that is doing a job that can easily be done without RxJava -
with the same amount of code!
Try debugging a flow of events when the person who wrote it
did not even think about events.

I'm not saying that mantras do not work.
I hope that at this point that's pretty obvious. ;)

## So what exactly is wrong here?

This entire rant is about simplicity.

**Figure B.** says:
"take two ints and return one of them that is bigger" 

**Figure A.** says:
"Take two streams of integers, they will emit values once in a while, and there
will probably be not equal amount of values in these streams, and these streams
will probably will never emit any values, but just take these two streams
and create another one that will wait unless both streams will emit at least one value,
and return a value every time this happens, and the value should be
the maximum of these two values, but if one of streams will return less values then
you should wait unless it will emit something and you should buffer
items of another stream while you're waiting.
Keep in mind that I 
can subscribe any number of times and you need to create the new sequence every time.
There is also a possibility to unsubscribe at any moment, so you will also need
to unsubscribe from these two streams.
Both streams can emit new values on different threads
so you will need to emit the next value on the thread of the latest emission.
Propagate all exceptions and stop stream once it happens.
Stop returned stream if one of incoming streams stops." 

The enthusiastic developer will probably say:
"But we will never have such conditions!
Both streams will always have exact amount of values,
and I'm not going to use multithreading here,
and... and..."

C'mon, tell this to your mother.

Bugs happen all the time and one day one of these observables
will NOT return any value and your app will just hang.
And it will happen with 10% of all of your users.
And you will not even get a stacktrace because nothing crashes.
Or you will get a stacktrace, but it will not change anything,
because RxJava stacktraces are something special.

## DO NOT

#### The absence of events

**Do not** use RxJava if there is no "event". Just don't.
If something does not say: "this will happen at some point in time",
do not create observables.

RxJava is not a *data processing* library.
It is an *event processing* library.

Events are more complex than data
because, in addition to data, events have *time* or *order of execution*.

If you need to process a list of items, do not turn it into an observable.

Besides the complexity that will increase
for a person trying to understand
the code, there are other downsides.
Have you seen RxJava source code?
It contains quite complex logic, so it is hard to debug.
It also consumes additional resources to make multithreading safe.

If you want to use functional programming while you do not have Java 8 streams,
just take a third-party library or write a couple of utility functions.
Your code will be much simpler and there will be less possibilities for bugs.

#### Same life duration

Say, you have `Screen` and `Button` objects.
Both operate on one thread, both have the same life duration.

There are two general ways to pass events between them in OOP.
You can either set a callback or pass one object into another's
constructor arguments.

Passing an object into constructor is usually simpler because
this way you're strictly saying that the passed object's life duration
is at least as long as the object's that is being created.

```
class Screen {
    
    Button logoutButton;
    
    Screen(Button logoutButton) {
        this.logoutButton = logoutButton;    
    }
    
    onProfileReceived(Profile profile) {
        logoutButton.setVisibility(profile.isLoggedIn ? VISIBLE : GONE);
    }
}
```

A callback without the possibility to uninstall it
is another technique that can be used if you need to
propagate events into another direction.

```
class Screen {

    Screen(Button logoutButton) {
        logoutButton.setCallback(() -> doLogout());    
    }

    doLogout() {
        network.logout();
        profile.update();
    }
}
```

However, if `Button` already exposes an observable for click notifications,
it is OK to use it as it will not require any
additional lines of code to implement.

```
logoutButton.clicks()
    .subscribe(() -> doLogout());
```

#### Reactive spaghetti

Don't do like this (simplified example):

```
.switchMap(it -> serverApi.login(name, password)
    .switchMap(loginResult -> serverApi.requestChatToken(loginResult.loginToken)
        .switchMap(chatToken -> serverApi.doOtherThingsAfter(chatToken))))
```

Do like this:

```
LoginResult loginResult = serverApi.login(name, password);
String chatToken = serverApi.requestChatToken(loginResult.loginToken);
return serverApi.doOtherThingsAfter(chatToken);
```

Why the second way is better?
Because we're not trading ONE event into FOUR.
When we're developing software we want to simplify things, not to complicate them.
Events are more complex than linear code execution.

It is easier to modify linear code as well.
Imagine you decided to wrap `login` and `requestChatToken` into `synchronized` block
to stop some other network interaction in-between.
It is trivial to do when you have linear code
but it is crazy hard to do when every your network call can be triggered by something
you can't control.

Write spaghetti only when there are no other ways.

#### Returning lambdas, implementing functional interfaces by objects

If you're doing something the title says,
there is a very high chance that there are simpler ways
implementing the required functionality.

We *can* replace *all* methods in OOP using RxJava.
It is doable but it complicates understanding.

Java is not a functional language -- it is an OOP language
with some functional capabilities.
Java functional capabilities are verbose,
they are also prone to accidentally making bugs.

Clean and understandable code design matters.

## DO

If it seems to you at this point that using RxJava is an entirely bad idea,
there *are* very good use cases where RxJava really shines.

#### Multithreading

Not much libraries compare to RxJava when it comes
to dealing with multithreading code.
Having a simple way to schedule function calls on different threads
is priceless.

#### Subscriptions

When your objects have different life duration, having
a simple way to manage their communication is very important.

Forget about writing callback interfaces, lists of subscribers,
event dispatching loops, and so on.

## Good practices

#### Side effects

*Side effect* -- is when function modifies variables outside of its scope
or makes I/O. Side effects should normally be executed in a specific order,
for example `print` that prints information to console creates a side-effect
and if you will call several `print` in wrong order you will get wrong output.
If you modify variables in wrong order you will also likely have bugs.

RxJava has very clear semantic for using it with functions which
create side-effects. This is useful when you're trying to
understand someone's code, and it can also save you from having
more bugs.

Use `subscribe` function callbacks -- they are intentionally designed for doing this.
You can also use `do*` operators (`doOnNext`, `doOnError`, etc) in case you
need to create a side-effect during chain execution.

The default way to go is you need to pass data INTO the chain from a side:
`flatMap`, `switchMap`, `concatMap`.
There are more functions which fit this purpose,
look at `Observable` definition.

It is wrong to use other functions (`Observable.map` for example)
for creating side-effects
because it assumes that you're just transforming one value into another.
A developer who will try to modify your code
will think that it is OK to place this transformation into
a different place, put it into another thread
or even completely remove it while refactoring.
If your `map` was saving the values into a database
or was showing them on the screen
then the refactoring will cause bugs.

#### Simple first

Once you need to execute some action in background, it is
better to implement it as a simple function first,
and only then wrap it into an observable in case you need it.

In some cases you will discover that composition of
non-rx code is *easier* than having an observable that
does hell knows what else.

Simple code can be turned into complex very easy.
It is hard to do the opposite.

#### Single thread first

This is a sub-rule for "Simple first".

Once your function returns an observable,
it is better to not schedule the returned observable
on a specific scheduler inside the function.

Unpredictable things can happen if the caller does not expect
to get multithreading from the returned observable.

Excessive jumps from one thread to another can also
be slower than you can afford.

Let the caller decide where it wants to schedule actions.

## Have safe rx-ing! :)

