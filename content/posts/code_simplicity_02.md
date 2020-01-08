+++
title = "Immutable data is the main key to simple code"
date = "2018-02-06"
tags = ["Programming theory"]
+++

Use immutable data like a pro.

<!--more-->

### What this article is about?

I was often told that to understand how to work with immutable data
I have to study a real functional language like Haskell,
where mutable data does not exist at all.

Luckily, you don't have to - I'm already typing this article, it should be enough.
I'm not that kind of a snob to give someone hard to reach goals just to
make a secret, draw attention and to show how smart I am.
I'm drawing your attention by *revealing* the secret. ;)

If you haven't yet - I'm inviting you to read my first article on
the subject of code simplicity - [The Code Simplicity Chart](/post/code_simplicity_01).
The current article is an in-depth look into the problem of code simplicity
and it is based on the forementioned article completely.

# Reducing the amount of events

We usually cannot reduce the number of events that are coming into our program
from external sources.
Things like button clicks, network requests and responses, timers,
operating system events -- we can barely control.

However, our modern programming practices force us to create internal program events
in addition to those we get from external sources.
Every book on object-oriented programming teach us how to create
a self-controlling piece of program that reacts on events and initiates other events
in response.
At the end our programs consist of thousands of interconnected pieces
that send events to each other in a totally uncontrolled manner.

By forcing our data to be immutable we're reducing the amount
of events that are happening inside of our programs.
How is this possible?

### An example

Imagine we have a `Person` object that is being shown on the screen.
Every time we mutate its fields we need to raise a corresponding event to update the
corresponding text shown on the screen.
We're changing the person name - we have to update the person's name on the screen.

What if we want this data to be persisted as well?
The person name has changed - how we need to save the new name to the database.
What if `Person` has tens of fields? 

This strategy is complex.

In reality we're trying to bundle several events like this into one - the person
was changed, so we're serializing it into the database entirely to avoid writing
a separate flow of events for each field.

But what if we want to show a *list* of persons on the screen?

That's where our event bundling strategy usually fails
because this is so easy -- to mutate a data field, keeping the
rest of the list unchanged.
We're installing a callback for each item in the list so when it changes -
it updates the corresponding text on the screen.
Sounds easy?

Not that easy because we can get a bunch of other problems:

- We need to keep track of newly created and shown persons
  to subscribe to them.
- We need to keep track of items we have already subscribed to
  to prevent double subscription;
- It becomes harder to implement pagination (now we have to subscribe
  to new items in the list that appear while user scrolls for more items);
- We have to track that new instances of persons got shared properly and we do not have
  two instances of the same person hanging in memory;
- If our program can show the same items in another part of the
  program and in addition to showing persons it wants to save
  them into database on each change this can cause unbelievable amount
  of complexities and, uh, bugs.

But having immutable list of immutable persons solves all of these
problems completely.
Now, instead of having to track each person we're tracking
the list of persons at once, the number of possible events gets reduced dramatically.

Imagine now we have only *one* mutable variable, and the variable name is `items`.
It contains an immutable list of persons.
When we need to update the list on the screen we're
just raising *one* event that passes the immutable list
to renderer that will show it to user.
Do we need to show more persons?
Just create a new list with persons added to the end,
assign this list to `items` variable and pass this new instance
of immutable list to the renderer.
What can be simpler?

### To summarize

Instead of creating a mutable data structure for modifying it directly
and calling different types of events to update other pieces of data,
it is simpler to have the data structure immutable
and only call *one* event "data has changed" passing the new instance
of the data structure.

We can bundle quite big chunks of such immutable data pieces.
Take a look at the famous [Redux](https://redux.js.org/) library for web development -
it completely relies on this pattern - we keep everything immutable and
join all updating events into one.
This approach works amazingly.

In addition, Redux makes yet another trick -- they are replacing
*events* with *immutable data*.
In terms of program complexity, it is like jumping from infinity to zero.
I cannot even tell how awesome it is -- instead of raising events
they're creating immutable data structures like "button X pressed" and then calling
a *pure* function that creates a new instance of the entire program
which is also just an immutable data structure by itself! 

### Immutable data is not a new idea

We can consider it as the final stage of the evolutionary process we started long time ago.
In our mainstream languages, we are just moving towards reducing the scope of mutability.

In the beginning, all of our variables were global and mutable and we did not have anything local.
Then some smart guys discovered that it is better to group data and functions together to
reduce amount of code that has access to it.

OOP appeared later with the defensive strategy of hiding data.
From an OOP perspective this was totally correct - we do not want
someone break the data our functions are working on.

Then community evolved some best practices that were including
some degree of immutability in specific places.
These ideas supposedly came from
functional languages that are stricter in ways we can mutate stuff.

Later we got some libraries for automatic generation of
"data classes" and libraries supporting immutability.

Finally, data classes got more adoption in recent programming languages
and even developers of Java are considering them.

While this evolutionary process obviously takes place,
not all people are getting how the world changes with immutability.
And one of the main ideas that appears here is that **we are not hiding data**.

Nothing can break immutable data.
So there is no point in protecting it from external access.

Data is not an "implementation detail" anymore
and our functions do not rely on internal data structure anyway.
We're shifting our programming paradigm
from manipulating on mutable fields into
manipulating on large chunks of data.

Instead of "modify person's name and pass it through 10 functions to show it on the screen", we now have "modify the person's name and pass the *person* through 10 functions to show it on the screen".
You see?
Now these 10 functions that are passing data are not passing name (implementation detail) anymore, so it does not matter if it hidden or not.
In addition, if we will want to pass the person's *age* we will not even need to write yet another 10 passing functions.

# Immutability and OOP

Nowadays people who are still writing plain mutable OOP code are
sometimes very suspicious about data classes.
They think that data classes violate OOP paradigm by being "too dumb" and "too open".
But reality is that data classes are the best way so write simple code.

Excessive data hiding in OOP leads to the fact that in OOP languages
we have to pass data in very intricate ways.
If we want to serialize `Person` into JSON
we're passing some kind of abstraction like `Serializer`
and ask `Person` to serialize itself.
Sounds easy, but this practice leads to having class hierarchies
and gigantic classes with mostly copy-pasted code.
At the end `Person` will have to be able to do everything:
serialize, deserialize, copy,
print, send itself over network, store itself into database,
compare and so on.
At some point `Serializable` will appear, because we will want to
somehow control this hell, and things will get even worse.

In the world of open data, we can just have a simple serialization
function without class hierarchies.
And this function will have zero complexity as we already know.

Before understanding immutability benefits we were doomed to select
one of two evils - we either break data hiding rule and make our data vulnerable
to external forces OR we use class hierarchies that are very verbose and 
hard to understand, compose and test.
But now the problem is solved.

Let's leave pure OOP to those who "like to be verbose" and go to the next stop.
We already won lambdas that became acceptable in most languages
despite OOP purist complaints, and there are very few steps to do left.
The "base programming language feature pack" is getting closer.

Definitely, when we're writing a library we should still limit amount
of internal data and functions visibility - for the sake of simplicity of API
and for the sake of the freedom to change the library internals.

# Immutability at smaller scale

Once we decided to use immutability we often find ourselves using
functional approach more than OOP.
Especially in order to work with lists.

Imagine we need to remove all persons that are younger than 18 from our list
we're showing on the screen.

We can't just take the list and remove the items from it, remember: the list is immutable.
We will need to make a copy of it with items removed.

In Java with mutable data this code typically look like this:

```java
for (int i = list.size() - 1; i >= 0; i--) {
    if (list.get(i).age < 18) {
        list.remove(i);        
    }
}
```

How many times we're repeating this pattern over and over again?
Ubiquitous `for` loops are plaguing our codebases,
violating the "Do not Repeat Yourself" rule.

Just count the amount of times we can make a mistake filtering the mutable list.
We could forget to type `-1` at the beginning. We could forget to put `>=` sign instead of `>`.
The idea of reverse iteration is totally unobvious to newbies and they're constantly making
these mistakes, hanging up for tens of minutes every time they need to remove items.

But if we decided to go immutable we just cannot write such loops anymore.
There are a few strategies for working with immutable lists,
but all of them are very verbose.
So instead of repeating ourselves we tend to write small
reusable functions for working with immutable data.

In addition, we don't want the temporary mutable data to leak
and break our strategy, so the main policy about temporary mutable data - it
should not leave the function it was created in.

This is the typical code we can find in a codebase
that is oriented on working with immutable data:

```java
newList = filter(list, person -> person.age >= 18);
```

How many times we can make a logical mistake filtering this list?
No matter how hard I try, I cannot find more potential bugs
than in the example with `for` loop.

# Performance FAQ

This is the most questionable topic when people are considering
to adopt immutability strategy.

There are three answers.

1. **Yes**, in most cases it is possible to squeeze more performance
from mutable data than from immutable.

    Garbage collection can also hit performance significantly if all what the app is doing
    is allocating large chunks of memory on 16 cores of the server.
    However, at this stage it is better to avoid having a garbage-collecting language at first,
    or try to run several instances of the program with proper load balancing.

2. **No**, in most cases neither developer nor users will notice any difference.
    Bottlenecks are rarely connected with immutability strategy.

    Modern languages have very fast memory allocation algorithms - they just select
    a next free chunk and increase the index in the array of free chunks of memory.

    Once I was able to decrease execution time of a specific function in two times
    by replacing immutable data with mutable.
    But this happened just because I was intentionally spending my time on optimizing
    a function that gets fired during rendering.
    All the data outside of the function was still immutable, getting all the benefits
    of simpler code.

    The opposite is also possible - in the example
    above `filter` function can actually work faster because
    when deleting items from a mutable list with `for` loop `remove` function
    wastes time on copying items every time we're deleting an item.

3. **Sometimes** program performance can even increase up to several times.
The aforementioned Redux library does some nifty tricks to compare large chunks of immutable data.

We have a long history of using immutable data in very different places - from
mobile devices with restricted processing power to servers under heavy load.
Erlang, for example, is known as a language
which is able to scale well under heavy load,
but it does not have mutable data at all.

Remember: "premature optimization is the root of all evil".
It is always easier to refactor a tiny piece of performance-critical code
than to rewrite the whole program because codebase became an unmaintainable buggy mess.

# Afterthought: The Miracle

In our daily lives, we all know that when we take something from
one place it appears in another.
This is one of the main laws of universe.

But when we consider ideas it sometimes works in
opposite direction: when we remove complexity -- other parts of the code
also become simpler.
The miracle, as I see it, is that we're reducing amount of events
by reducing amount of shared mutable data.
Complexity does not appear somewhere in exchange -- it just disappears.

Use simple tools to solve simple problems -- hard problems will not even appear.

# -

Thanks to [Alex Littlejohn](https://github.com/alexlittlejohn)
for proofreading this article.
