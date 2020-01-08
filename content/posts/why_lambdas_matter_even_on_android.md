+++
title = "Why lambdas matter (even on Android)"
draft = false
date = "2016-01-06"
tags = ["Java", "Android", "Programming theory"]
+++

A little knowledge exchange from an active lambdas user.

_Term: **Lambda** is an anonymous function._

<!--more-->

### Introduction

I started writing computer programs about twenty
years ago, when C++ has just been released.
Applications written entirely in Assembler were not rare those days.
We had Pascal, C, Assembler, FoxPro.
Garbage collection? If you wanted to create an
object - please, allocate and release it's memory manually!
We had an entire class of languages without stack!
And they were quite popular in the business area.

OOP was for weird guys.
Believe me, if you would start studying programming 20 years ago,
the OOP architecture looked *very* alien.
You knew that there are *no* objects inside computers!
There are only data structures and code.
Data do not have behavior.
It just lays there, waiting for what you're going to do with it.

For us, who started writing programs without OOP,
the only advantage OOP has is that it allows to pass a reference to a function
(via an overridden method)
and execute the function with arguments (object fields) *later*.
Sorry, there are no other real (not imaginary) benefits of OOP.

Finally, we on Android got the possibility to pass functions without
architecting and constructing classes first!
Amazing! Lambda is a fresh air of freedom.
Now we don't need to invent overcomplicated
relationships between variables and to apply entangled patterns
just to compose actions!

*Pure power*

The more flexible our code is, the more
efficient it is. Having a function that can be applied in 100 cases
is more efficient (more powerful) than having 10 functions that
should be applied in those 100 cases instead.

_Having to write 10 lines of code instead of 100 lines **is** power of
a programming language._

Yes, it is true that we *can* do everything without lambdas.
It is also true that we can do everything without Java, using C++ or even pure Assembler.
The main difference we have is the time spent on each feature and the amount of
time spent while fixing bugs.

Here are my simple practical examples of creation of more flexible code using lambdas.
We'll walk through some examples and see how they can change usual development practices we have.

I hope these examples will be useful to make more code better
(powerful, flexible and reliable).

### Syntax extension

Let's start from something simple.

Sometimes Java is too verbose.
We can tweak some stuff to make it more digestible.

##### Rethrow

`rethrow` is the function I use most often. It allows to rethrow an exception from
any other function that has checked exceptions declared.
Sure, it is not a good idea to blindly rethrow *all* exceptions, but if
you want your app to crash (fail-fast strategy), why bother?

Was:

``` java
try {
    Thread.sleep(100);
} catch (InterruptedException e) {
    throw new RuntimeException(e);
}
```

Became:

``` java
rethrow(() -> Thread.sleep(100));
```

And even:

``` java
int byte = rethrow(() -> inputStream.read());
```

Implementation:

``` java
public interface RunnableThrows {
    void run() throws Exception;
}

public static <T> T rethrow(Callable<T> callable) {
    try {
        return callable.call();
    } catch (Exception e) {
        throw rethrow(e);
    }
}

public static void rethrow(RunnableThrows runnable) {
    try {
        runnable.run();
    } catch (Exception e) {
        throw rethrow(e);
    }
}

/**
 * http://www.mail-archive.com/javaposse@googlegroups.com/msg05984.html
 */
public static RuntimeException rethrow(Throwable throwable) {
    sneakyThrow0(throwable);
    throw null;
}

@SuppressWarnings("unchecked")
private static <T extends Throwable> void sneakyThrow0(Throwable throwable) throws T {
    throw (T) throwable;
}
```

This `sneakyThrow0` just rethrows a checked exception without wrapping it into
`RuntimeException`, so the exception's stacktrace becomes shorter.

##### Suppress

`suppress` is a brother of `rethrow`. It suppresses a thrown exception.

Instead of:

``` java
BigDecimal number = null
try {
    number = new BigDecimal(string);
}
catch (NumberFormatException ignored) {
}
```

Write:

``` java
BugDecimal number = suppress(() -> BigDecimal(string));
```

You can also be specific:

``` java
BugDecimal number = suppress(() -> BigDecimal(string), NumberFormatException.class);
```

Implementation:

``` java
@Nullable
public static <T> T suppress(Callable<T> callable, Class<? extends Throwable>... throwables) {
    try {
        return callable.call();
    } catch (Throwable thrown) {
        if (throwables.length == 0)
            return null;

        for (Class<?> throwable : throwables) {
            if (throwable.isAssignableFrom(thrown.getClass())) {
                return null;
            }
        }
        throw rethrow(thrown);
    }
}
```

You can modify `suppress` to return boolean, `Optional` or to have any other tricky behavior.

##### Delayed

`Delayed` is a class that gives access to a variable that should be initialized on demand
and cached for later usage.

Was:

``` java
private int value;
private boolean isValueInitialized;

private int getValue() {
    if (!isValueInitialized) {
        value = calculateValue();
        isValueInitialized = true;
    }
    return value;
}
```

Became:

``` java
private Delayed<Integer> value = new Delayed(() -> calculateValue());
```

This kind of variables are quite helpful when it comes to declaring data that does not exist at the moment
of the class instantiation. On Android we can refer to Activity's extras like this:

``` java
private Delayed<Record> record = new Delayed<>(() -> getIntent().getParcelableExtra("record"));
```

Implementation:

``` java
public class Delayed<T> {

    public interface Factory<T> {
        T create();
    }

    private final Factory<T> factory;
    private T value;
    private boolean initialized;

    public Delayed(Factory<T> factory) {
        this.factory = factory;
    }

    public T get() {
        if (!initialized) {
            value = factory.create();
            initialized = true;
        }
        return value;
    }
}
```

This pattern allows to solve one of fundamental problems: we don't want to calculate a value
twice but we don't want to manually control the variable initialization/usage order.

### Algorithm (Strategy, Policy) pattern

_Term: **Algorithm** is a procedure or formula for solving a problem._

Technically, any sequence of actions that has more than one step is an algorithm.

One of the most widely used OOP patterns disappears in languages that have lambdas.

There is a known example of the pattern: `Collections.sort(list, comparator)`
function. Instead of implementing your own class which has `Comparator` implemented,
you can just pass a lambda into the `sort` function.

Here is another example. How many times did you write something like this?

``` java
public static void copy(InputStream in, OutputStream out) throws IOException {
    byte[] buffer = new byte[4096];
    int read;
    while ((read = in.read(buffer)) != -1) {
        out.write(buffer, 0, read);
    }
}
```

But what if you want to copy not from `InputStream` into `OutputStream` but from any other
pool of data into another pool?
OOP way is to implement `InputStream` and `OutputStream`
every time you need a buffered copying. This is very hard so nobody actually do this and we have
tons of buffered copying functions everywhere.

Here is an algorithm that does not depend upon a specific class.

``` java
public interface Action1Throws<T1, E extends Throwable> {
    void call(T1 t) throws E;
}

public interface Func1Throws<T1, R, E extends Throwable> {
    R call(T1 t1) throws E;
}

public static <E1 extends Throwable, E2 extends Throwable>
void bufferedCopy(Func1Throws<byte[], Integer, E1> reader, Action1Throws<byte[], E2> consumer) throws E1, E2 {
    byte[] buffer = new byte[4096];
    int read;
    while ((read = reader.call(buffer)) != -1) {
        consumer.call(read == buffer.length ? buffer : Arrays.copyOf(buffer, read));
    }
}
```

Yep, the function declaration looks terrible. :D This is Java, a quite verbose beast.

But now we can reuse the code!

``` java
public static String md5(InputStream in) throws IOException {
    MessageDigest digest = rethrow(() -> MessageDigest.getInstance("MD5"));
    bufferedCopy(in::read, buffer -> digest.update(buffer, 0, buffer.length));
    return new BigInteger(1, digest.digest())
        .toString(16).substring(0, 32).replace(' ', '0');
}
```

The OOP version of Algorithm pattern was very poor. Now we can *really* reuse our algorithms.

Consequences of this pattern replacement are fundamental.
In computer science we have two basic things - code and data.
We can modify data in many ways but our code is not so easily modifiable (at least in Java).
This is weird because latest "best practices" say that we must rely on
*immutable* data and the problem we're facing every day is that
our code always needs better flexibility!

_Term: **Dependency** is an object that is referenced by another object._

In OOP languages we achieve code modification by leveraging inheritance
and building dependency graphs that behave
differently when we replace some of graph nodes.
This is an *extremely* complicated solution.

Class graphs are hard to work with. You modify an object in one place
something other breaks in another place.
This happens *always* even with greatest professionals,
this is our OOP curse.

Good class graphs are hard to create. They are extremely hard to test.
(Hey, I know that someone will say that everything is easy, but!
Please, recall how you tried to understand a dependency framework for the first time.
Some of us have huge ego and short memory, so there are probably some people who
will still say that it was easy and try to play wunderkinds around this.
Just don't buy it. (Some of us *are* wunderkinds, but this is a different story.))

**Lambdas allow easy, safe code modifications without the need in OOP.
We don't need to imagine that data have behavior for the sake of code modification anymore!**

### Streams

The fact that now we can reuse our algorithms raised an obvious question:
"What are the most usual algorithms that can be reused?"

Traditionally, functional languages like Haskell and LISP have libraries
of algorithms for data processing.
Java-based languages (Scala, Kotlin) also have them, and
Java 8 got it's own library: "Java SE 8 Streams".

We don't get Java SE 8 Streams library on Android with Retrolambda, but if you're interested,
there is a couple of alternatives:
[Solid](https://github.com/konmik/solid),
[Lightweight-Stream-API](https://github.com/aNNiMON/Lightweight-Stream-API).

How streams look like? (Solid)

Was:

``` java
public static List<File> tree(File path) {
    List<File> result = new ArrayList<>();
    result.add(path);

    File[] files = path.listFiles();
    if (file == null) {
        return result;
    }

    for (File file : files) {
        files.addAll(tree(file));
    }
    return files;
}
```

Became:

``` java
public static Stream<File> tree(File path) {
    File[] files = path.listFiles();
    return files == null ? of(path) :
        of(path).merge(stream(files).flatMap(it -> tree(it)));
}
```

The syntax can look weird and you need to study several new methods,
but it is just a matter of habit. It took just one month for me
to completely switch my way of thinking about data.

Another awesome library with reusable algorithms
is [RxJava](https://github.com/ReactiveX/RxJava).
It allows to reuse multithreading and event-processing operations.
The library is hard to get at
the beginning, but once you start, you can't stop using it everywhere.
I can easily imagine an application
where all events are processed with RxJava instead of usual OOP method calls.

### Transforming a listener class into lambda

Here is a trick on the final note.

Sometimes you want to set a listener, but if the listener is an interface
with several methods, such interfaces can not be substituted by lambdas.

Was:

``` java
animator.addListener(new Animator.AnimatorListener() {
    @Override
    public void onAnimationStart(Animator animation) {

    }

    @Override
    public void onAnimationEnd(Animator animation) {
        view.setVisibility(GONE)
    }
});
```

Became:

``` java
animator.addListener(new AnimatorEndListener(() -> view.setVisibility(GONE));
```

With:

``` java
public class AnimatorEndListener implements Animator.AnimatorListener {

    private final Runnable onEnd;

    public AnimatorEndListener(Runnable onEnd) {
        this.onEnd = onEnd;
    }

    @Override
    public void onAnimationStart(Animator animation) {

    }

    @Override
    public void onAnimationEnd(Animator animation) {
        onEnd.run();
    }
}
```

### The future of programming languages

In the introduction I wrote about power of languages. The idea is not new. Any programmer who
knows a couple of languages with different possibilities can easily say which one has more power.

I highly recommend reading
[Beating the Averages](http://www.paulgraham.com/avg.html)
by Paul Graham on this subject.
The article was written in 2001, so which language is the most powerful *now*?
Guys, you will hate me - nothing changed. The most powerful language these days is...
[Clojure](http://clojure.org/), a LISP dialect.

What does it have Java-based languages don't?

* An easy compile-time code generation and extensible syntax (unbelievable flexibility of algorithms);
* Enforced immutability of data (awesome for architecting and multithreading);
* A very efficient type system;
* A uniform scientifically justified syntax which has composition as its primary intention.

No, this is not possible to accomplish in Java-based languages
under any circumstances not in the near feature, not 100 years later, so LISP will always be the
most powerful language unless something specific will happen, which is unlikely.

We can't use Clojure on Android right now because of it's slow startup time on mobile devices.
There are different possibilities for running ClojureScript but they are not tested thoroughly yet.
I hope that we will get Clojure sooner or later.

### References

[How Scala killed the Strategy Pattern - Alvin Alexander](http://alvinalexander.com/scala/how-scala-killed-oop-strategy-design-pattern)

[Beating the Averages - Paul Graham](http://www.paulgraham.com/avg.html)

[Succinctness is Power - Paul Graham](http://www.paulgraham.com/power.html)

[SOLID Principles of Object Oriented and Agile Design - Robert C. Martin](https://www.youtube.com/watch?v=TMuno5RZNeE)

[The repeated deaths of OOP - Loup Vaillant](http://loup-vaillant.fr/articles/deaths-of-oop)

[Simple Made Easy - Rich Hickey](http://www.infoq.com/presentations/Simple-Made-Easy)

[gradle-retrolambda](https://github.com/evant/gradle-retrolambda)

[RxJava](https://github.com/ReactiveX/RxJava)

[Solid](https://github.com/konmik/solid)