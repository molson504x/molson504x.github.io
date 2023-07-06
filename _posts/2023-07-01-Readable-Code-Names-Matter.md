---
layout: post
title: Readable and Maintainable Code - Names Matter
date: 2023-07-01 13:00:00 -0400
description: Writing readable code makes it easier to maintain and understand.  A lot of time that could be spent fixing bugs is spent simply trying to understand what code is doing, and that is often due to how things are named.  In this blog post, I'm going to highlight a few naming practices that developers can improve on to make code more readable and easier to maintain.
tags: [Coding, Best Practices, Technology]
---

> There are two hard things in computer science: cache invalidation, naming things, and off-by-one errors.
>
> -- <cite>Jeff Atwood</cite>

Have you ever been doing a code review and been stuck wondering what the purpose of a variable is?  Or had a late-night debugging session and found yourself stuck trying to interpret what a function is supposed to be doing?  I have.  Often.  I find that when reading through a section of code a lot of my time is spent just trying to figure out what things are supposed to be represent.  This can usually be fixed pretty easily by doing two key things - ading more comments to describe what's going on and naming things in a manner that makes them easier to understand and _read_.  Adding more meaningful comments is pretty easy - just do it.  Giving things more meaningful names is a lot harder, and something that's really difficult to do as a developer.  A name that's meaningful to me might be meaningless to someone else.  In this post, I'm going to describe a few key tips I would give to any developer to unwrap this mystery of "how to name things" and make code easier to read.

## Variables with a single letter

As a computer science major, we are presented with math formulas that look something like $y = mx + b$ and $((ax^2 + bx + c) = 0$.  Math formulas use single-letter variables, and mathemeticians seem to pride themselves on working equations down to the smallest possible footprint.  Sometimes, developers like to do the same thing.  The thing is, you generally don't have to edit and debug math, but you do have to edit and debug code, for many reasons.  

It's becoming more and more obvious then that single-letter variable names are a bad practice.

Consider this simple function:

``` typescript
public static parseCookie(n: string): string | null {
 const s = `; ${document.cookie}`.split('; ');
 for (let c of s) {
  if (c.indexOf(n) === 0) {
   return c.substring(c.length + 1);
  }
 }

 return null;
}
```

The name of the function says it's going to do something to do with parsing a cookie, but what's it actually doing?

### Make your names meaningful

Now for a quick side-note: this same principal also applies to making variable names meaningful.  For example, let's take our sample "parseCookie" function - if I rewrite it as this, does it make any more sense than before?

``` typescript
public static readTheThing(bacon: string): string | null {
 const strawberry = `; ${document.cookie}`.split('; ');
 for (let turkey of strawberry) {
  if (turkey.indexOf(bacon) === 0) {
   return turkey.substring(turkey.length + 1);
  }
 }

 return null;
}
```

It's almost harder to follow than when the function had single-letter variable names now because instead of the variable names having any meaning, it's complete gibberish.

When we actually name our variables with meaningful names, we get a better picture:

``` typescript
public static getCookieByName(name: string): string | null {
 const cookies = `; ${document.cookie}`.split('; ');
 for (let cookie of cookies) {
  if (cookie.indexOf(name) === 0) {
   return cookie.substring(cookie.length + 1);
  }
 }

 return null;
}
```

Now we can tell what this function is going to do - find a cookie with a given name and return it, or return null if it can't find it.  By making the code easier to read it is immediately easier to understand, and thus easier to debug and enhance in the future.

## Be Verbose - Stop Abbreviating So Much

Abbreviations require a reader to have context around what is being abbreviated.  Depending on the context, an abbreviation like "est" could mean "estimated," "eastern standard time," or "established."  Without context, it's hard to know.  Now let's suppose that we had a function named "GetEstCustomers" - is this function going to get a list of established customers or an amount of estimated customers?  Screens now have plenty of space to be more verbose, so there's no real penalty for being verbose.  Calling that function "GetEasternCustomers" gives a lot more insight into what the function is going to be doing.

## Units in names

Unlike the other recommendations here, this is a "you should do this" moment.  How often do you come across a parameter named "delay?"  What about "distance?"  Should that be seconds or years?  Meters or miles?  As a best practice, always recommend being explicit by specifying units in variable names.  That way you know what's actually being stored.  "DistanceInParsecs" or "DelayInMilliseconds" is more meaningful and eliminates the guess-work of figuring out

The same idea goes for method names too.  A method named "SetTotalDelay" or "GetAmountOfPaint" doesn't describe what is actually being returned.  Is that the total delay in days?  Is that ounces of paint or gallons?  Being more verbose adds a lot of clarity.  "SetTotalDelayInMinutes" or "GetCustomerLastName" better describes what's being returned.

### Use types that are unit-agnostic

There's an exception to this rule, and in this case it's one I'd argue is a good one to make.  Using types that are unit-agnostic is a good idea.  Let's look at the "SetTotalDelayInMinutes" example.  It now forces a user to supply the unit of time in minutes.  What if we rewrote that method as:

``` c#
public void SetDelayTime(TimeSpan delay) 
{
 _settings.DelayMinutes = delay.Minutes;
 _settings.Save();
}
```

Now, the unit of time being passed in doesn't matter because TimeSpan is unit-agnostic, meaning that it going to store the value once (as ticks) and then do on-the-fly conversion to return the other values ([source code here](https://github.com/microsoft/referencesource/blob/master/mscorlib/system/timespan.cs)).

## The "Utils" or "Helpers" catch-all

Developers LOVE to create classes called "Utils" or "Helpers" and throw miscellaneous functions into those classes.  You should really ask yourself if that's the right spot to place a function.  Would it make more sense for some of those functions to be a part of their respective types?  Are they re-used across that many classes that it makes sense to share them in a "Helpers" or "Utils" class?  Would it make more sense to move some of those functions into a more meaningful class with a more descriptive name?  In general, well written libraries don't have a bundle of "Utils" or "Helpers" because they can all be sorted into modules or classes that have well-defined names.  We should do the same in our code.

---

I'd like to leave you with this thought:
> Programs are meant to be read by humans and only incidentally for computers to execute.
>
> -- <cite>Donald Knuth</cite>

If your code isn't able to be easily read by humans, it probably can be greatly improved.  Your code should be understandable, well-organized, and well-named.  By making conscious decisions to avoid single-letter variable names and ambiguous names such as "Utils" you can immediately improve the human-readability of your code.
