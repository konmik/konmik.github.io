+++
title = "Is programming actually math?"
date = "2017-04-24"
tags = ["Programming in general"]
+++

At first I was annoyed by the subject. I was even angry sometimes.
But once I found an article which conforms to my ideas on it and got a little bit more calm.
Now, a few years later, I finally decided to write my own ideas down.
The main difference between programming and math is...

<!--more-->

## The grand intro

People who say that programming is math are either do not get what
programming is, or studied math too much.

Just think: our reasons *why* we write software mathematicians call "side effects".
Is your app showing something on the screen?
It is just a side effect of some calculation.

Nice, yeah?

They say that "function" is a synonym for "computation"
(there is also a crazy definition of math function, but it is out of subject of this article).

We say that "function" is a series of steps to do.

Maybe we're just mistaking?
Let's check in a dictionary.
In Latin, "function" means "execution".
In usual life we say that a device is "functioning" to describe that it is "working".
So we're not mistaking - we're just using the same word with a different meaning.
And the difference is crucial.

I'm not telling that mathematicians are bad guys, of course no!
I just want to stress the difference between them and us, programmers.

## How did we end up putting the equation sign between math and programming?

There are two reasons why people think programming is related to math.

Mainly because of historical reasons.
First computers were created to calculate different stuff. 
They were even called "computers".

But modern computers aren't actually computers, they do much more.
When was the last time you actually calculated something on your computer?
Oh, some of you can't even probably remember.
I will remind you.
Your money, two months ago.
What have you been doing with your so called "computers" since then?
Not calculations, obviously.
Some YouTube, some Facebook, maybe you were reading some articles,
writing and receiving e-mails.

The second reason is a little bit more ridiculous.
When programming started to have real business value, universities decided to teach programmers.
I highly appreciate this attempt, despite the results we get sometimes.
So, they didn't have enough skilled programmers to teach programmers,
because programmers are always at high demand as we all know.
So they said mathematicians to give some training to students.
In fact, not many universities actually realized the difference,
they are still thinking that programming is a subdivision of math.

## So why isn't it a subdivision of math anymore?

Programming is a science of handling data.
Programs receive data, transform data and provide data.
And they do lots of these simultaneously.

I have been working as a professional computer programmer for 20 years,
and I haven't calculated a single integral.
I used mostly 2nd grade math - multiplication and division,
that's what we use in our daily life,
and we don't need to be mathematicians to do this, right?

Some programming languages that were used to build the most popular
accounting software can not even calculate a square root.
Add, Multiply and Subtract!
Division can sometimes be used, but beware of the scary consequences!
You can lose a cent because of rounding, so use it wisely.

You may say: "Hey, but computer processors are very complex,
they probably use lots of math internally, you need to know it!"
No, they don't.
They use ridiculously small amount of math.
The same 2nd grade math.
Except they don't even use division.

## So what is the real relation of programming to math?

> "Mathematicians do math, engineers use math"<br/>
> -- Somebody from the Internets

Math has exactly the same relation to programming as chemistry.

You can write a program for a chemist without knowing chemistry at all,
you just ask the chemist to provide you with formulas and you're done.
You don't actually need to study chemistry, you know?

When I was 18 I got my first serious job, and my first task was to create a database
for chemical calculations with the possibility to print results.
I had "below average" at my school chemistry class, so it was a very educative experience for me.
I didn't have *any* idea on what I was actually calculating.
I tried to find something similar in my school book and failed.
There were some crazy coefficients with 6 digits after the decimal points,
and data with mystical names: K1, C23 and such.
So I just entered the formulas into the programming language and got my money.

The same with math.
If you are a programmer you will never even think about opening a math book.
You'll ask your fellow mathematician to give you some formulas
and then you'll write an awesome looking app with database, e-mail notifications,
nice graphs, sharing the results with his colleagues, and so on.

Can a mathematician write a decent app?
Not everyone, you know.
People who write software as their primary job are usually
much better at writing software.

## Some consequences of the confusion we have now

Being a mathematician is not so cool these days - everyone wants to hire programmers.
Mathematicians who got hired are really lucky people.
So some of them got a little bit jealous and are saying that programming is math, so they
could be useful and prove that they didn't waste their life studying something useless.
I understand this.

But I want to say that while we clearly have a quite 
straight line between two very different disciplines, we do not have such lines between people.
If you're a mathematician, nobody will be against if you will create a new high-performance
cryptography algorithm out of your findings.
Or a nice sound encoding library.
Or a crazy artificial intelligence algorithm that you will sell to Google for a lot of bucks.
"Data science" is also a great field where math is vital.

Math has very wide areas of application in computer programming.
But it is just not the only thing anymore. 

## The final note

When I finally showed my "chemical" app to a group of chemists, everyone got very excited about it. 
So they started testing it and "found a bug": after applying a very long formula,
the final result was twice as bigger than they got from their calculator.
They said they should have never had such result otherwise we were all
doomed and the gas factory we were working at
would explode because of this.

I was trying to find the bug for a few days.
After I gave up, I decided to ask a chemist to make all the calculations manually
to verify my app's calculations step by step.
And I found out that she was throwing away some very small numbers after the decimal point, that's why
we had different results!!!

After some discussions we decided to lower the app's accuracy.
The factory didn't explode.
And I still have no idea what I was calculating there.

## References:

[Programming Is Not Math](http://www.sarahmei.com/blog/2014/07/15/programming-is-not-math/)

