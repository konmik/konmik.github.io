+++
title = "How to live better without Dependency Injection"
draft = false
date = "2020-01-08"
tags = ["Java", "Android"]
+++

*(I've just learned Dagger, what is wrong?
Imagine my anxiety now, I am NOT happy seeing an article with this title!
I hope that the author does not have any idea!)*

Five years have passed since my first [Dagger 2 article](http://konmik.com/posts/snorkeling_with_dagger_2/).
It is time to write the last one.
We do not need Dagger anymore.
In fact, we didn't even need it in the first place.

<!--more-->

### What is wrong

Dagger allows us to solve many issues that we have with OOP code.
It helps with testing,
makes code declarative,
makes it easy to use some best programming practices,
verifies the dependency graph at compile-time,
and allows us to look down on the developers who are not able to understand it.

But it comes with serious costs.

First, **Dagger is a code generator**.

It increases build time and, like any other code generator, it causes build configuration issues.
Some developers have to clean their projects before building even though they have changed just one line.
Incremental compilation breaks often.
I heard of a project where Dagger takes 50% of the build time.
Maybe it is OK for some, but I waste more than 10 hours monthly waiting
for my builds, adding another 10 hours would not help.

Second, **Dagger is complex to setup**.

Build configs, configuration classes, configuration interfaces,
annotations, scopes, modules, components, whatever.
This is ridiculously complex for a developer who just wants to get a dependency.
If we want to use Dagger for tests it comes with an even bigger tax.

Third, and the most important, **It locks the codebase to OOP paradigm**.

With the rise of functional programming, passing dependencies using contructors becomes harder.
There are simply no constructors! There are even no mutable fields to inject into!

When instead of creating an OOP hierarchy there is a possibility to solve a problem in one function
call with a lambda, convincing yourself in Dagger benefits becomes harder and harder.

### Are there alternatives?

I am not telling anyone to go and buy something like [Koin](https://insert-koin.io/).
If we could be satisfied with a dependency injection framework that has so little
features I would recommend keeping all of the dependencies
inside of `Map<Class, Lazy>`, one for each scope.
At least that would have the benefit of being
[Stupid Simple](http://people.apache.org/~fhanik/kiss.html).
(Looking at Koin source code... hey, they are actually using maps...)

I am telling that we do not need dependency injection at all!

What are the problems we want to deal with using DI?

- **modularization** - We want to put our code into different modules and have them linked after the app starts.
- **testing** - We want to supply mocks and test objects instead of dependencies.
- **boilerplate** - We want to get rid of the pain of passing dependencies everywhere manually.

So if we find a way to get rid of these problems while adding fewer problems than Dagger adds, we win!
Can we have a solution that would deal with these issues for us?

### Hold my beer (examples are in Kotlin, because 2020)

Let's start with a simple example.
The code will look terrible at the beginning but it will become better at the end.

We're going to travel into the functional world. That's why we will start by injecting *functions*.

**Module A**:

```Kotlin
lateinit var output: (Any) -> Unit

fun outMax(a: Int, b: Int) {
    output(max(a, b))
}
```

**Module Main** (depends on **Module A**):

```Kotlin
fun main() {
    output = ::print

    outMax(2, 4)
    outMax(2, 1)
}
```

Output:

```
42
```

Modularization problem has been solved!

*wat\
That was too fast.\
What is happening here?*

Please, pay some attention - I am explaining the magic.

In **Module A** we're defining `outMax` that will output (print to standard output) the max of two numbers.

It depends on `print` function, but instead of calling `print` directly (assume it exists only in **Module Main**),
we're using variable `output` that will hold a reference to `print` function.

When `main` function starts the first thing it does is initializes `output` with
the real function implementation (we're using standard Kotlin `print`).
Then it is OK to call `outMax`.

If we try calling `outMax` without initializing `output` first
there will be `UninitializedPropertyAccessException` thrown,
it will signify that our dependencies are broken
(like we forgot to call Dagger's `inject` and get `NullPointerException`).

Literally, we've just replaced a class hierarchy (as we usually do with Dagger)
with a function reference.

Zero annotations so far, let's move further.

### Hold my beer 2 - testing

**Module A** test

```Kotlin
class ModuleAKtTest {
    @Test
    fun printMaxPrints() {
        var printed: Any? = null
        output = { printed = it }
        outMax(2, 42)
        assertEquals(42, printed)
    }
}
```

Output:

```
BUILD SUCCESSFUL in 0s
```

This was a little joke about gradle.

I will just type here the string that I see in my IDE because configuring gradle test output is too hard.

```
Tests passed: 1 of 1 test - 14ms
```

*Wait a moment!\
Are my eyes lying to me?\
It took more lines to write the stupid joke than injecting a test `output`?\
How is it even possible?\
Where is the test Dagger module definition?\
Where is a Dagger component?\
Why did it take just 14 ms?\
WHO STOLE MY ANNOTATIONS? :-/*

I will say more: in a real test we will be using Mockito, so we will not even need `var printed: Any? = null` line.
One dependency injection = one line of code.
And if we keep our code clean and tidy this rule will work in ALL tests.

This is it!
This is the power of KISS!

### Solving the boilerplate issue

Solved!

Right, there is no boilerplate code to deal with.

Feeling a bit upset? Or happy? In a doubt? The example code sucks a bit?

### Making it ready for production

Anyone can come and redefine `output` at runtime.
Let's add some protection to prevent inexperienced developers from breaking it.
A single variable has no logic, so it cannot control its assignment.

The obvious design is to create an intermediate object that has `fun inject()` function
that will check if it is the right time to have an injection.

```Kotlin
class Late<T : Any> internal constructor() {

    private var value: T? = null

    fun get(): T =
        value ?: throw UninitializedPropertyAccessException()

    fun inject(it: T?) {
        if (locked) throw IllegalStateException("Already injected")
        value = it
    }
}

fun <T : Any> late(): Late<T> =
    Late()

fun lockLateInjections() {
    locked = true
}

private var locked = false
```

In **Module A**:

```Kotlin
val out = late<(Any) -> Unit>()

fun printMax(a: Int, b: Int) {
    out.get().invoke(max(a, b))
}
```

In **Module Main**:

```Kotlin
fun main() {
    out.inject(::print)
    lockInjections()

    printMax(2, 4)
    printMax(2, 1)
}
```

If somebody will try to inject a new `out` function after `lockInjections()` he or she will get `IllegalStateException`.
If somebody will try to call `out.get()` before it was injected `UninitializedPropertyAccessException` will be thrown.

All checks are in place!

*Is it a good design to crash the app in case if a developer makes a mistake?*

Java does it all the time, just check standard library.
Having an exception *in case of a developer's mistake* is a good design.
That's why they were created for.

The only thing that left is the ugly call syntax `out.get().invoke(42)`, let's add some Kotlin sugar.

```Kotlin
operator fun <T : Any> Late<T>.getValue(thisRef: Any?, property: KProperty<*>): T =
    get()

operator fun <T : Any> Late<T>.setValue(thisRef: Any?, property: KProperty<*>, value: T?) =
    inject(value)
```

Now it can be used even nicer:

```Kotlin
var out by late<(Any) -> Unit>()
out = ::print
out(42)
```

Done!

# Pros and cons of `Late`

Pros:

- a dependency definition is one line of code
- a dependency injection is one line of code
- a test injection is one line of code
- no code generation is needed
- no OOP hierarchies are needed
- no frameworks are needed
- no libraries are needed
- Stupid Simple

Cons:

- no OOP-stile dependency management -
we cannot have different instances of the same function injected into different places
- no compile-time checks of the dependency graph -
a dependency can be not initialized at the right moment in time, causing `UninitializedPropertyAccessException`
- no scopes -
if we have a dependency that is limited to one screen the dependency will not be disposed when the screen closes

##### Why OOP-style dependency management is not that important

OOP-style dependency management can only be needed when we're dealing with OOP code.

If we're trying to implement our apps in a functional manner
we inject functions, and they rarely have mutable variables inside.
If they have no mutable variables inside it means that there is no
point in having different instances of the same function in different places.

When the amount of OOP code with mutable variables all
over the application decreases we need less OOP-style dependencies
so we need DI less and less (it can be replaced by manual parameter passing using constructors).

There is another trick.
Open your Dagger module, or where you define the dependencies in your DI framework.
Look carefully.
About 90% of the dependencies are singletons.
Just make them injectable with `Late`!

##### Why compile-time checks of the dependency graph are not that important

Not all DI frameworks have it.
Most of the modern Kotlin DI frameworks do not even have an annotation processor!

Practice shows that uninitialized dependencies are almost always getting caught during development time.
I had it only once when dependency access was happening before it's initialization in production and Dagger
could not help in that case too because it would not be initialized yet.

To avoid having uninitialized dependencies I recommend writing a single function `injectAll()`
and then add `injectToModuleA()`, `injectToModuleB()` and other injection functions,
and call them during app startup in one place.
After injection ends services and other application parts can be started.

##### Why scopes are not that important

Using this simple approach we can deal with about 90% of the dependencies without involving OOP and DI -
they can be singletons or function references.

The scope-specific OOP part can be "injected" manually by calling constructors with parameters as we did in pre-DI era.

# Other info

I had the same approach used in a JavaScript pet project.
JavaScript does not have `by` operator overloading, but I was able
to make the injection code look nice by having `late()` function
returning a function that has methods,
so I could call `out.inject(x)` and then call `out()`.
JavaScript may look a bit weird now, but think a bit - the same trick is possible in Kotlin! ;)

# A fitting cite that contains a summary of this article

> "All problems in computer science can be solved by another level of indirection" -- David Wheeler

One of the main keys to the KISS principle is to use as little indirections as possible.
Instead of having them all over constructors and DI frameworks it is enough to define them once.

Have fun!
