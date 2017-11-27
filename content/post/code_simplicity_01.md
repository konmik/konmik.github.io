+++
title = "The Code Simplicity Chart (c)"
draft = false
date = "2017-11-21"
tags = ["Programming theory"]
+++

How to understand if your code is simple or complex.

<!--more-->

### Preface

This is probably the most confusing and the most important topic
of computer programming at the same time.
There are thousands of articles telling about the importance of
writing simple code, and how it is important for debugging,
adding new features quickly, for easier refactoring
and for catching up by newcomers.
Hundreds of widespread phrases were told by famous computer programmers
telling how good it is to write simple code.

But there is absolutely zero articles telling what simple code actually is.

This and the following articles are targeting these topics - what simple
code is, how to distinguish it from complicated code and how to replace
some patterns we use in our daily work with simpler patterns.

# What "simple" means here?

> Simple:<br/>
  "Easily understood or done; presenting no difficulty."<br/>
  "Plain, basic, or uncomplicated in form, nature or design;
  without much decoration or ornamentation."<br/>
  -- Oxford Dictionary

Simplicity is subjective, as it comes from our ability to
understand - our ability to recognize abstractions,
subdivide them into smaller parts and to find connections between them.
Luckily, the code we write is measurable so we can, to some extent,
measure the amount of hoops our brains have to jump through to understand how the
program works.

"What does this program do?" is the main question.
If it is easy and obvious to answer by reading the code -
the code is considered to be simple. 

# "Simple" does not mean "familiar"

While it is easier to write code using familiar
methods, it is just a matter of habit.
We learn new things to change our habits.
There can be habits which allow to produce less complicated code.
If we will learn them, they will be as familiar to us as our current methods
but the code we’re creating will be much simpler from the point of view
of the more educated persons we will become.

In these articles on simplicity I'm not going to feed you
with patterns which can change with
new programming languages or which can
even become anti-patterns with time.

Let's go.

# Quantity matters

This is the most important part and all other ideas on simplicity
grow from this fact. Quantity matters. Be it the number of Lines of Code (LoC),
the number of `if` conditions, the number of variables - it all matters.

It is a widespread fact that people can keep in mind a limited amount of variables.
I disagree.
Given enough time we can literally place the entire program into our mind
and play with it as we want to.
However, while working with real-life tasks we always have
limited amount of time and limited amount of code to read.
In addition, our code is constantly evolving, invalidating
the pieces of code we know.
We switch between tasks, our attention switches,
and now we need to re-read the code to catch things up.

I sometimes meet people who say: "I like to be verbose".
And sometimes I am not lucky enough so I have to read their code 
and this is **hard**.
Instead of using if/else they're creating a class hierarchy.
Instead of creating a reusable function they're copy-pasting 90% of their code
into several places.

There is a common practice to be more verbose to explain an algorithm
with longer variable names or by splitting long expressions into
several small statements.
There are also exceptional cases when the algorithm is very compressed,
and replacing it with a longer version increases readability.
But given the same readability, the shorter version is almost always simpler.

The obvious thing is that understanding a part of a program (the smaller amount of code)
is easier than understanding the entire program (the bigger amount of code).

So, this is the first "axiom" here: quantity matters.

> Axiom - "a rule or principle that most people believe to be true"<br/>
  -- Oxford Dictionary

# The Code Simplicity Chart (c)

I never saw it written anywhere, but it literally flies in the air.
If you know several programming languages implementing different paradigms,
this thing becomes pretty obvious.
However, nobody wrote it.
So I've added this (c) to mark the authorship and to draw your 
attention to the fact that this thing is actually *NEW*. 

Here it is, from simple to complex:

- Immutable data
- Pure function
- I/O function
- Shared mutable data
- Event function

Let's consider each of the items here.

### Immutable data
 
**Immutable data** is the simplest possible thing our programs consist of.
Be it a constant or some data structure passed into a function as a parameter -
if it is immutable then it is simple.

It is simple because it does not change over time.
It contains no actions or logic.

It also does not matter with which other functions it is shared with,
the data can be considered without thinking about the other application parts,
allowing us to write totally isolated pieces of program.
It will just never break and we can always rely on it.

In programming languages without support of immutable data
a simple agreement to not modify the data structure can do the trick.

### Pure function

**Pure function** is a function that does only one thing - it transforms data.

Pure function is more complex than immutable data because to write
a function we need to know the data structure it works on.
Complexity just adds, it is not different.
Pure functions are also simple and rock solid because they 
never break anything outside of them and they always produce the same
result given the same arguments.

So, here they are - two building blocks that can be used to 
write code that is actually scalable.

Only *zero* scales infinitely.

Zero mutation data and zero side-effecting functions.
They can be composed and used without limitations or the fear that something will break.
We can grow a program to whatever size we want without being afraid that
the amount of bugs will start growing exponentially.

When speaking about pure function simplicity, we just mean that
even if it is complex, its complexity does not affect other parts of
the program.

### I/O function

**I/O function** transforms data but it also should
take place at a certain moment in time.

We can call pure functions in any order and any number of times -
the result will always be the same.
I/O functions do not have such luxury: if we change the order of I/O function
calls we will get "Joe!Hello, " instead of "Hello, Joe!".

This complexity does not scale well.
Once, we have a couple of functions that should be called
in a certain order, we have to architect the whole of our
program in a way that these functions will never
be called in wrong order, otherwise bugs will happen.

Imagine calling `closeFile` function before calling `read`.
To prevent this, we wrap such functions into one function which
does these calls in the right order.
We also should not forget to call `closeFile`.
But now we have a more general I/O function and this function
must also be called in a specific order.

We may say that when joining two I/O function calls we
reduce these function complexity 2 to 1 but it is still not 0.
It can become even worse when we have parallel code execution
in the program and the I/O function can covertly affect the execution
of other functions sharing the same resource.

### Shared mutable data

**Shared mutable data** goes next.
It is important that it must be modified in a certain order,
and in addition it affects, covertly, execution of other functions.

Shared mutable data is one of the main reasons behind bugs.
It is hard to trace all the ways the data can be
modified and so it makes program harder to understand.
In large programs, mutable data can be shared quite intricately.

For example, we have a list of goods in memory cache and we want to share it
across different application screens to avoid looking up database too often.
Suddenly, one of the screens decides to apply a filter to the list and removes
some goods that it does not need to show.
Nice!
The next time another screen will want to take the list of goods,
it will only get a part of the list while expecting to get all of them.
This is a real bug I fixed once.
It could never had happened if the list was immutable. 
And who knows how many similar bugs exist being unnoticed
by testers and annoying users with misbehavior or even crashes. 

The problem of mutable shared data makes it
almost impossible to write reliable multi-threading code.
The complexity of the order of data mutation in multi-threading
environment becomes unbearable *very* quickly, so refactoring
code to make use of immutable data is the most reasonable thing we can usually do
to eliminate race conditions.
It usually leads to code that is easier to understand as well,
because immutable data encourages writing pure functions
that are very easy to understand.

Shared mutable data can sometimes be considered as a couple of
I/O functions - one reads and another writes, while the
data can be considered as external to our program.
This approach even works in some programming languages,
but it just adds another layer of complexity
on top of shared mutable data.

### Event function

**Event function** has the highest complexity because it
can perform several I/O calls, mutate shared data,
and the worst - it can also call other event functions.

For the sake of simplicity, I'm joining under this category
all the multi-purpose functions which do hell knows what.
"Here be dragons".

# The chart is not complete

There are items in-between.
We can consider, for example, local function mutable variables
or some fancy ways to work with shared mutable data.
The chart shows the main milestones.
The rest lies somewhere in-between.

There can also be obvious cases when by having one shared mutable
variable we can avoid writing one hundred lines of 
pure functions, but such cases are extremely rare.
Quantity matters and if it allows to write less code,
then that is cool, but we should also consider how much other code
the reader will have to dig through to understand all the possible
interactions with the variable, be they direct or mediated, and how
often he or she will have to re-read the code
if something will be changed.

# What's next?

The application of this chart is a skill that every computer programmer
should develop to consider (him|her)self a professional.
It boosts productivity and application reliability to the highest levels.
Maintenance costs also go down significantly as we have to spend less time
on debugging and reading old code.

These principles already took the web development experience by storm,
raising the latest wave of popular frameworks.

The story just starts here - we can use these principles
for writing most of our software, getting all the benefits
of having simpler code, but this time the approach on simplicity
is not just based on someone's opinion - it is logically
justified and can be actually measured.

# TODO

* Some magnificent ways to apply The Code Simplicity Chart
* How code simplicity principles evolve the way we do IT
* Simplicity in programming language design
* Reducing complexity in fine details
* Architecture and code simplicity

I'm planning to release one article per couple of weeks,
but life is an unpredictable thing.

Ping me on Twitter if you have to wait too 
much for the next article in the series,
this can actually motivate me to write more. :)

Sharing the article also helps! ;)

