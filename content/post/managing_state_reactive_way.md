+++
title = "Managing state reactive way"
draft = false
date = "2016-12-25"
tags = ["Java", "Android"]
+++

RxJava nice and easily covers our multithreading needs.
But it provides only a half of the solution.
While we can compose functions with RxJava, there is no safe way we can have
our state (field variables) handled without race conditions and low-level synchronization blocks.

This article describes a simple and practical way for solving multithreading issues
that appear even when we're using such advanced technology as RxJava.

<!--more-->

## State, what is it?

> "State - all the stored information, at a given instant in time,
> to which the program has access"
> -- Wikipedia

For our needs we can say that state is the sum of all variables in the application memory.
State can also be limited to a subsystem, an object, even a function can have state.

## Object-oriented representation of state

In a usual OO application its state is distributed over a graph of interconnected objects.
When an external event happens, or when a background thread finishes
its execution (at best), it simply mutates the graph.
The downside of such approach is that complexity of such modifications is horrifying.

You're calling `getUserProfile()` and you have no idea what will happen next
and which variables will be changed during the call.
It may read database and update profile,
it may update your network token,
it may download new list of unread messages,
it may change user's age and so on.

You didn't want all of that while composing "Hello, %username%" string isn't it?
Especially if you was doing it while filling the profile view and you got an update
callback while processing a previous callback?
All these events, especially while happening concurrently, can give grey heir to anyone!

OOP model does not have any answers to all these issues.
Programmer is expected to write callback spaghetti, `synchronized` blocks, constantly
checking what is going on the other side of the application to avoid having conflicts.
Passing data in OOP is like giving someone a live rat instead of a juggling ball.
You want to juggle a bit, but
your data can run away or even bite you, suddenly throwing `NullPointerException`.
Multithreading issues in OOP code can not be reliably solved at all.

## Functional state model

[RxJava](https://github.com/ReactiveX/RxJava)
is a library for Functional Reactive Programming.
"Functional" here stands for composing functions.
But functional programming is not limited only to composing functions.
What about the other side - state?
Can functional programming teach us more?
Sure.

In real functional languages like [Clojure](https://clojure.org/), state is immutable.

That's it - if you have `Rect` data structure, you can't just change it's `x` and `width`.
The only way you can have some state changed - is to make a copy of the
data structure with new values of fields.
Want to modify a list? Make a copy of it.

It is a very simple and straightforward practice.
When an event happens, new "state" graph is created by taking
the old one with some fields changed.
And the new graph is assigned to the *single* variable which holds the root of the graph.

This sounds weird and very memory and performance consuming.
But in practice the difference is not so big, and what we get in exchange for
a negligible performance hit is outstanding reliability.
And you still can use mutable data in performance-critical parts.
Remember the Knuth's cite? It can totally be applied here.

> "We should forget about small efficiencies, say about 97% of the time:
> premature optimization is the root of all evil.
> Yet we should not pass up our opportunities in that critical 3%."
>   -- Donald Knuth

Because data is immutable
you don't have to make deep copy of the entire data graph.
You're not copying `String` values while passing them around, isn't it?
Because `String` is immutable, it is absolutely safe to pass it anywhere.
So we only need to allocate the difference between the old and the new graphs.

Let's take an example.
Here is the representation of state in a currency selection dialog:

```java
class CurrencySelectionState {

    public final List<CurrencyInfo> currencies;
    public final Optional<CurrencyInfo> selected;
    public final Optional<String> networkError;

    CurrencySelectionState() {
        this(emptyList(), empty(), empty());
    }

    CurrencySelectionState(List<CurrencyInfo> currencies,
            Optional<CurrencyInfo> selected, Optional<String> networkError) {
        this.currencies = currencies;
        this.selected = selected;
        this.networkError = networkError;
    }

    CurrencySelectionState withCurrencies(List<CurrencyInfo> currencies) {
        return new CurrencySelectionState(currencies, selected, networkError);
    }

    CurrencySelectionState withSelected(Optional<CurrencyInfo> selected) {
        return new CurrencySelectionState(currencies, selected, networkError);
    }
}
```

When user selects another currency, we only create a new instance
of `CurrencySelectionState` with the old list and new `selected` field.
We don't need to copy the currency list itself because it was not changed.
The list is immutable so we can safely reference it from the new state instance.

It can look tedious to write lots of `with-` methods, but in practice it does
not take too much time. If you're free in selection of libraries for your
project, you can try automatic code generators for such purposes, for example
[AutoValue: With Extension](https://github.com/gabrielittner/auto-value-with).

We can also stack such "modifications" by calling them in a chain:

```java
CurrencySelectionState newViewState = oldViewState
    .withCurrencies(newList)
    .withSelected(selectedItem);
```

After getting a new `CurrencySelectionState` instance,
all we need is to call our `currencySelectionDialog.update(newViewState)`
to refresh view and the job is done.

This solution with immutable state and separation
of state and its representation also becomes very popular
in web development.

When we're creating the new immutable 
data structure which represents some state, 
there are just no places where unpredictable things can happen.
Your jugging ball does not suddenly turn into a rat or a bird,
it does not try to bite you or to fly away.

#### Single source of truth

The idea of having a single source of truth fits nicely into this approach.
You have some state and when it changes you get notified. Nice!
And it is better to have the truth immutable, just to avoid having some grey hairs.

## Adding some rocket science

The article says "reactive", was it just do draw attention to some weird stuff
from functional languages? Of course not!
All the previous discussion was just to prepare you,
dear reader, to hit the next episode with some ideas about what "reliable", "predictable"
words mean when we're talking about state.

Let's adopt this model to our reactive multithreading environment,
so we could have a reliable and predictable state management for our
lovely RxJava.

What we really want is to have these things:

- Avoid deadlocks, race conditions, and all the fears of multithreading programming
- Make state modification sequential because it usually fits our requirements
- Observe state changes in the same sequence as we actually do them
- Of course, our solution must be dead simple
- Reusable

Let's take the example we already have with the selection of currency.
While user is scrolling we want to get next page of currencies from server
in background and when we receive it, we want it to be added to our list of already displayed
currencies. And when we do this we want to have a callback to update our view.

In RxJava we have a similar feature -- `BehaviorSubject`.
Unfortunately, it does not guarantee sequential emission of items on one thread.
(What if you call it's `onNext` on different threads? Hint: a race condition.)
It does not have thread-safe way to get current value and to emit a new - everything
is in hands of the user.

Why race conditions and wrong emission order is an issue?

Say, we have 10 currencies,
and when user scrolls down, we want to show a progress bar instead of the last item
on the bottom, and when we get the next page of items, we want to add them to the list.

In case of a race condition we can get items first and then progress bar without new items next.
This is not the most frequent case, but it will happen sometimes, especially
if you're using some tricky caching mechanics.

There can be much worse cases. Say if we're creating a messaging app
where items can be paged up and down, cached in a database, modified, animated, updated -
we must be very precise and should always get the updated list of items in the right sequence.

## Usage example

Here is how we're going to use it:

```java
class CurrencySelection {

    // The state itself.
    // We're providing the initial value for it.
    // We also need to specify a scheduler where we want to observe
    // emitted values.
    RxState<CurrencySelectionState> state =
         new RxState<>(new CurrencySelectionState(), mainThread());

    void onCreate() {
        state.observable()
            .subscribe(this::updateView);
    }

    // Call it from any thread, all the multithreading
    // will be handled by RxState.
    void onPagingUp(List<Currency> newItems) {
        state.apply(old -> {
            // Do not modify the old state, create a new one!
            ArrayList<Currency> newList = new ArrayList<>();
            newList.addAll(newItems);
            newList.addAll(old.currencies);
            return old.withCurrencies(unmodifiableList(newList));
        });
    }
}
```

This pattern perfectly fits into any architecture.
You can keep `RxState` in a view if you're adhering to "I don't care" architecture,
you can put it into a presenter when doing MVP,
it can represent ViewModel when you're on MVVM,
and it can be put into any singleton to represent
a service state, etc.

Hint: all these `addAll`, `unmodifiableList` can be too boring to type, so it is OK to write
your own utility functions like `List concat(List, List)` or use a functional
library with all the primitive functions and immutable collections included:
[Solid](https://github.com/konmik/solid).

## Implementation

Here is some implementation reasoning:

- We're making modification sequence strict by
   taking the old state to produce the new one
- We do state modification in a lock to prevent multiple modifications
   of state happening concurrently
- Each new state is being put into a queue
- Our emissions should always happen in one thread,
   this is the only way we can guarantee their sequence

I'm not sure yet if I need to release it as a library.
For now you can just copy/paste the code (see below).

Here is the implementation for RxJava 1:
[RxState](https://github.com/konmik/rxstate/tree/master/rxstate/src/main/java/util/rx).

Here is the implementation for RxJava 2:
[RxState](https://github.com/konmik/rxstate/tree/master/rxstate2/src/main/java/util/rx2).
